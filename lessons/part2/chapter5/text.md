# Vulkan. Руководство разработчика. Пересоздание swap chain

## Вступление

Теперь наша программа успешно справляется с отрисовкой треугольника, но есть то, что она пока не умеет обрабатывать. Window surface может измениться так, что swap chain больше не будет с ней совместима. Такое может произойти, например, из-за изменение размера окна. Мы должны вовремя заметить это и обновить swap chain.

## Пересоздание swap chain

Создадим новую функцию `recreateSwapChain`, которая вызывает `createSwapChain` и другие функции создания объектов, зависящих от swap chain или размера окна.

```cpp
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```

Сначала вызовем [vkDeviceWaitIdle](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDeviceWaitIdle.html), т.к. мы не должны затрагивать ресурсы, которые все еще могут использоваться. Очевидно, что первым делом нужно заново создать сам swap chain. Также нужно пересоздать image views, т. к. они основываются непосредственно на images из swap chain. Render pass нужно создать заново, т.к. он зависит от формата images из swap chain. Несмотря на то, что изменение размера окна редко приводит к изменению формата image, мы все равно должны выполнить это действие. Размер вьюпорта и прямоугольника отсечения \(scissor rectangle\) определяется во время создания `VkPipeline`, поэтому конвейер тоже нужно пересоздать. Однако, чтобы не пересоздавать конвейер, для вьюпорта и прямоугольника отсечения можно использовать динамическое состояние. Наконец, обновим фреймбуферы и буферы команд, т. к. они напрямую зависят от images из swap chain.

Чтобы убедиться, что старые версии этих объектов удалены, мы должны переместить часть кода очистки в отдельную функцию, которая вызывается из `recreateSwapChain`. Назовем ее `cleanupSwapChain`:

```cpp
void cleanupSwapChain() {

}

void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```

Переместим код очистки всех заново созданных объектов из `cleanup` в `cleanupSwapChain`:

```cpp
void cleanupSwapChain() {
    for (size_t i = 0; i < swapChainFramebuffers.size(); i++) {
        vkDestroyFramebuffer(device, swapChainFramebuffers[i], nullptr);
    }

    vkFreeCommandBuffers(device, commandPool, static_cast<uint32_t>(commandBuffers.size()), commandBuffers.data());

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        vkDestroyImageView(device, swapChainImageViews[i], nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

Мы могли бы воссоздать пул команд с нуля, но это довольно затратно. Вместо этого я удалил существующие буферы команд с помощью функции [vkFreeCommandBuffers](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkFreeCommandBuffers.html). Теперь мы можем повторно использовать существующий пул для выделения новых буферов команд.

Обратите внимание, что в `chooseSwapExtent` мы уже запрашиваем новое разрешение окна, чтобы для images из swap chain использовался \(новый\) правильный размер, поэтому `chooseSwapExtent` изменять не нужно \(вспомните, что нам уже приходилось использовать `glfwGetFramebufferSize`, чтобы получить разрешение surface в пикселях при создании swap chain\).

Это все, что нужно для пересоздания swap chain! Недостаток такого подхода в том, что перед созданием новой swap chain необходимо остановить весь рендеринг. Однако вы можете создать новую swap chain, пока команды рисования из старой swap chain все еще находятся в конвейере (in-flight). Для этого нужно указать старую swap chain в поле `oldSwapChain` в структуре `VkSwapchainCreateInfoKHR` и уничтожить ее после использования.

## Неоптимальная или устаревшая swap chain

Теперь осталось выяснить, в какой момент нужно пересоздать swap chain и вызвать функцию `recreateSwapChain`. К счастью, Vulkan сам сообщит, что swap chain больше не подходит для использования. Функции `vkAcquireNextImageKHR` и `vkQueuePresentKHR` могут возвращать следующие значения, указывающие на это:

- `VK_ERROR_OUT_OF_DATE_KHR`: swap chain стала несовместима с surface и больше не может использоваться для рендеринга. Обычно это происходит после изменения размера окна.
- `VK_SUBOPTIMAL_KHR`: swap chain не полностью соответствует surface, но по-прежнему может использоваться для успешного отображения на экране.

```cpp
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```

Если при попытке получить image выясняется, что swap chain устарела, то его будет невозможно использовать для отображения. Поэтому мы должны немедленно пересоздать swap chain и повторить попытку в следующем вызове `drawFrame`.

Вы можете заново создать swap chain и в том случае, если она стала неоптимальной, но я решил не делать этого, поскольку мы уже получили от нее image. Значения `VK_SUCCESS` и `VK_SUBOPTIMAL_KHR` указывают на успешное завершение.

```cpp
result = vkQueuePresentKHR(presentQueue, &presentInfo);

if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    throw std::runtime_error("failed to present swap chain image!");
}

currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

Функция `vkQueuePresentKHR` возвращает те же самые значения. В данном случае мы пересоздадим неоптимальную swap chain, т. к. хотим получить наилучший возможный результат.

## Явная обработка изменения размеров окна

Хотя многие драйверы и платформы после изменения размера окна автоматически запускают `VK_ERROR_OUT_OF_DATE_KHR`, нет абсолютной уверенности, что это произойдет. Именно поэтому добавим дополнительный код для явной обработки изменения размеров. Сначала добавим новую переменную класса, которая сообщит о том, что размер окна изменился:

```cpp
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;

bool framebufferResized = false;
```

Функцию `drawFrame` также необходимо изменить, чтобы проверить этот флаг:

```cpp
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```

Важно сделать это после `vkQueuePresentKHR`, чтобы семафоры находились в надлежащем состоянии. В противном случае при ожидание семафоров могут возникнуть ошибки. Теперь, чтобы обнаружить изменение размеров окна, мы можем использовать функцию `glfwSetFramebufferSizeCallback` во фреймворке GLFW для настройки callback-функции:

```cpp
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```

Причина, по которой мы использовали `static` функцию в качестве callback-функции в том, что GLFW не знает, как нужно вызывать функцию класса с правильным указателем `this` на наш экземпляр \(instance\) `HelloTriangleApplication`.

Обратите внимание, мы получаем указатель на `GLFWwindow` в callback-функции. Мы можем привязать к окну указатель на произвольный объект, используя `glfwSetWindowUserPointer`:

```cpp
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

А теперь этот указатель можно получить внутри callback-функции с помощью `glfwGetWindowUserPointer` и установить флаг `framebufferResized`:

```cpp
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```

Теперь попробуйте запустить программу и изменить размер окна, чтобы проверить, изменятся ли размеры фреймбуфера в соответствии с размером окна.

## Обработка сворачивания окна

Есть еще один случай, когда swap chain может устареть. Это особый вид изменения размера окна — сворачивание окна. Мы назвали его особым, т.к. в результате размер фреймбуфера будет равен `0`. В этом уроке мы будем обрабатывать сворачивание окна, просто приостанавливая рендер до тех пор, пока окно не будет развернуто. Для этого расширим функцию `recreateSwapChain`:

```cpp
void recreateSwapChain() {
    int width = 0, height = 0;
    glfwGetFramebufferSize(window, &width, &height);
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```

Поздравляем, создание вашей первой рабочей программы с Vulkan завершено! В следующей главе мы избавимся от нашего хака с вершинами внутри шейдера, используя для этого вершинные буферы.

[C++ code](16_swap_chain_recreation.cpp) / [Vertex shader](09_shader_base.vert) / [Fragment shader](09_shader_base.frag)
