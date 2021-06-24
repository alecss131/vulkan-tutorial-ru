# Vulkan. Руководство разработчика. Отрисовка

## Фреймбуферы

В последних главах мы много говорили о фреймбуферах и настроили проход рендера для одного фреймбуфера с таким же форматом, что и image из swap chain. Однако сам фреймбуфер до сих пор не создали.

Буферы \(attachments\), на которые мы ссылались при создании прохода рендера, необходимо обернуть в [VkFramebuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkFramebuffer.html). Он указывает на все объекты [VkImageView](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImageView.html), которые мы будем использовать как таргеты. В нашем случае только один буфер: буфер цвета. Однако swap chain содержит множество image, и использовать мы должны именно тот, который получили от swap chain для рисования. Другими словами, нам необходимо создать фреймбуферы для каждого image из swap chain и использовать тот фреймбуфер, к которому прикреплен интересующий нас image.

Для этого добавим еще один член класса:

```cpp
std::vector<VkFramebuffer> swapChainFramebuffers;
```

Добавим функцию `createFramebuffers` и вызовем ее сразу после создания графического конвейера. Внутри этого метода будем создавать объекты для нашего массива:

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
}

...

void createFramebuffers() {

}
```

Сначала выделим необходимое место в контейнере для хранения фреймбуферов:

```cpp
void createFramebuffers() {
    swapChainFramebuffers.resize(swapChainImageViews.size());
}
```

Затем обойдем все image views и создадим из них фреймбуферы:

```cpp
for (size_t i = 0; i < swapChainImageViews.size(); i++) {
    VkImageView attachments[] = {
        swapChainImageViews[i]
    };

    VkFramebufferCreateInfo framebufferInfo{};
    framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    framebufferInfo.renderPass = renderPass;
    framebufferInfo.attachmentCount = 1;
    framebufferInfo.pAttachments = attachments;
    framebufferInfo.width = swapChainExtent.width;
    framebufferInfo.height = swapChainExtent.height;
    framebufferInfo.layers = 1;

    if (vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create framebuffer!");
    }
}
```

Как видите, создать фреймбуфер несложно. Сначала нужно указать, с каким `renderPass` он должен быть совместим. Фреймбуферы могут быть использованы только с совместимыми проходами рендера, то есть для них используется одинаковое количество и тип буферов.

Параметры `attachmentCount` и `pAttachments` указывают на объекты [VkImageView](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImageView.html), которые должны соответствовать описанию `pAttachments`, использованному при создании прохода рендера.

Параметры `width` и `height` не вызывают вопросов. В поле `layers` указывается количество слоев для images. Наши images состоят только из одного слоя, поэтому `layers = 1`.

Фреймбуферы нужно удалить перед image views и проходом рендера, но только после окончания рендеринга:

```cpp
void cleanup() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    ...
}
```

Теперь у нас есть все объекты, необходимые для рендеринга. В следующей главе мы уже сможем написать первые команды рисования.

[Код C++](13_framebuffers.cpp) / [Вершинный шейдер](09_shader_base.vert) / [Фрагментный шейдер](09_shader_base.frag)


## Буферы команд

Команды в Vulkan, такие как операции рисования и операции перемещения в памяти, не выполняются непосредственно во время вызова соответствующих функций. Все необходимые операции должны быть записаны в буфер. Это удобно тем, что сложный процесс настройки команд может быть выполнен заранее и в нескольких потоках. В основном цикле вам остается лишь указать Vulkan, чтобы он запустил команды.

### Пул команд

Прежде чем перейти к созданию буфера команд мы должны создать пул команд. Пул команд управляет памятью, которая используется для хранения буферов.

Добавим новый член класса [VkCommandPool](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkCommandPool.html).

```cpp
VkCommandPool commandPool;
```

Теперь создадим новую функцию `createCommandPool` и вызовем ее из `initVulkan` после создания фреймбуферов.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
}

...

void createCommandPool() {

}
```

Для создания пула команд потребуется только два параметра:

```cpp
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
poolInfo.flags = 0; // Optional
```

Чтобы запустить буферы команд, их необходимо отправить в очередь, например, в графическую очередь или очередь отображения. Пул команд может выделять буферы команд только для одного семейства очередей. Нам нужно записать команды рисования, поэтому мы используем семейство очередей с поддержкой графических операций.

Есть два возможных флага для пула команд:

- `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`: подсказка, сообщающая Vulkan, что буферы из пула будут часто сбрасываться и выделяться \(может поменять поведение пула при выделении памяти\)
- `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`: позволяет перезаписывать буферы независимо; если флаг не установлен, сбросить буферы можно будет только все и одновременно

Мы запишем буферы команд только в начале работы программы, а затем будем неоднократно запускать их в основном цикле, поэтому ни один из этих флагов нам не подходит.

```cpp
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```

Завершим создание пула команд функцией [vkCreateCommandPool](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateCommandPool.html). В ней нет никаких специальных параметров. Команды будут использоваться на протяжении всего жизненного цикла программы, поэтому пул нужно уничтожить в самом конце:

```cpp
void cleanup() {
    vkDestroyCommandPool(device, commandPool, nullptr);

    ...
}
```

### Выделение памяти для буферов команд

Теперь мы можем выделить память для буферов команд и записать в них команды рисования. Поскольку каждая из команд рисования привязана к соответствующему [VkFrameBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkFramebuffer.html), мы должны записать буфер команд для каждого image из swap chain. Для этого создадим список объектов [VkCommandBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkCommandBuffer.html) в виде члена класса. После уничтожения пула команд буферы команд автоматически освобождаются, поэтому нам не требуется явная очистка.

```cpp
std::vector<VkCommandBuffer> commandBuffers;
```

Перейдем к функции `createCommandBuffers`, которая выделяет память и записывает команды для каждого image из swap chain.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createCommandBuffers();
}

...

void createCommandBuffers() {
    commandBuffers.resize(swapChainFramebuffers.size());
}
```

Буферы команд создаются с помощью функции [vkAllocateCommandBuffers](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAllocateCommandBuffers.html), которая принимает структуру [VkCommandBufferAllocateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkCommandBufferAllocateInfo.html) в качестве параметра. В структуре указывается пул команд и количество буферов:

```cpp
VkCommandBufferAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate command buffers!");
}
```

Параметр `level` определяет, первичными или вторичными будут буферы команд.

- `VK_COMMAND_BUFFER_LEVEL_PRIMARY`: первичные буферы могут отправляться в очередь, но не могут вызываться из других буферов команд.
- `VK_COMMAND_BUFFER_LEVEL_SECONDARY`: вторичные буферы не отправляются в очередь напрямую, но могут вызваться из первичных буферов команд.

Мы не будем использовать вторичные буферы команд, хотя иногда это может быть удобным перенести в них некоторые часто используемые последовательности команд и переиспользовать их в первичном буфере.

### Запись буфера команд

Начнем запись буфера команд с вызова [vkBeginCommandBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkBeginCommandBuffer.html). В качестве аргумента передадим небольшую структуру [VkCommandBufferBeginInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkCommandBufferBeginInfo.html), которая содержит информацию об использовании этого буфера.

```cpp
for (size_t i = 0; i < commandBuffers.size(); i++) {
    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = 0; // Optional
    beginInfo.pInheritanceInfo = nullptr; // Optional

    if (vkBeginCommandBuffer(commandBuffers[i], &beginInfo) != VK_SUCCESS) {
        throw std::runtime_error("failed to begin recording command buffer!");
    }
}
```

Параметр `flags` указывает, как будет использоваться буфер команд. Доступны следующие значения:

- `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`: после каждого запуска буфер команд перезаписывается.
- `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`: указывает, что это будет вторичный буфер команд, который целиком находится в пределах одного прохода рендера.
- `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`: буфер может быть повторно отправлен в очередь, даже если предыдущий его вызов ещё не выполнен и находится в состоянии ожидания.

На данный момент ни один из флагов нам не подходит.

Параметр `pInheritanceInfo` используется только для вторичных буферов команд. Он определяет состояние, наследуемое из первичных буферов.

Если буфер команд однажды уже был записан, вызов [vkBeginCommandBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkBeginCommandBuffer.html) неявно сбросит его. Нельзя добавлять команды в уже заполненный буфер.

### Начало прохода рендера

Приступим к отрисовке треугольника, начав проход рендера с помощью [vkCmdBeginRenderPass](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdBeginRenderPass.html). Проход рендера настраивается с помощью некоторых параметров в структуре [VkRenderPassBeginInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkRenderPassBeginInfo.html).

```cpp
VkRenderPassBeginInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[i];
```

Первые параметры определяют проход рендера и фреймбуфер. Мы создали фреймбуфер для каждого image из swap chain, который используется в качестве цветового буфера.

```cpp
renderPassInfo.renderArea.offset = {0, 0};
renderPassInfo.renderArea.extent = swapChainExtent;
```

Следующие два параметра определяют размер области рендеринга. Область рендеринга \(render area\) определяет участок фреймбуфера, на котором шейдеры могут сохранять и загружать данные. Пиксели за пределами этой области будут иметь неопределенные значения. Для достижения лучшей производительности область рендеринга должна соответствовать размеру буферов.

```cpp
VkClearValue clearColor = {0.0f, 0.0f, 0.0f, 1.0f};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```

Последние два параметра определяют значения для очистки буферов при использовании операции `VK_ATTACHMENT_LOAD_OP_CLEAR`. Заполним фреймбуфер черным цветом со 100% непрозрачностью.

Теперь начнем проход рендера.

```cpp
vkCmdBeginRenderPass(commandBuffers[i], &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```

Функции для записи команд в буфер можно узнать по префиксу [vkCmd](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmd.html). Все они возвращают `void`, поэтому обработка ошибок будет производиться только после окончания записи.

Первый параметр для каждой команды — буфер команд, в который эта команда записывается. Второй параметр — информация о проходе рендера. Последний параметр определяет, как будут предоставляться команды в подпроходе \(subpass\). Вы можете выбрать одно из двух значений:

- `VK_SUBPASS_CONTENTS_INLINE`: команды будут вписаны в первичный буфер команд без запуска вторичных буферов.
- `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS`: команды будут запущены из вторичных буферов команд.

Мы не будем использовать вторичные буферы команд, поэтому выберем первый вариант.

### Основные команды рисования

Теперь мы можем подключить графический конвейер:

```cpp
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```

Второй параметр указывает, какой конвейер используется — графический или вычислительный. Осталось нарисовать треугольник:

```cpp
vkCmdDraw(commandBuffers[i], 3, 1, 0, 0);
```

Функция [vkCmdDraw](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdDraw.html) выглядит весьма скромно, но ее проста связана с тем, что все данные мы указали заранее. Помимо буфера команд в функцию передаются следующие параметры:

- vertexCount: несмотря на то, что мы не используем вершинный буфер, технически у нас есть 3 вершины для отрисовки треугольника.
- instanceCount: используется, когда необходимо отрисовать несколько экземпляров одного объекта \(instanced rendering\); передайте `1`, если не используете эту функцию.
- firstVertex: используется в качестве смещения в вершинном буфере, определяет наименьшее значение `gl_VertexIndex`.
- firstInstance: используется в качестве смещения при отрисовке нескольких экземпляров одного объекта; определяет наименьшее значение `gl_InstanceIndex`.

### Завершение

Теперь мы можем закончить проход рендера:

```cpp
vkCmdEndRenderPass(commandBuffers[i]);
```

И завершить запись буфера команд:

```cpp
if (vkEndCommandBuffer(commandBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```

В следующей главе мы напишем код для основного цикла, где image извлекается из swap chain, после чего запускается соответствующий буфер команд, а затем готовый image возвращается в swap chain для вывода на экран.

[Код C++](14_command_buffers.cpp) / [Вершинный шейдер](09_shader_base.vert) / [Фрагментный шейдер](09_shader_base.frag)
