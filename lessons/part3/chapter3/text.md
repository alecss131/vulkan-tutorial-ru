# Vulkan. Руководство разработчика. Загрузка данных через промежуточный буфер

## Вступление

Вершинный буфер, который мы создали, работает правильно, но тип памяти, позволяющий получить к нему доступ из CPU, может оказаться неоптимальным для чтения со стороны видеокарты. Для обозначения наиболее оптимальной памяти используется флаг `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`, но эта память обычно недоступна для CPU. В этой главе мы создадим два вершинных буфера: один промежуточный буфер в памяти, доступной для CPU, для загрузки данных из массива вершин и конечный вершинный буфер в локальной памяти устройства. Затем мы используем команду копирования буфера, чтобы переместить данные из промежуточного буфера в конечный вершинный буфер.

## Очередь передачи

Для отправки команды копирования буфера требуется семейство очередей с поддержкой операций передачи, которая может быть определена с помощью `VK_QUEUE_TRANSFER_BIT`. К счастью, любое семейство очередей с поддержкой `VK_QUEUE_GRAPHICS_BIT` или `VK_QUEUE_COMPUTE_BIT` автоматически поддерживает операции `VK_QUEUE_TRANSFER_BIT`. В этом случае не требуется явно указывать их в реализации в queueFlags.

Если для вас это слишком просто, вы можете попробовать использовать отдельное семейство очередей для операций передачи. Для этого вам потребуется внести следующие изменения в программу:

- Изменить `QueueFamilyIndices` и `findQueueFamilies` для явного поиска семейства очередей с битом `VK_QUEUE_TRANSFER_BIT`, а не `VK_QUEUE_GRAPHICS_BIT`
- Изменить `createLogicalDevice` для запроса дескриптора очереди передачи
- Создать второй пул команд для буферов команд, отправляемых в очередь передачи
- Изменить sharingMode на `VK_SHARING_MODE_CONCURRENT` и указать оба семейства — с поддержкой графических операций и с поддержкой операций передачи
- Отправлять команды передачи, например [vkCmdCopyBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdCopyBuffer.html) \(ее мы будем использовать в этой главе\), в очередь передачи, а не в графическую очередь

## Общий метод для создания буферов

Поскольку нам нужно создать несколько буферов, было бы неплохо вынести создание буфера во вспомогательную функцию. Создадим новую функцию `createBuffer` и переместим в нее код из `createVertexBuffer` \(за исключением маппинга\).

```cpp
void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory) {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = size;
    bufferInfo.usage = usage;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create buffer!");
    }

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate buffer memory!");
    }

    vkBindBufferMemory(device, buffer, bufferMemory, 0);
}
```

Добавим следующие параметры: размер буфера, его использование и свойства памяти, чтобы мы могли использовать эту функцию для создания разных типов буферов. Последние два параметра — это выходные переменные для записи дескрипторов.

Теперь можно удалить код создания буфера и код выделения памяти из `createVertexBuffer` и вызвать `createBuffer`:

```cpp
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
    createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, vertexBufferMemory);
}
```

Запустим программу, чтобы убедиться, что вершинный буфер по-прежнему работает правильно.

## Использование промежуточного буфера

Изменим `createVertexBuffer`, чтобы использовать видимый для хоста буфер в качестве временного буфера и локальный буфер устройства в качестве вершинного буфера.

```cpp
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
}
```

Теперь используем новый `stagingBuffer` вместе с `stagingBufferMemory` для копирования данных вершин. Обратите внимание, мы использовали два новых флага при создании буферов:

 - `VK_BUFFER_USAGE_TRANSFER_SRC_BIT`: буфер используется в качестве источника во время операции передачи памяти.
 - `VK_BUFFER_USAGE_TRANSFER_DST_BIT`: буфер используется в качестве назначения во время операции передачи памяти.

`vertexBuffer` теперь выделяется из локальной памяти устройства, а значит мы не можем использовать [vkMapMemory](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkMapMemory.html). Однако мы можем скопировать данные из `stagingBuffer` в `vertexBuffer`. Чтобы это сделать, нужно указать флаг `VK_BUFFER_USAGE_TRANSFER_SRC_BIT` для `stagingBuffer` и `VK_BUFFER_USAGE_TRANSFER_DST_BIT` для `vertexBuffer`.

Теперь напишем функцию `copyBuffer`, чтобы скопировать содержимое из одного буфера в другой:

```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {

}
```

Как и команды рисования, операции передачи памяти выполняются с помощью буферов команд. Поэтому прежде всего выделим временный буфер команд. Возможно, для таких временных буферов вы захотите создать отдельный пул команд, поскольку это позволит применить оптимизации выделения памяти на уровне драйвера. В таком случае во время создания пула команд используйте флаг `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`.

```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
}
```

И сразу начнем запись буфера команд:

```cpp
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

vkBeginCommandBuffer(commandBuffer, &beginInfo);
```

Мы отправим буфер на выполнение только один раз, после чего будем ожидать, пока операция копирования не завершится. Лучше будет сообщить драйверу о наших намерениях с помощью `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`.

```cpp
VkBufferCopy copyRegion{};
copyRegion.srcOffset = 0; // Optional
copyRegion.dstOffset = 0; // Optional
copyRegion.size = size;
vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);
```

Содержимое буферов передается с помощью команды [vkCmdCopyBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCmdCopyBuffer.html). Она принимает исходный буфер и буфер назначения в качестве аргументов и массив участков памяти для копирования. Участки памяти определены в структурах [VkBufferCopy](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkBufferCopy.html) и состоят из смещения в исходном буфере, смещения в буфере назначения и размера. В отличие от [vkMapMemory](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkMapMemory.html), мы не можем указать здесь `VK_WHOLE_SIZE`.

```cpp
vkEndCommandBuffer(commandBuffer);
```

Теперь запустим буфер команд, чтобы завершить передачу:

```cpp
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;

vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
vkQueueWaitIdle(graphicsQueue);
```

В отличие от команд рисования, копирование можно провести здесь и сейчас, необходимо лишь дождаться его завершения. Для этого есть два способа. Можно использовать барьер \(fence\) и дождаться завершения передачи с помощью [vkWaitForFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkWaitForFences.html), либо дождаться, когда очередь передачи освободится, с помощью [vkQueueWaitIdle](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueueWaitIdle.html). Барьер позволяет запланировать несколько передач одновременно, что дает драйверу больше возможностей для оптимизации.

```cpp
vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
```

Не забудьте удалить буфер команд, используемый для операции передачи.

Теперь мы можем вызвать `copyBuffer` из `createVertexBuffer`, чтобы переместить данные вершин в локальный буфер устройства:

```cpp
createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

copyBuffer(stagingBuffer, vertexBuffer, bufferSize);
```

После копирования данных из промежуточного буфера в буфер устройства его нужно будет удалить:

```cpp
    ...

    copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

Теперь запустим программу, чтобы увидеть уже знакомый треугольник. Возможно, улучшения пока не будут заметны, но в этот момент данные вершин треугольника загружаются из высокопроизводительной памяти. Разница будет видна, когда мы начнем рендерить более сложную геометрию.

## Заключение

Следует отметить, что в реальной программе не стоит вызывать [vkAllocateMemory](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAllocateMemory.html) для каждого буфера. Максимальное количество одновременных выделений памяти ограничено лимитом физического устройства `maxMemoryAllocationCount` и может составлять всего `4096` даже на высокопроизводительном оборудовании, таком как NVIDIA GTX 1080. Чтобы одновременно выделить память для множества объектов, нужно создать кастомный аллокатор, который разобьет одно выделение между множеством разных объектов с помощью уже знакомого нам параметра `offset`.

Вы можете реализовать такой аллокатор самостоятельно или использовать библиотеку [VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator), предоставленную GPUOpen. Однако в рамках нашего руководства можно отдельно выделить память для каждого ресурса, т. к. это не превысит лимит.

[Код С++](19_staging_buffer.cpp) / [Вершинный шейдер](17_shader_vertexbuffer.vert) / [Фрагментный шейдер](17_shader_vertexbuffer.frag)
