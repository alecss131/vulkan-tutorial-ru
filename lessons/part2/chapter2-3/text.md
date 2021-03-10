# Vulkan. Руководство разработчика. Image views

Для использования [VkImage](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImage.html) мы должны создать объект [VkImageView](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImageView.html) в графическом конвейере. Image view — это буквально взгляд в image. Он описывает, как интерпретировать image и какая часть image будет использоваться.

В этой главе мы напишем функцию `createImageViews`, которая создаст базовый image view для каждого image в swap chain, чтобы в дальнейшем использовать их в качестве буфера цвета \(color target\).

Прежде всего добавим член класса для хранения image views:

```cpp
std::vector<VkImageView> swapChainImageViews;
```

Создадим функцию `createImageView` и вызовем ее сразу после создания swap chain.

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

Первое, что мы сделаем — выделим необходимое место в контейнере, чтобы вместить все image views.

```cpp
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```

Затем создадим цикл, который обходит все image из swap chain.

```cpp
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

Параметры для создания image view передаются в структуру [VkImageViewCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImageViewCreateInfo.html). Первые несколько параметров не вызывают сложностей.

```cpp
VkImageViewCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

Поля `viewType` и `format` указывают, как нужно интерпретировать данные image. Параметр `viewType` позволяет использовать изображения как 1D, 2D, 3D текстуры или cube maps.

```cpp
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

Поле `components` позволяет переключать цветовые каналы между собой. Например, мы можем считывать все цветовые каналы только из `r` компоненты, получив тем самым монохромную картинку. Или, например, назначить `1` или `0` как константу для альфа канала. Здесь же мы будем использовать дефолтные настройки.

```cpp
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

Поле `subresourceRange` описывает, какая часть image будет использоваться. Наши images состоят только из 1 слоя без уровней детализации и будут использоваться в качестве буфера цвета.

```cpp
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

Если вы работаете со стереоизображениями, вам нужно создать swap chain с несколькими слоями. После чего для каждого image создайте несколько image views с отдельным изображением для каждого глаза.

Для создания image view осталось вызвать функцию [vkCreateImageView](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateImageView.html):

```cpp
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```

В отличие от объектов `VkImage`, image views были созданы нами, поэтому нужно описать аналогичный цикл, чтобы уничтожить их перед закрытием программы:

```cpp
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```

Нам достаточно image view, чтобы использовать image в качестве текстуры, однако чтобы использовать image в качестве render target, нужно создать фреймбуфер. Но сначала настроим графический конвейер.

[C++](07_image_views.cpp)
