# Vulkan. Руководство разработчика. Swap chain

В Vulkan нет такого понятия, как default framebuffer, поэтому ему нужна инфраструктура с буферами, куда будут рендериться изображения перед выводом на экран. Такая инфраструктура называется swap chain, и ее нужно явно создать в Vulkan. Swap chain – это очередь из изображений, ожидающих вывода на экран. Программа сначала запрашивает объект `image(VkImage)`, в который будет рисовать, а после отрисовки отправляет его обратно в очередь. То, каким именно образом работает очередь, зависит от настроек, но основная задача swap chain – синхронизировать вывод изображений с частотой обновления экрана.

## Проверка поддержки swap chain

Некоторые специализированные видеокарты не имеют выходов для дисплея и поэтому не могут выводить изображения на экран. Кроме того, отображение на экран привязано к оконной системе и не является частью ядра Vulkan. Поэтому нам нужно подключить расширение `VK_KHR_swapchain`.

Для начала изменим функцию `isDeviceSuitable`, чтобы проверить, поддерживается ли расширение. Ранее мы уже работали со списком поддерживаемых расширений, поэтому сложностей возникнуть не должно. Обратите внимание, что заголовочный файл Vulkan предоставляет удобный макрос `VK_KHR_SWAPCHAIN_EXTENSION_NAME`, который определен как "`VK_KHR_swapchain`". Преимущество этого макроса в том, что, если вы допустите ошибку в написании, компилятор вас об этом предупредит.

Начнем с того, что объявим список требуемых расширений.

```cpp
const std::vector<const char*> deviceExtensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME
};
```

Для дополнительной проверки создадим новую функцию `checkDeviceExtensionSupport`, вызываемую из `isDeviceSuitable`:

```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    bool extensionsSupported = checkDeviceExtensionSupport(device);

    return indices.isComplete() && extensionsSupported;
}

bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    return true;
}
```

Изменим тело функции, чтобы проверить, все ли нужные нам расширения есть в списке поддерживаемых.

```cpp
bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());

    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

Здесь я использовал `std::set<std::string>`, чтобы хранить имена требуемых, но еще не подтвержденных расширений. Вы также можете использовать вложенный цикл, как в функции `checkValidationLayerSupport`. Разница в производительности не существенна.

Теперь запустим программу и убедимся, что наша видеокарта годится для создания swap chain. Обратите внимание, что наличие очереди отображения уже подразумевает поддержку расширения swap chain. Тем не менее, лучше убедиться в этом явно.

## Подключение расширений

Чтобы использовать swap chain, сначала нужно включить расширение `VK_KHR_swapchain`. Для этого немного изменим заполнение `VkDeviceCreateInfo` при создании логического устройства:

```cpp
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

## Запрос информации о поддержке swap chain

Одной проверки, позволяющей узнать, доступна ли swap chain, недостаточно. Создание swap chain включает в себя гораздо больше настроек, поэтому нам нужно запросить больше информации.

Всего необходимо выполнить проверку 3-х типов свойств:

- Базовые требования \(capabilities\) surface, такие как мин\/макс число изображений в swap chain, мин\/макс ширина и высота изображений
- Формат surface \(формат пикселей, цветовое пространство\)
- Доступные режимы работы


Для работы с этими данными мы будем использовать структуру:

```cpp
struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

А теперь создадим функцию `querySwapChainSupport`, которая заполняет эту структуру.

```cpp
SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device) {
    SwapChainSupportDetails details;

    return details;
}
```

Начнем с surface capabilities. Их легко запросить, и они возвращаются в структуру `VkSurfaceCapabilitiesKHR`.

```cpp
vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
```

Эта функция принимает созданные ранее [VkPhysicalDevice](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPhysicalDevice.html) и `VkSurfaceKHR`. Каждый раз, когда мы будем запрашивать поддерживаемый функционал, эти два параметра будут первыми, поскольку они являются ключевыми компонентами swap chain.

Следующим шагом будет запрос поддерживаемых форматов surface. Для этого совершим уже знакомый ритуал с двойным вызовом функции:

```cpp
uint32_t formatCount;
vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);

if (formatCount != 0) {
    details.formats.resize(formatCount);
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
}
```

Убедитесь, что вы выделили достаточно места в векторе, чтобы получить все доступные форматы.

Этим же способом запросим поддерживаемые режимы работы с помощью функции `vkGetPhysicalDeviceSurfacePresentModesKHR`:

```cpp
uint32_t presentModeCount;
vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, nullptr);

if (presentModeCount != 0) {
    details.presentModes.resize(presentModeCount);
    vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());
}
```

Когда вся необходимая информация будет в структуре, дополним функцию `isDeviceSuitable`, чтобы проверить, поддерживается ли swap chain. В рамках этого руководства, будем считать, что если есть хотя бы один поддерживаемый формат изображений и один поддерживаемый режим работы для window surface, значит swap chain поддерживается.

```cpp
bool swapChainAdequate = false;
if (extensionsSupported) {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
    swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
}
```

Запрашивать поддержку swap chain нужно только после того, как вы убедитесь, что расширение доступно.

Последняя строка функции меняется на:

```cpp
return indices.isComplete() && extensionsSupported && swapChainAdequate;
```

## Выбор настроек для swap chain

Если `swapChainAdequate` имеет значение `true`, значит swap chain поддерживается. Но у swap chain может быть несколько режимов. Напишем несколько функций, чтобы подобрать подходящие настройки для создания наиболее эффективной swap chain.

Всего выделим 3 типа настроек:

- формат surface (глубина цвета)
- режим работы (условия для смены кадров на экране)
- swap extent (разрешение изображений в swap chain)


Для каждой настройки мы будем искать какое-то "идеальное" значение, а если оно недоступно, мы будем использовать некую логику, чтобы выбрать из того, что есть.

### Формат surface

Добавим функцию для выбора формата:

```cpp
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {

}
```

Позже мы будем передавать член `formats` из структуры `SwapChainSupportDetails` в качестве аргумента.

Каждый элемент `availableFormats` содержит члены `format` и `colorSpace`. Поле `format` определяет количество и типы каналов. Например, `VK_FORMAT_B8G8R8A8_SRGB` обозначает, что у нас есть B, G, R и альфа каналы по 8 бит, всего 32 бита на пиксель. С помощью флага `VK_COLOR_SPACE_SRGB_NONLINEAR_KHR` в поле `colorSpace` указывается, поддерживается ли цветовое пространство SRGB. Обратите внимание, что в ранней версии спецификации этот флаг назывался `VK_COLORSPACE_SRGB_NONLINEAR_KHR`.

В качестве цветового пространства мы будем использовать SRGB. SRGB – это стандарт представления цветов в изображениях, он лучше передает воспринимаемые цвета. Именно поэтому в качестве цветового формата мы также будем использовать один из форматов SRGB — `VK_FORMAT_B8G8R8A8_SRGB`.

Пройдемся по списку и проверим, доступна ли нужная нам комбинация:

```cpp
for (const auto& availableFormat : availableFormats) {
    if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
        return availableFormat;
    }
}
```

Если нет, мы можем отсортировать доступные форматы от более подходящих до менее подходящих, но в большинстве случаев можно просто взять первый из списка.

```cpp
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {
    for (const auto& availableFormat : availableFormats) {
        if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
            return availableFormat;
        }
    }

    return availableFormats[0];
}
```

### Режим работы

Режим работы, пожалуй, самая важная настройка swap chain, поскольку он определяет условия для смены кадров на экране.

Всего в Vulkan доступны четыре режима:

- `VK_PRESENT_MODE_IMMEDIATE_KHR`: изображения, отправленные вашим приложением, немедленно отправляются на экран, что может приводить к артефактам.
- `VK_PRESENT_MODE_FIFO_KHR`: изображения для вывода на экран берутся из начала очереди в момент обновления экрана. В то время, как программа помещает отрендеренные изображения в конец очереди. Если очередь заполнена, программа будет ждать. Это похоже на вертикальную синхронизацию, используемую в современных играх.
- `VK_PRESENT_MODE_FIFO_RELAXED_KHR`: этот режим отличается от предыдущего только в одном случае, когда происходит задержка программы и в момент обновления экрана остается пустая очередь. Тогда изображение передается на экран сразу после его появления без ожидания обновления экрана. Это может привести к видимым артефактам.
- `VK_PRESENT_MODE_MAILBOX_KHR`: это еще один вариант второго режима. Вместо того, чтобы блокировать программу при заполнении очереди, изображения в очереди заменяются новыми. Этот режим подходит для реализации тройной буферизации. С ней вы можете избежать появления артефактов при низком времени ожидания.

Гарантированно доступен только режим `VK_PRESENT_MODE_FIFO_KHR`, поэтому нам снова придется написать функцию для поиска лучшего доступного режима:

```cpp
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    return VK_PRESENT_MODE_FIFO_KHR;
}
```

Лично я считаю, что лучше всего использовать тройную буферизацию. Она позволяет избежать появления артефактов при низком времени ожидания.

Итак, давайте пройдемся по списку, чтобы проверить доступные режимы:

```cpp
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        }
    }

    return VK_PRESENT_MODE_FIFO_KHR;
}
```

### Swap extent

Осталось настроить последнее свойство. Для этого добавим функцию:

```cpp
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {

}
```

Swap extent – это разрешение изображений в swap chain, которое почти всегда совпадает с разрешением окна \(в пикселях\), куда рендерятся изображения. Допустимый диапазон мы получили в структуре `VkSurfaceCapabilitiesKHR`. Vulkan сообщает нам, какое разрешение мы должны выставить, с помощью поля `currentExtent` \(соответствует размеру окна\). Однако некоторые оконные менеджеры допускают использование разных разрешений. Для этого указывается специальное значение ширины и высоты в `currentExtent` — максимальное значение типа `uint32_t`. В таком случае из промежутка между `minImageExtent` и `maxImageExtent` мы выберем разрешение, которое больше всего соответствует разрешению окна. Главное — правильно указать единицы измерения.

В GLFW используется две единицы измерения: пиксели и [экранные координаты](https://www.glfw.org/docs/latest/intro_guide.html#coordinate_systems). Так, разрешение `{WIDTH, HEIGHT}`, которое мы указали при создании окна, измеряется в экранных координатах. Но поскольку Vulkan работает с пикселями, разрешение swap chain тоже должно быть указано в пикселях. Если вы используете дисплей с высоким разрешением \(например, дисплей Retina от Apple\), экранные координаты не соответствуют пикселям: из-за более высокой плотности пикселей разрешение окна в пикселях выше, чем в экранных координатах. Так как Vulkan сам не исправит разрешение swap chain для нас, мы не можем использовать исходное разрешение `{WIDTH, HEIGHT}`. Вместо этого мы должны использовать `glfwGetFramebufferSize`, чтобы запросить разрешение окна в пикселях, прежде чем сопоставлять его с минимальным и максимальным разрешением изображений.

```cpp
#include <cstdint> // Necessary for UINT32_MAX

...

VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != UINT32_MAX) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        actualExtent.width = std::max(capabilities.minImageExtent.width, std::min(capabilities.maxImageExtent.width, actualExtent.width));
        actualExtent.height = std::max(capabilities.minImageExtent.height, std::min(capabilities.maxImageExtent.height, actualExtent.height));

        return actualExtent;
    }
}
```

Функции `max` и `min` здесь используются для ограничения значений `width` и `height` в пределах доступных разрешений. Не забудьте подключить заголовочный файл `<algorithm>` для использования функций.

## Создание swap chain

Теперь у нас есть вся необходимая информация для создания подходящей swap chain.

Создадим функцию `createSwapChain` и вызовем ее из `initVulkan` после создания логического устройства.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
}

void createSwapChain() {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);
}
```

Теперь нужно решить, сколько объектов image должно быть в swap chain. В реализации указывается минимальное количество, необходимое для работы:

```cpp
uint32_t imageCount = swapChainSupport.capabilities.minImageCount;
```

Однако, если использовать только этот минимум, иногда придется ждать, когда драйвер закончит внутренние операции, чтобы получить следующий image. Поэтому лучше запросить хотя бы на один больше указанного минимума:

```cpp
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
```

Важно не превышать максимальное количество. Значение 0 обозначает, что максимум не задан.

```cpp
if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount) {
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

Swap chain – это объект Vulkan, поэтому для его создания требуется заполнить структуру. Начало структуры нам уже знакомо:

```cpp
VkSwapchainCreateInfoKHR createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;
```

Сначала указывается surface, к которой привязан swap chain, далее — информация для создания image объектов:

```cpp
createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

В `imageArrayLayers` указывается число слоев, из которых состоит каждый image. Здесь всегда будет значение `1`, если, конечно, это не стереоизображения. Битовое поле `imageUsage` указывает, для каких операций будут использоваться images, полученные из swap chain. В руководстве мы будем рендерить непосредственно в них, но вы можете сначала рендерить в отдельный image, например, для постобработки. В таком случае используйте значение `VK_IMAGE_USAGE_TRANSFER_DST_BIT`, а для переноса используйте операции перемещения в памяти \(memory operation\).

```cpp
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if (indices.graphicsFamily != indices.presentFamily) {
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0; // Optional
    createInfo.pQueueFamilyIndices = nullptr; // Optional
}
```

Затем нужно указать, как обрабатывать объекты images, которые используются в нескольких семействах очередей. Это актуально для случаев, когда семейство с поддержкой графических операций и семейство с поддержкой отображения — это разные семейства. Мы будем рендерить на image в графической очереди, а затем отправлять их в очередь отображения.

Есть два способа обработки image с доступом из нескольких очередей:

- `VK_SHARING_MODE_EXCLUSIVE`: объект принадлежит одному семейству очередей, и право владения должно быть передано явно перед использованием его в другом семействе очередей. Такой способ обеспечивает самую высокую производительность.
- `VK_SHARING_MODE_CONCURRENT`: объекты могут использоваться в нескольких семействах очередей без явной передачи права владения.

Если у нас несколько очередей, мы будем использовать `VK_SHARING_MODE_CONCURRENT`. Для этого способа требуется заранее указать, между какими семействами очередей будет разделено владение. Это можно сделать с помощью параметров `queueFamilyIndexCount` и `pQueueFamilyIndices`. Если семейство графических очередей и семейство очередей отображения совпадают, что случается чаще, используйте `VK_SHARING_MODE_EXCLUSIVE`.

```cpp
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
```

Можно указать, чтобы к изображениям в swap chain применялось какое-либо преобразование из поддерживаемых \(`supportedTransforms` в `capabilities`\), например, поворот на 90 градусов по часовой стрелке или отражение по горизонтали. Чтобы не применять никаких преобразований, просто оставьте `currentTransform`.

```cpp
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
```

Поле `compositeAlpha` указывает, нужно ли использовать альфа-канал для смешивания с другими окнами в оконной системе. Скорее всего, альфа-канал вам не понадобится, поэтому оставьте `VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`.

```cpp
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
```

Поле `presentMode` говорит само за себя. Если мы выставим `VK_TRUE` в поле `clipped`, значит нас не интересуют скрытые пикселы \(например, если часть нашего окна перекрыта другим окном\). Вы всегда сможете выключить clipping, если вам понадобится прочитать пиксели, а пока оставим clipping включенным.

```cpp
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

Остается последнее поле — `oldSwapChain`. Если swap chain станет недействительной, например, из-за изменения размера окна, ее нужно будет воссоздать с нуля и в поле `oldSwapChain` указать ссылку на старую swap chain. Это сложная тема, которую мы рассмотрим в одной из следующих глав. Пока представим, что у нас будет только одна swap chain.

Добавим член класса для хранения объекта `VkSwapchainKHR`:

```cpp
VkSwapchainKHR swapChain;
```

Теперь надо просто вызвать `vkCreateSwapchainKHR` для создания swap chain:

```cpp
if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS) {
    throw std::runtime_error("failed to create swap chain!");
}
```

В функцию передаются следующие параметры: логическое устройство, информация о swap chain, опциональный кастомный аллокатор и указатель для записи результата. Никаких сюрпризов. Swap chain нужно уничтожить с помощью `vkDestroySwapchainKHR` до уничтожения устройства:

```cpp
void cleanup() {
    vkDestroySwapchainKHR(device, swapChain, nullptr);
    ...
}
```

Теперь запустим программу, чтобы убедиться, что swap chain была создана успешно. Если придет сообщение об ошибке или сообщение типа "Н`е удалось найти vkGetInstanceProcAddress в SteamOverlayVulkanLayer.dll`", зайдите в раздел [FAQ](https://vulkan-tutorial.com/FAQ).

Попробуем удалить строку `createInfo.imageExtent = extent;` с включенными слоями валидации. Один из уровней валидации сразу же обнаружит ошибку и уведомит нас:

![](0.png)

## Получение image из swap chain

Теперь, когда swap chain создана, осталось получить дескрипторы [VkImages](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImage.html). Добавим член класса для хранения дескрипторов:

```cpp
std::vector<VkImage> swapChainImages;
```

Объекты image из swap chain будут уничтожены автоматически после уничтожения самой swap chain, поэтому добавлять код очистки не нужно.

Сразу после вызова `vkCreateSwapchainKHR` добавим код для получения дескрипторов. Помните, что мы указали только минимальное количество изображений в swap chain, это значит, что их может быть и больше. Поэтому сначала запросим реальное количество изображений с помощью функции `vkGetSwapchainImagesKHR`, затем выделим необходимое место в контейнере и снова вызовем `vkGetSwapchainImagesKHR` для получения дескрипторов.

```cpp
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

И последнее — сохраним формат и разрешение изображений swap chain в переменные класса. Они понадобятся нам в дальнейшем.

```cpp
VkSwapchainKHR swapChain;
std::vector<VkImage> swapChainImages;
VkFormat swapChainImageFormat;
VkExtent2D swapChainExtent;

...

swapChainImageFormat = surfaceFormat.format;
swapChainExtent = extent;
```

Теперь у нас есть image для отрисовки и вывода на экран. В следующей главе мы расскажем, как настроить image для использования в качестве render target-ов, и начнем знакомиться с графическим конвейером и командами рисования!

[C++](06_swap_chain_creation.cpp)
