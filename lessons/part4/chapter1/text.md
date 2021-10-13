# Vulkan. Руководство разработчика. Layout дескрипторов и буфер

## Вступление

Мы можем передавать произвольные атрибуты вершинному шейдеру, но как быть с глобальными переменными? С этой главы мы переходим к трехмерной графике, и для этого нам потребуются матричные преобразования. Мы будем использовать 3 матрицы: model — матрица преобразования из локальных координат в мировые, view — для преобразования из мировых координат во view пространство, и proj — проекционная матрица. Мы могли бы записать уже преобразованные координаты в вершинный буфер, но для этого пришлось бы выделять лишнюю память и постоянно обновлять данные.

Чтобы решить эту проблему в Vulkan, нужно использовать дескрипторы ресурсов. Дескрипторы предоставляют шейдерам доступ к таким ресурсам, как буферы и images. Мы создадим буфер, содержащий матрицы преобразования, и предоставим вершинному шейдеру доступ к ним через дескриптор. Использование дескрипторов состоит из трех этапов:

 - Указание layout-а дескрипторов во время создания конвейера
 - Выделение набора дескрипторов из пула дескрипторов
 - Привязка \(bind\) набора дескрипторов во время рендеринга

Есть два вида объектов для работы с дескрипторами ресурсов: layout-ы дескрипторов и сеты дескрипторов \(descriptor set\). Layout определяет типы ресурсов, к которым будет осуществляться доступ из конвейера, точно так же, как render pass определяет типы attachment-ов, с которыми он работает. Сет дескрипторов определяет, какие именно буферы или images нужно привязать к дескрипторам, точно так же, как фреймбуфер определяет, какие именно images использовать для работы с render pass-ом. Именно сеты привязываются к командам рисования, как вершинные буферы или фреймбуферы.

Есть разные типы дескрипторов, но в этой главе мы будем работать с объектами uniform-буферов \(uniform buffer objects или UBO\). В дальнейшем мы рассмотрим и другие типы дескрипторов, но принцип работы у них один.

Допустим, у нас есть данные, которые мы хотим расположить в вершинном шейдере в виде следующей структуры:

```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

Затем мы можем скопировать эти данные в VkBuffer и получить к ним доступ из вершинного шейдера с помощью дескриптора uniform-буфера:

```glsl
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

Мы постоянно будем обновлять матрицы model, view и proj, чтобы прямоугольник, который мы получили в предыдущей главе, вращался в 3D.

## Вершинный шейдер

Изменим вершинный шейдер, чтобы подключить uniform-буфер. Вероятно, вы уже знакомы с MVP преобразованиями. Если нет, см. [ссылку](https://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/).

```glsl
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

Обратите внимание, что порядок объявлений `uniform`, `in` и `out` не имеет значения. Директива `binding` аналогична директиве `location` для атрибутов. Мы укажем на нее в layout-е дескрипторов. Строка с вычислением `gl_Position` изменена, чтобы использовать матрицы для преобразования координат. В отличие от двумерных треугольников, последний компонент координат в пространстве отсечения может быть не равен 1, что приведет к делению при преобразовании в конечные нормализованные экранные координаты устройства. Это используется в перспективной проекции в качестве перспективного деления и необходимо для того, чтобы более близкие объекты выглядели больше, чем объекты вдалеке.

## Layout дескрипторов

Наш следующий шаг — описать используемый в шейдере uniform buffer на стороне C ++.

```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

Мы дублируем описание буфера из шейдера с помощью соответствующих типов GLM. Данные в матрицах бинарно совместимы с тем, как ожидает шейдер, поэтому в дальнейшем мы можем просто скопировать `UniformBufferObject` в [VkBuffer](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkBuffer.html) с помощью `memcpy`.

Нам нужно предоставить подробную информацию о каждом дескрипторе, используемом в шейдерах. Для этого создадим новую функцию, которая называется `createDescriptorSetLayout`. Ее нужно вызвать перед самым созданием конвейера, т. к. она нам там понадобится.

```cpp
void initVulkan() {
    ...
    createDescriptorSetLayout();
    createGraphicsPipeline();
    ...
}

...

void createDescriptorSetLayout() {

}
```

Каждая привязка должна быть описана с помощью структуры [VkDescriptorSetLayoutBinding](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorSetLayoutBinding.html).

```cpp
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding{};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
}
```

Первые два поля определяют `binding`, используемый в шейдере, и тип дескриптора, который представляет собой uniform-буфер. Переменная шейдера может представлять собой массив uniform-буферов, а `descriptorCount` указывает количество значений в массиве.
Это можно использовать, например, для указания трансформаций для каждой кости скелета в скелетной анимации. Наше MVP преобразование находится в единственном uniform-буфере, поэтому в `descriptorCount` мы укажем `1`.

```cpp
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

Также необходимо указать, на каких этапах шейдера будет упоминаться дескриптор. В поле `stageFlags` может быть указана комбинация значений [VkShaderStageFlagBits](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkShaderStageFlagBits.html) или значение `VK_SHADER_STAGE_ALL_GRAPHICS`. В нашем случае мы ссылаемся на дескриптор только из вершинного шейдера.

```cpp
uboLayoutBinding.pImmutableSamplers = nullptr; // Optional
```

Поле `pImmutableSamplers` актуально только для дескрипторов, связанных с чтением из текстур. Их мы рассмотрим позже. Здесь можно оставить значение по умолчанию.

Все привязки дескрипторов объединены в один объект [VkDescriptorSetLayout](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorSetLayout.html). Определим новый член класса перед `pipelineLayout`:

```cpp
VkDescriptorSetLayout descriptorSetLayout;
VkPipelineLayout pipelineLayout;
```

Затем мы можем создать его с помощью [vkCreateDescriptorSetLayout](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateDescriptorSetLayout.html). Эта функция принимает довольно простую структуру [VkDescriptorSetLayoutCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkDescriptorSetLayoutCreateInfo.html), описывающую массив привязок:

```cpp
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 1;
layoutInfo.pBindings = &uboLayoutBinding;

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor set layout!");
}
```

Мы должны указать на только что созданный layout при создании конвейера, чтобы сообщить Vulkan, какие дескрипторы будут использоваться шейдерами. Layout-ы дескрипторов указываются в объекте layout-а конвейера. Изменим [VkPipelineLayoutCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineLayoutCreateInfo.html):

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;
```

Вас может удивить, почему здесь возможно указать несколько layout-ов
дескрипторов, ведь один layout уже содержит все привязки. Мы еще вернемся к этому вопросу в следующей главе, когда будем рассматривать пулы дескрипторов и сеты дескрипторов.

Layout дескриптора должен быть доступен до тех пор, пока мы можем создавать новые графические конвейеры, то есть до завершения программы:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...
}
```

## Uniform-буфер

Пришло время создать буфер для хранения наших матриц. Мы будем копировать новые данные в uniform-буфер в каждом кадре, поэтому нет необходимости создавать промежуточный буфер — он лишь приведет к снижению производительности.

Сразу несколько кадров могут обрабатываться одновременно \(например, один кадр рендерится, другой подготавливается к рендеру\). Мы не должны допускать ситуации, когда данные из буфера считываются видеокартой и обновляются процессором в один и тот же момент, поэтому нам необходимо использовать несколько буферов. Мы можем использовать по одному uniform-буферу на кадр или на image из swap chain. Но поскольку мы ссылаемся на uniform-буферы из буфера команд, то лучше использовать свой uniform-буфер для каждого image.

Для этого добавим новые члены класса для `uniformBuffers` и `uniformBuffersMemory`:

```cpp
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;

std::vector<VkBuffer> uniformBuffers;
std::vector<VkDeviceMemory> uniformBuffersMemory;
```

Аналогичным образом создадим новую функцию `createUniformBuffers`, которая вызывается после `createIndexBuffer` и выделяет буферы:

```cpp
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    createUniformBuffers();
    ...
}

...

void createUniformBuffers() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    uniformBuffers.resize(swapChainImages.size());
    uniformBuffersMemory.resize(swapChainImages.size());

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);
    }
}
```

Мы напишем отдельную функцию, которая будет обновлять буфер на каждом кадре, поэтому [vkMapMemory](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkMapMemory.html) здесь не используется. Uniform-данные будут использоваться для всех вызовов отрисовки, поэтому буфер, содержащий их, должен быть уничтожен только после того, как мы закончим рендеринг. Поскольку он также зависит от количества images в swap chain, которые могут измениться после пересоздания swap chain, мы удалим его в `cleanupSwapChain`:

```cpp
void cleanupSwapChain() {
    ...

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }
}
```

И пересоздадим в `recreateSwapChain`:

```cpp
void recreateSwapChain() {
    ...

    createFramebuffers();
    createUniformBuffers();
    createCommandBuffers();
}
```

## Обновление uniform-данных

Создадим новую функцию `updateUniformBuffer` и вызовем ее из `drawFrame` сразу после того, как узнаем, какой image будет получен из swap chain:

```cpp
void drawFrame() {
    ...

    uint32_t imageIndex;
    VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    updateUniformBuffer(imageIndex);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    ...
}

...

void updateUniformBuffer(uint32_t currentImage) {

}
```

Эта функция будет постоянно генерировать новое преобразование, чтобы заставить геометрию вращаться. Для реализации этой функции нам нужно подключить два новых заголовочных файла:

```cpp
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```

Заголовочный файл `glm/gtc/matrix_transform.hpp` предоставляет функции, которые можно использовать для генерирования model-преобразований, таких как `glm::rotate`, view-преобразований, таких как `glm::lookAt`, и projection-преобразований, таких как `glm::perspective`. Определение `GLM_FORCE_RADIANS` необходимо, чтобы такие функции, как `glm::rotate`, использовали радианы в качестве аргументов во избежание любой возможной путаницы.

Заголовочный файл стандартной библиотеки `chrono` предоставляет функции для точного хронометража. Мы будем использовать их, чтобы обеспечивать вращение геометрии на 90 градусов в секунду независимо от частоты кадров.

```cpp
void updateUniformBuffer(uint32_t currentImage) {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();
}
```

В начале функции `updateUniformBuffer` мы вычисляем время, прошедшее от начала рендеринга, причем используем для этого числа с плавающей запятой.

Теперь вычислим model, view и proj матрицы. Трансформация в мировую систему координат \(model\) – это просто вращение вокруг оси Z с использованием переменной `time`:

```cpp
UniformBufferObject ubo{};
ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

В качестве параметров функция `glm::rotate` принимает существующее преобразование, угол вращения и ось вращения. Конструктор `glm::mat4(1.0f)` возвращает единичную матрицу. Использование угла вращения `time * glm::radians(90.0f)` позволяет осуществлять вращение на 90 градусов в секунду.

```cpp
ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

Для view-преобразования я решил взглянуть на геометрию сверху под углом в 45 градусов. В качестве параметров функция `glm::lookAt` принимает точку, в которой находится камера \(eye position\), точку, на которую смотрит камера \(center position\) и направление «вверх» для камеры \(up axis\).

```cpp
ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);
```

Я решил использовать перспективную проекцию с вертикальным полем зрения в 45 градусов. Остальные параметры — это отношение сторон, ближняя плоскость отсечения и дальняя плоскость отсечения. Важно использовать текущее разрешение swap chain, чтобы рассчитать отношения сторон с учетом новой ширины и высоты после изменения размера окна.

```cpp
ubo.proj[1][1] *= -1;
```

Изначально GLM был разработан для OpenGL, где ось Y в пространстве отсечения имеет противоположное направление. Самый простой способ компенсировать это — инвертировать знак соответствующего поля в матрице проекции. Если этого не сделать, изображение будет рендериться вверх ногами.

Теперь все матрицы вычислены, поэтому мы можем скопировать данные в текущий uniform-буфер.

```cpp
void* data;
vkMapMemory(device, uniformBuffersMemory[currentImage], 0, sizeof(ubo), 0, &data);
    memcpy(data, &ubo, sizeof(ubo));
vkUnmapMemory(device, uniformBuffersMemory[currentImage]);
```

Такой способ использования UBO — не самый эффективный для передачи постоянно меняющихся значений шейдеру. Для передачи небольших буферов данных эффективнее будет использовать push-константы. Мы рассмотрим их в следующих главах.

В следующей главе мы рассмотрим сеты дескрипторов, которые привяжут \(bind\) VkBuffers к дескрипторам uniform-буферов, чтобы шейдер мог получить доступ к данным преобразований.

[Код С++](21_descriptor_layout.cpp) / [Вершинный шейдер](21_shader_ubo.vert) / [Фрагментный шейдер](21_shader_ubo.frag)
