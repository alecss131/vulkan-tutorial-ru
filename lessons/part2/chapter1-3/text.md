# Vulkan. Руководство разработчика. Устройства и очереди

## Физические устройства и семейства очередей

### Выбор физического устройства

После инициализации библиотеки Vulkan через [VkInstance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstance.html) нам нужно подобрать видеокарту с поддержкой всех необходимых нам функций. Можно выбрать сразу несколько видеокарт и использовать их одновременно, но в руководстве мы будем использовать только одну, которая отвечает всем нашим требованиям.

Добавим функцию `pickPhysicalDevice` и добавим ее вызов в функцию `initVulkan`.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```

Для ссылки на выбранную видеокарту используется дескриптор [VkPhysicalDevice](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPhysicalDevice.html), который добавлен как новый член класса. Он будет уничтожен вместе с [VkInstance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstance.html), поэтому не нужно вносить никаких изменений в функцию `cleanup`.

```cpp
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```

Составление списка видеокарт похоже на составление списка расширений и начинается с запроса их количества.

```cpp
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```

Если устройства, поддерживающие Vulkan, не найдены, нет смысла выполнять дальнейшие действия.

```cpp
if (deviceCount == 0) {
    throw std::runtime_error("failed to find GPUs with Vulkan support!");
}
```

Если устройства найдены, выделите массив для хранения дескрипторов [VkPhysicalDevice](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPhysicalDevice.html).

```cpp
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

Теперь нужно определить, подходят ли эти устройства для выполнения необходимых нам операций. Для этого будем использовать новую функцию `isDeviceSuitable`, но сейчас реализуем только заготовку. Мы вернемся к ней позже.

```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

А сейчас проверим, есть ли хотя бы одно устройство, которое удовлетворяет нашим требованиям.

```cpp
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("failed to find a suitable GPU!");
}
```

В следующем разделе мы расскажем, проверку каких требований нужно сделать в функции `isDeviceSuitable`. Поскольку в дальнейшем мы будем использовать больше возможностей Vulkan, мы также расширим эту функцию и добавим больше проверок.

### Проверка соответствия устройства

Чтобы проверить, отвечает ли устройство заданным требованиям, вы можете запросить дополнительные данные. Основные свойства устройства, такие как имя, тип и поддерживаемая версия Vulkan, запрашиваются с помощью vkGetPhysicalDeviceProperties.

```cpp
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

Информация о поддержке опциональных возможностей, таких как, сжатие текстур, 64-битные числа с плавающей точкой и рендеринг в несколько viewport-ов (multi viewport rendering) запрашивается с помощью [vkGetPhysicalDeviceFeatures](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkGetPhysicalDeviceProperties.html):

```cpp
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```

Вы можете запросить и другие данные, связанные с памятью устройства и семействами очередей, которые мы обсудим чуть позже \(см. следующую главу\).

Приведем пример. Представим, что нашу программу можно использовать только с дискретными видеокартами, которые поддерживают геометрические шейдеры. Тогда функция `isDeviceSuitable` будет иметь вид:

```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```

Вместо обычной проверки вы можете дать оценку каждому устройству, а затем выбрать устройство с самой высокой оценкой. Это позволит вам использовать дискретную видеокарту, если она доступна, но при этом не отсеивать интегрированную, если выбора нет.

Вы можете реализовать что-то подобное:

```cpp
#include <map>

...

void pickPhysicalDevice() {
    ...

    // Use an ordered map to automatically sort candidates by increasing score
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // Check if the best candidate is suitable at all
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // Discrete GPUs have a significant performance advantage
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // Maximum possible size of textures affects graphics quality
    score += deviceProperties.limits.maxImageDimension2D;

    // Application can't function without geometry shaders
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```

Ваша реализация может отличаться от этой, мы лишь хотим показать, как можно построить процесс выбора устройства. Разумеется, вы можете просто спросить пользователя, какое устройство использовать.

Поскольку мы только начинаем изучение Vulkan, нам подойдет любой графический процессор с поддержкой Vulkan:

```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

### Семейства очередей

Как уже упоминалось, почти каждая операция в Vulkan требует отправки команд в очереди. Разные типы очередей образуются из разных *семейств очередей*, и каждое семейство допускает только определенное подмножество команд. Например, одно семейство очередей может обрабатывать только команды вычислений, другое семейство очередей обрабатывает только команды, связанные с перемещением данных в памяти.

Нужно проверить, какие семейства очередей поддерживает наше устройство и какое из этих семейств поддерживает необходимые нам команды. Для этого добавим новую функцию `findQueueFamilies`, которая будет искать подходящие семейства очередей.

Сейчас нам надо найти очередь, которая поддерживает графические команды. Так что функция будет выглядеть как-то так:

```cpp
uint32_t findQueueFamilies(VkPhysicalDevice device) {
    // Logic to find graphics queue family
}
```

Однако в дальнейшем мы будем искать и другие очереди, поэтому лучше поместить индексы в структуру:

```cpp
struct QueueFamilyIndices {
    uint32_t graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // Logic to find queue family indices to populate struct with
    return indices;
}
```

Но что, если семейство очередей недоступно? Можно сгенерировать исключение в `findQueueFamilies`, но такой способ не совсем подходит для выбора подходящего устройства. Так например, нам предпочтительнее использовать устройство с отдельным семейством очередей для передачи данных, но это не является обязательным требованием. Поэтому нам нужен способ обозначить, доступны ли отдельные семейства.

Нет такого магического значения, указывающего на отсутствие семейства очередей, поскольку любое значение `unit32_t`, в теории, может быть валидным индексом семейства очередей, включая 0. К счастью, в C++ появился шаблонный класс, который позволяет определить, существует ли значение или нет:

```cpp
#include <optional>

...

std::optional<uint32_t> graphicsFamily;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // false

graphicsFamily = 0;

std::cout << std::boolalpha << graphicsFamily.has_value() << std::endl; // true
```

`std::optional` – этот объект-обертка не содержит значения, пока вы не присвоите ему что-либо. В любой момент вы можете узнать, существует ли значение, с помощью метода `has_value()`. Это значит, что мы можем изменить логику:

```cpp
#include <optional>

...

struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
};

QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;
    // Assign index to queue families that could be found
    return indices;
}
```

Начнем реализацию `findQueueFamilies`:

```cpp
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```

Мы используем [vkGetPhysicalDeviceQueueFamilyProperties](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkGetPhysicalDeviceQueueFamilyProperties.html) уже известным вам способом:

```cpp
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

Структура [VkQueueFamilyProperties](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkQueueFamilyProperties.html) содержит информацию о семействе очередей, включая типы поддерживаемых операций и количество очередей, которые можно создать. Нам нужно найти хотя бы одно семейство, которое поддерживает `VK_QUEUE_GRAPHICS_BIT`.

```cpp
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    i++;
}
```

Теперь, когда у нас есть функция для поиска семейств, мы можем использовать ее в функции `isDeviceSuitable`, чтобы проверить, может ли устройство обрабатывать нужные нам команды:

```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.graphicsFamily.has_value();
}
```

Для удобства мы также добавим общую проверку в саму структуру:

```cpp
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};

...

bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```

Теперь мы можем использовать ее для раннего выхода из `findQueueFamilies`:

```cpp
for (const auto& queueFamily : queueFamilies) {
    ...

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```

Отлично! Мы сделали все необходимое для того, чтобы найти подходящее физическое устройство! Следующим шагом будет создание логического устройства.

[Код C++](03_physical_device_selection.cpp)

## Логическое устройство и семейства очередей

### Вступление

После выбора физического устройства необходимо создать *логическое устройство*. Создание логического устройства похоже на создание VkInstance и включает в себя описание возможностей, которые мы будем использовать. Нам нужно указать, какие очереди из имеющихся доступных семейств необходимо создать. При необходимости из одного физического устройства можно создать несколько логических устройств.

Начнем с добавления нового члена класса для хранения дескриптора логического устройства.

```cpp
VkDevice device;
```

Добавим функцию `createLogicalDevice` и ее вызов из функции `initVulkan`.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

### Указание очередей, которые нужно создать

Чтобы создать логическое устройство, нам снова надо указать множество деталей, используя структуры. Первая из них – это [VkDeviceQueueCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDeviceQueueCreateInfo.html). В этой структуре задается необходимое количество очередей для одного семейства. На данном этапе нас интересует только очередь с поддержкой графических операций.

```cpp
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo{};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

Современные драйверы позволяют создать лишь небольшое количество очередей в каждом семействе. На самом деле, нам и не нужно больше чем одна очередь. Всё потому, что мы можем создавать буферы команд в нескольких потоках, а затем отправлять их все в главном потоке с помощью одного вызова и с минимальными затратами.

Каждая очередь имеет приоритет \(число с плавающей точкой от 0 до 1\), который влияет на порядок выполнения командных буферов. Приоритет необходимо указать даже в случае, если мы используем всего одну очередь:

```cpp
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```


### Указание используемых возможностей устройства

Это те возможности, которые мы запрашивали в предыдущей главе с помощью [vkGetPhysicalDeviceFeatures](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkGetPhysicalDeviceFeatures.html). Сейчас это только поддержка геометрических шейдеров, другие возможности нам пока не нужны, поэтому оставим всё равным `VK_FALSE`. Мы еще вернемся к этой структуре, как только перейдем к более интересным вещам.

```cpp
VkPhysicalDeviceFeatures deviceFeatures{};
```

### Создание логического устройства

После того, как мы подготовили предыдущие структуры, можно перейти к заполнению главной структуры [VkDeviceCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDeviceCreateInfo.html).

```cpp
VkDeviceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

Для начала добавим указатели на структуры с информацией об очередях и возможностях устройства:

```cpp
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

Остальная информация напоминает структуру [VkInstanceCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstanceCreateInfo.html), в которой требуется указать расширения и слои валидации. Только на этот раз используются параметры для конкретного устройства.

Примером расширения, применяемого к конкретному устройству, является `VK_KHR_swapchain`. Оно позволяет выводить отрендеренные изображения на экран. Однако в системе могут быть и такие устройства, которые не поддерживают это расширение, потому что они, например, реализуют только вычислительные операции. Мы еще вернемся к этому расширению в главе, посвященной swap chain.

В ранних реализациях Vulkan было принято разграничивать слои валидации для экземпляра и для конкретного устройства. На сегодняшний день такой [подход устарел](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#extendingvulkan-layers-devicelayerdeprecation), и поля `enabledLayerCount` и `ppEnabledLayerNames` в структуре [VkDeviceCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDeviceCreateInfo.html) не учитываются. Тем не менее, мы советуем настроить эти параметры для совместимости с более ранними реализациями:

```cpp
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

Расширения для конкретного устройства нам пока не понадобятся.

Теперь мы можем создать логическое устройство с помощью вызова в функции [vkCreateDevice](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateDevice.html).

```cpp
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

Подобно функции создания `VkInstance` эта функция может вернуть код ошибки, связанной с включением несуществующих расширений или указанием неподдерживаемых возможностей.

Устройство нужно будет уничтожить в `cleanup` с помощью функции [vkDestroyDevice](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDestroyDevice.html):

```cpp
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

`VkInstance` не передается в качестве аргумента, потому что логическое устройство с ним напрямую не взаимодействует.

### Получение дескрипторов очередей

Очереди создаются автоматически вместе с логическим устройством, но у нас пока нет дескриптора для взаимодействия с ними. Чтобы его получить, добавим член класса для хранения дескриптора очереди графических команд:

```cpp
VkQueue graphicsQueue;
```

Очереди устройства будут уничтожены вместе с устройством, поэтому не нужно вносить никаких изменений в функцию `cleanup`.

Чтобы получить дескрипторы для каждой очереди, можно использовать функцию [vkGetDeviceQueue](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkGetDeviceQueue.html). В функцию передаются следующие параметры: логическое устройство, индекс семейства очередей, индекс очереди внутри семейства и указатель на переменную для хранения дескриптора очереди. Поскольку мы создаем только одну очередь, мы будем использовать индекс 0.

```cpp
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

Имея дескрипторы логического устройства и очереди мы наконец-то сможем что-то сделать! В следующих главах мы будем настраивать ресурсы для вывода результата на экран.

[Код C++](04_logical_device.cpp)
