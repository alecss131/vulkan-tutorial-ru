# Vulkan. Руководство разработчика. Window surface

Поскольку Vulkan API полностью независим от платформы, он не может напрямую взаимодействовать с оконной системой. Чтобы Vulkan мог выводить результат на экран, необходимо использовать стандартизованные расширения WSI \(Window System Integration\). В этой главе мы расскажем про одно из них — `VK_KHR_surface`. Расширение предоставляет объект `VkSurfaceKHR` — абстрактный тип поверхности для показа отрендеренных изображений. Эта поверхность будет создана при поддержке окна GLFW, полученного нами ранее.

`VK_KHR_surface` – это расширение Vulkan уровня экземпляра. У нас оно уже подключено, поскольку находится в списке расширений, возвращаемых функцией `glfwGetRequiredInstanceExtensions`. В списке есть и другие расширения WSI, которые мы будем использовать в следующих главах.

Window surface нужно создать сразу после VkInstance, поскольку это может повлиять на выбор физического устройства. Нужно помнить, что window surfaces — полностью опциональный компонент Vulkan. Вы можете обойтись без него, если вам нужен offscreen рендеринг. Это позволяет избежать таких хаков, как, например, создание невидимого окна для OpenGL.

## Создание window surface

Начнем с того, что добавим новый член класса surface сразу после вызова `debugMessenger`.

```cpp
VkSurfaceKHR surface;
```

Процесс создания объекта `VkSurfaceKHR` зависит от платформы. Так, например, для создания в Windows нужны дескрипторы `HWND` и `HMODULE`. Для разных платформ у Vulkan есть платформенно-зависимое дополнение к расширению, которое в Windows называется `VK_KHR_win32_surface`. Оно также автоматически включено в список расширений, возвращаемых функцией `glfwGetRequiredInstanceExtensions`.

Мы покажем, как можно использовать это расширение для создания surface в Windows, однако в руководстве оно нам не понадобится. В библиотеке GLFW, которую мы используем, есть функция-обертка `glfwCreateWindowSurface`, содержащая специфичный для платформы код. Но было бы не плохо увидеть, что происходит за кулисами.

Window surface – это объект Vulkan, поэтому мы должны заполнить структуру `VkWin32SurfaceCreateInfoKHR`, чтобы создать его. В ней есть два важных параметра: `hwnd` и `hinstance` — дескрипторы окна и текущего процесса.

```cpp
VkWin32SurfaceCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

Здесь для получения сырого `HWND` используется функция `glfwGetWin32Window`, а для получения `HINSTANCE` - функция `GetModuleHandle`.

После этого мы можем создать surface с помощью функции `vkCreateWin32SurfaceKHR`, в которую передаются следующие параметры: экземпляр Vulkan, информация о surface, кастомный аллокатор и указатель для записи результата. Технически это функция расширения WSI, но она используется так часто, что была включена в стандартный загрузчик Vulkan, поэтому вам не нужно загружать ее явно.

```cpp
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

Для других платформ подход аналогичен. Для Linux, например, используется функция `vkCreateXcbSurfaceKHR`.

Функция `glfwCreateWindowSurface` делает именно эту работу, но имеет свою реализацию для каждой платформы. Интегрируем ее в нашу программу. Для этого добавим функцию `createSurface`, которая вызывается из `initVulkan` сразу после `createInstance` и `setupDebugMessenger`.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

Вместо структуры для вызова GLFW нужны простые параметры, что упрощает реализацию функции:

```cpp
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

В функцию передаются следующие параметры: [VkInstance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstance.html), указатель на окно GLFW, кастомный аллокатор и указатель для записи результата. Функция возвращает [VkResult](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkResult.html).

У GLFW нет специальной функции для уничтожения surface, но это легко можно сделать непосредственно с помощью Vulkan:

```cpp
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
    }
```

Не забудьте уничтожить surface до VkInstance.

## Проверка поддержки отображения

Хотя конкретная реализация Vulkan может поддерживать интеграцию с оконной системой, это не значит что каждое из устройств в системе это тоже поддерживает. Поэтому нам нужно расширить `isDeviceSuitable`, чтобы быть уверенными, что устройство может отображать изображения на surface, которую мы создали. Поскольку отображение — это процесс, происходящий через очереди команд, то задача заключается в том, чтобы найти семейство очередей, которое поддерживает отображение в созданную surface.

Вполне возможно, что семейства, поддерживающие команды рисования, и семейства, поддерживающие отображение, не будут совпадать. Поэтому мы должны изменить структуру `QueueFamilyIndices`, чтобы учитывать этот факт.

```cpp
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

Изменим функцию `findQueueFamilies`, чтобы найти семейство очередей с поддержкой отображения на surface нашего окна. Для проверки используем функцию `vkGetPhysicalDeviceSurfaceSupportKHR`, которая принимает следующие параметры: физическое устройство, индекс семейства очередей и surface. Добавим вызов функции в тот же цикл, в котором находится проверка `VK_QUEUE_GRAPHICS_BIT`:

```cpp
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

Затем проверим значение типа `VkBool32` и сохраним индекс нужного нам семейства:

```cpp
if (presentSupport) {
    indices.presentFamily = i;
}
```

Обратите внимание, что очень вероятно, что в конечном итоге это будет одно и то же семейство очередей, но мы будем рассматривать их, как если бы они были отдельными очередями. Однако вы можете отдать предпочтение физическому устройству с поддержкой графических операций и с поддержкой отображения в одной очереди.


## Создание очереди отображения

Осталось изменить процесс создания логического устройства, чтобы создать очередь с поддержкой отображения и получить дескриптор [VkQueue](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkQueue.html). Добавим переменную класса:

```cpp
VkQueue presentQueue; 
```

Нам нужно несколько [VkDeviceQueueCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDeviceQueueCreateInfo.html), чтобы создать очередь для каждого из семейств. Элегантный способ это сделать — использовать `std::set`, чтобы выделить уникальные семейства из найденных:

```cpp
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo{};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

Изменим структуру [VkDeviceCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDeviceCreateInfo.html), чтобы она указывала на наш вектор:

```cpp
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

В результате, если семейство для рисования и отображения одно и тоже, то его индекс будет передан один раз.

Наконец добавим вызов для получения дескриптора очереди. Если семейство очередей одно, дескрипторы должны иметь одинаковое значение.

```cpp
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

В следующей главе мы рассмотрим swap chains и расскажем, как они помогают выводить изображения на экран.

[Код C++](05_window_surface.cpp)
