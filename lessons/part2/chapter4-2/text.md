# Vulkan. Руководство разработчика. Рендеринг и отображение на экране

## Подготовка

В этой главе мы сможем собрать все части воедино. Напишем функцию drawFrame и вызовем ее из mainLoop, чтобы вывести наш треугольник на экран:

```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
}

...

void drawFrame() {

}
```

## Синхронизация

Функция `drawFrame` выполняет следующие операции:

- Получение image из swap chain
- Запуск соответствующего буфера команд для этого image
- Возвращение image в swap chain для вывода на экран

Каждое из этих действий выполняется с помощью одного вызова функции, однако все они выполняются асинхронно. Выполнение функции завершается еще до выполнения операций, и порядок выполнения не определен. Это является проблемой, поскольку каждая операция зависит от результата предыдущей операции.

Есть два способа синхронизировать операции в swap chain: с помощью барьеров \(fences\) и семафоров \(semaphores\). Эти объекты используются для координации операций: пока одна операция выполняется, следующая ожидает, когда синхронизатор перейдет из состояния unsignaled в signaled.

Разница между барьером и семафором в том, что состояния барьеров можно получить из вашей программы, например, с помощью [vkWaitForFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkWaitForFences.html), а для семафоров — нельзя. Барьеры в основном предназначены для синхронизации самой программы с операциями рендеринга, а семафоры используются для синхронизации операций внутри видеокарты. Нам нужно синхронизировать операции с очередями графических команд и очередями отображения, для этой цели больше подойдут семафоры.

## Семафоры

Мы будем использовать два семафора. Первый семафор сообщит нам о том, что image получен и готов к рендерингу, а второй сообщит об окончании рендеринга и о том, что image можно выводить на экран. Создадим два члена класса `VkSemaphore`:

```cpp
VkSemaphore imageAvailableSemaphore;
VkSemaphore renderFinishedSemaphore;
```

Чтобы создать семафоры, добавим функцию `createSemaphores`:

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
    createSemaphores();
}

...

void createSemaphores() {

}
```

Для создания семафоров требуется заполнить структуру [VkSemaphoreCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSemaphoreCreateInfo.html). В текущей версии API она содержит всего одно обязательное поле — `sType`:

```cpp
void createSemaphores() {
    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;
}
```

Создадим семафоры уже знакомым нам способом — с помощью вызова функции [vkCreateSemaphore](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateSemaphore.html):

```cpp
if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphore) != VK_SUCCESS ||
    vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphore) != VK_SUCCESS) {

    throw std::runtime_error("failed to create semaphores!");
}
```

Семафоры нужно очистить в конце работы программы после выполнения всех команд, когда синхронизация больше не требуется:

```cpp
void cleanup() {
    vkDestroySemaphore(device, renderFinishedSemaphore, nullptr);
    vkDestroySemaphore(device, imageAvailableSemaphore, nullptr);
```

## Получение image из swap chain

Как уже говорилось, первое, что мы должны сделать в функции `drawFrame` — это получить image из swap chain. Напомним, что swap chain — это часть расширения Vulkan, поэтому в названии функции должно быть `vk * KHR`:

```cpp
void drawFrame() {
    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
}
```

Первые два параметра, передаваемые в `vkAcquireNextImageKHR`, — это логическое устройство и swap chain, из которой нужно получить image. Третий параметр указывает время ожидания в наносекундах, по истечении которого image станет доступным. Использование максимального значения в виде 64-битного целого числа без знака отключает время ожидания.

Следующие два параметра — это синхронизаторы, которые переходят в состояние signaled, когда presentation engine закончит работу с image, после чего мы можем начать отрисовку в него. Здесь можно указать семафор, барьер или и то, и другое. Мы используем `imageAvailableSemaphore`.

Последний параметр указывает переменную для получения индекса уже доступного image из swap chain. Индекс ссылается на [VkImage](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImage.html) в массиве `swapChainImages`. Мы будем использовать его для выбора соответствующего буфера команд.

## Отправка буфера команд

Отправка в очередь и синхронизация настраиваются в структуре [VkSubmitInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSubmitInfo.html).

```cpp
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

VkSemaphore waitSemaphores[] = {imageAvailableSemaphore};
VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
```

Первые три параметра указывают, какие семафоры необходимо дождаться перед началом выполнения и на каком этапе (или этапах) графического конвейера происходит ожидание. Прежде чем начать запись цвета в image, нам нужно дождаться, когда image станет доступен, поэтому укажем этап графического конвейера, который выполняет запись в цветовой буфер. Это значит, что теоретически выполнение нашего вершинного шейдера может начаться тогда, когда image еще не доступен. Каждый элемент в массиве `waitStages` соответствует семафору с тем же индексом в `pWaitSemaphores`.

```cpp
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[imageIndex];
```

Следующие два параметра указывают, какие буферы команд отправляются для выполнения. Мы должны отправить буфер команд, прикрепленный к image, который мы получили из swap chain.

```cpp
VkSemaphore signalSemaphores[] = {renderFinishedSemaphore};
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = signalSemaphores;
```

Параметры `signalSemaphoreCount` и `pSignalSemaphores` указывают, какие семафоры сообщат об окончании выполнения буферов команд. Мы используем `renderFinishedSemaphore`.

```cpp
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

Теперь можно отправить буфер команд в графическую очередь с помощью [vkQueueSubmit](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueueSubmit.html). Функция принимает массив структур [VkSubmitInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSubmitInfo.html) в качестве аргумента для большей эффективности при слишком большой нагрузке. Последний параметр указывает на опциональный барьер, который переходит в состояние signaled при завершении выполнения буферов команд. Для синхронизации мы используем семафоры, поэтому просто передадим `VK_NULL_HANDLE`.

## Зависимости подпроходов

Помните, что image layout автоматически передается между подпроходами в проходе рендера. Зависимости памяти и порядка выполнения между подпроходами контролируются зависимостями подпроходов. Хоть мы используем только один подпроход, операции непосредственно до него и сразу после него также считаются неявными «подпроходами».

Есть две встроенные зависимости, которые отвечают за передачу данных в начале и в конце прохода рендера, но первая передача не происходит в нужное время. Предполагается, что передача данных должна происходить в начале конвейера, но на этом этапе image еще не получен! Есть два пути решения этой проблемы. Можно изменить `waitStages` для `imageAvailableSemaphore` и использовать `VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`, чтобы не допустить начало прохода рендера до тех пор, пока image не станет доступен. Другой способ — заставить проход рендера ждать `VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT`. Я решил выбрать второй способ, поскольку он позволит лучше рассмотреть зависимости подпроходов и то, как они работают.

Зависимости подпроходов указываются через [VkSubpassDependency](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSubpassDependency.html). Добавьте в функцию `createRenderPass` следующий код:

```cpp
VkSubpassDependency dependency{};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
```

Первые два параметра определяют индексы зависимых подпроходов. Значение `VK_SUBPASS_EXTERNAL` указывает на неявный подпроход перед или после прохода рендера в зависимости от того, где указано значение: в `srcSubpass` или `dstSubpass`. Индекс `0` указывает на наш единственный подпроход. Значение `dstSubpass` всегда должно быть больше, чем `srcSubpass`, чтобы не допустить циклов в графе зависимостей \(за исключением случаев, когда один из подпроходов равен `VK_SUBPASS_EXTERNAL`\).

```cpp
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
```

В следующих двух полях указаны операции, которые нужно ожидать, и этапы, на которых эти операции выполняются. Нам нужно дождаться, когда swap chain закончит считывание image, прежде чем мы сможем получить к нему доступ. Этого можно добиться, дождавшись output stage у записи в цветовой буфер.

```cpp
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
```

Эти настройки предотвратят передачу данных до тех пор, пока это не станет действительно необходимо \(и будет разрешено\): когда мы захотим записать цвет в буфер.

```cpp
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

В структуре [VkRenderPassCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkRenderPassCreateInfo.html) есть два поля для указания массива зависимостей.

## Отображение на экране

Последнее действие, которое осталось выполнить для отрисовки кадра — отправить результат обратно в swap chain, чтобы вывести на экран. Отображение на экране настраивается в структуре `VkPresentInfoKHR` в конце функции `drawFrame`.

```cpp
VkPresentInfoKHR presentInfo{};
presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = signalSemaphores;
```

Первые два параметра указывают, какие семафоры нужно дождаться перед началом отображения.

```cpp
VkSwapchainKHR swapChains[] = {swapChain};
presentInfo.swapchainCount = 1;
presentInfo.pSwapchains = swapChains;
presentInfo.pImageIndices = &imageIndex;
```

Следующие два параметра указывают swap chains для представления images и индекс image для каждого swap chain. Почти всегда будет использоваться только один swap chain.

```cpp
presentInfo.pResults = nullptr; // Optional
```

Последний опциональный параметр — `pResult`. Он позволяет указать массив значений [VkResult](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkResult.html) для проверки каждого swap chain при успешном отображении. В нем нет необходимости, если используется лишь один swap chain, поскольку в этом случае можно просто использовать возвращаемое значение функции.

```cpp
vkQueuePresentKHR(presentQueue, &presentInfo);
```

Функция `vkQueuePresentKHR` отправляет запрос на представление image в swap chain. Позже мы добавим обработку ошибок для `vkAcquireNextImageKHR` и `vkQueuePresentKHR`, поскольку их ошибки необязательно приводят к закрытию программы, в отличие от функций, встречавшихся ранее.

Если вы все сделали правильно, при запуске программы должно отобразиться что-то подобное:

![](0.png)

Этот треугольник может слегка отличаться от тех, что вы привыкли видеть в учебниках по графике. Дело в том, что в нашем руководстве шейдер интерполирует в линейном цветовом пространстве, а впоследствии преобразует в цветовое пространство sRGB. Чтобы узнать, в чем между ними отличие, перейдите по [ссылке](https://medium.com/@heypete/hello-triangle-meet-swift-and-wide-color-6f9e246616d9).

Ура! Получилось! К сожалению, при включенных слоях валидации программа будет падать при каждом закрытии. Сообщения, выводимые в терминал из `debugCallback`, укажут причину:

![](1.png)

Все операции в `drawFrame` выполняются асинхронно. Это значит, что при выходе из `mainLoop` операции рисования и отображения могут продолжаться. Освобождать ресурсы в такой момент — не самая лучшая идея.

Для решения этой проблемы перед выходом из `mainLoop` и уничтожением окна, нужно дождаться, когда логическое устройство завершит операции.

```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }

    vkDeviceWaitIdle(device);
}
```

Используйте [vkQueueWaitIdle](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueueWaitIdle.html), чтобы дождаться завершения операций в определенной очереди команд. Эта функция используется как примитивный способ выполнить синхронизацию. Теперь при закрытии программы не должно возникнуть никаких проблем.

## Кадры в конвейере \(Frames in flight\)

Если запустить программу с включенными слоями валидации, можно либо получить ошибки, либо заметить, что использование памяти постепенно растет. Дело в том, что программа сразу отправляет работу в функцию `drawFrame` и не проверяет, завершилась ли какая-нибудь ее часть. Если CPU отправляет работу быстрее, чем GPU успевает ее выполнять, очередь будет медленно заполняться работой. Но хуже то, что мы повторно используем семафоры `imageAvailableSemaphore` и `renderFinishedSemaphore` вместе с буферами команд одновременно для нескольких кадров!

Самый простой способ решить эту проблему — дождаться завершения работы сразу после отправки, например, с помощью [vkQueueWaitIdle](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueueWaitIdle.html):

```cpp
void drawFrame() {
    ...

    vkQueuePresentKHR(presentQueue, &presentInfo);

    vkQueueWaitIdle(presentQueue);
}
```

Однако в этом случае графический конвейер используется неоптимально, т.к. за раз только один кадр может пройти через конвейер. Этапы, через которые прошел кадр, освобождаются и уже могут использоваться для следующего кадра. Расширим нашу программу, чтобы каждый освободившийся этап начинал обрабатывать следующий кадр.

Сначала добавим константу, указывающую, сколько кадров должно обрабатываться одновременно:

```cpp
const int MAX_FRAMES_IN_FLIGHT = 2;
```

Для каждого кадра должен быть свой набор семафоров:

```cpp
std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
```

Изменим функцию `createSemaphores`, чтобы их создать:

```cpp
void createSemaphores() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create semaphores for a frame!");
        }
}
```

Также все семафоры должны быть очищены:

```cpp
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
    }

    ...
}
```

Чтобы каждый раз использовать соответствующую пару семафоров, нужно отслеживать текущий кадр. Для этого используем индекс кадра:

```cpp
size_t currentFrame = 0;
```

Теперь изменим функцию `drawFrame`, чтобы использовать соответствующие семафоры:

```cpp
void drawFrame() {
    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};

    ...

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphores[currentFrame]};

    ...
}
```

Не забывайте каждый раз переходить к следующему кадру:

```cpp
void drawFrame() {
    ...

    currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```

Использование деления по модулю \(%\) гарантирует, что индекс кадра закольцовывается после каждого `MAX_FRAMES_IN_FLIGHT` кадра в очереди.

Несмотря на то, что мы настроили необходимые объекты для облегчения обработки нескольких кадров одновременно, мы все еще не препятствуем отправке более `MAX_FRAMES_IN_FLIGHT` кадров. Сейчас используется только синхронизация GPU-GPU без синхронизации CPU-GPU. Мы можем использовать объекты кадра #0, в то время как сам кадр #0 все еще находится в конвейере \(in flight\)!

Для выполнения синхронизации CPU-GPU Vulkan предлагает использовать барьеры \(fences\). Барьеры похожи на семафоры тем, что они используются для ожидания и уведомляют об окончании операций, но на этот раз мы пропишем их ожидание в нашем коде. Сначала создадим барьер для каждого кадра:

```cpp
std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;
```

Я решил создать барьеры вместе с семафорами, поэтому переименовал `createSemaphores` в `createSyncObjects`:

```cpp
void createSyncObjects() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo{};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceInfo{};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS ||
            vkCreateFence(device, &fenceInfo, nullptr, &inFlightFences[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create synchronization objects for a frame!");
        }
    }
}
```

Создание барьеров \([VkFence](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkFence.html)\) очень похоже на создание семафоров. Также не забудьте выполнить очистку:

```cpp
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    ...
}
```

Теперь изменим функцию `drawFrame`, чтобы использовать барьеры. Вызов [vkQueueSubmit](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkQueueSubmit.html) содержит опциональный параметр для передачи барьера, который переходит в состояние signaled по окончании выполнения буфера команд. Мы можем использовать барьер для уведомления о создании кадра.

```cpp
void drawFrame() {
    ...

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
        throw std::runtime_error("failed to submit draw command buffer!");
    }
    ...
}
```

Теперь осталось изменить начало функции `drawFrame`, чтобы дождаться создания кадра:

```cpp
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    ...
}
```

Функция [vkWaitForFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkWaitForFences.html) принимает массив барьеров и ждет, когда один из них или все барьеры перейдут в состояние signaled перед возвратом функции. `VK_TRUE` обозначает, что выполняется ожидание всех барьеров, но в случае с одним барьером, очевидно, это не имеет значения. Как и `vkAcquireNextImageKHR`, эта функция также принимает в качестве параметра время ожидания. В отличие от семафоров, нам нужно вручную установить барьер в состояние unsignaled, сбросив его с помощью вызова [vkResetFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkResetFences.html).

Если запустить программу сейчас, можно заметить, что что-то не так. Кажется, будто ничего не рендерится. Если у вас были включены слои валидации, появится следующее сообщение:

![](2.png)

Это значит, что ожидаемый барьер не был отправлен. Проблема в том, что по умолчанию барьеры создаются в состоянии unsignaled. Поэтому, если барьер не использовался ранее, [vkWaitForFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkWaitForFences.html) будет ждать бесконечно. Для решения этой проблемы мы можем изменить создание барьера, чтобы инициализировать его в состоянии signaled:

```cpp
void createSyncObjects() {
    ...

    VkFenceCreateInfo fenceInfo{};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    ...
}
```

Утечка памяти устранена, но программа работает не совсем корректно. Если число `MAX_FRAMES_IN_FLIGHT` больше, чем количество images в swap chain или `vkAcquireNextImageKHR` возвращает images не по порядку, есть вероятность, что мы можем начать рендеринг в image, который уже находится в конвейере \(in flight\). Чтобы этого избежать, нужно отслеживать каждый image из swap chain и проверять, не использует ли его в текущий момент кадр в конвейере.

Для этого добавим новый список `imagesInFlight`:

```cpp
std::vector<VkFence> inFlightFences;
std::vector<VkFence> imagesInFlight;
size_t currentFrame = 0;
```

Подготовим его в `createSyncObjects`:

```cpp
void createSyncObjects() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);
    imagesInFlight.resize(swapChainImages.size(), VK_NULL_HANDLE);

    ...
}
```

Изначально ни один кадр не использует image, поэтому мы явно инициализируем его как "без барьеров". Изменим функцию `drawFrame`, чтобы дождаться любого из предыдущих кадров, использующих image, который мы только что ассигновали новому кадру:

```cpp
void drawFrame() {
    ...

    vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    // Check if a previous frame is using this image (i.e. there is its fence to wait on)
    if (imagesInFlight[imageIndex] != VK_NULL_HANDLE) {
        vkWaitForFences(device, 1, &imagesInFlight[imageIndex], VK_TRUE, UINT64_MAX);
    }
    // Mark the image as now being in use by this frame
    imagesInFlight[imageIndex] = inFlightFences[currentFrame];

    ...
}
```

Поскольку вызовов [vkWaitForFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkWaitForFences.html) стало больше, вызов [vkResetFences](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkResetFences.html) следует переместить. Лучше всего вызвать его прямо перед использованием барьера:

```cpp
void drawFrame() {
    ...

    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
        throw std::runtime_error("failed to submit draw command buffer!");
    }

    ...
}
```

Мы настроили всю необходимую синхронизацию, чтобы в очередь не отправлялось более двух кадров и чтобы эти кадры случайно не использовали один и тот же image. Обратите внимание, что, например для окончательной очистки, может использоваться более грубая синхронизация, такая как [vkDeviceWaitIdle](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDeviceWaitIdle.html). Вам нужно решить, какую синхронизацию использовать, исходя из функциональных требований.

Чтобы узнать о синхронизации больше, предлагаем ознакомиться с подробным обзором [Khronos](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present).

## Выводы

Написав чуть более 900 строк кода, мы наконец-то видим результат нашей работы на экране! Безусловно, настройка программы с Vulkan занимает немало времени, но благодаря тому, что состояния в Vulkan должны описываться явно, вы можете контролировать многие процессы. Я рекомендую вам потратить чуть больше времени и еще раз перечитать код, чтобы понять назначение всех объектов Vulkan и то, как они соотносятся друг с другом.

В следующей главе мы рассмотрим еще одну деталь, необходимую для правильной работы программы с Vulkan.

[C++ code](15_hello_triangle.cpp) / [Vertex shader](09_shader_base.vert) / [Fragment shader](09_shader_base.frag)
