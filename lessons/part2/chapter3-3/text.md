# Vulkan. Руководство разработчика. Проходы рендера (Render passes)

## Проходы рендера

### Подготовка

Прежде чем завершить создание графического конвейера нужно сообщить Vulkan, какие буферы \(attachments\) будут использоваться во время рендеринга. Необходимо указать, сколько будет буферов цвета, буферов глубины и сэмплов для каждого буфера. Также нужно указать, как должно обрабатываться содержимое буферов во время рендеринга. Вся эта информация обернута в объект прохода рендера \(render pass\), для которого мы создадим новую функцию `createRenderPass`. Вызовем эту функцию из `initVulkan` перед `createGraphicsPipeline`.

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
}

...

void createRenderPass() {

}
```


### Настройка буферов \(attachments\)

Мы используем только один цветовой буфер, представленный одним из images в swap chain.

```cpp
void createRenderPass() {
    VkAttachmentDescription colorAttachment{};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
}
```

Формат цветового буфера \(поле `format`\) должен соответствовать формату image из swap chain, и поскольку мы пока не задействуем мультисэмплинг, нам понадобится только 1 сэмпл.

```cpp
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

`loadOp` и `storeOp` указывают, что делать с данными буфера перед рендерингом и после него. Для `loadOp` возможны следующие значения:

- `VK_ATTACHMENT_LOAD_OP_LOAD`: буфер будет содержать те данные, которые были помещены в него до этого прохода \(например, во время предыдущего прохода\)
- `VK_ATTACHMENT_LOAD_OP_CLEAR`: буфер очищается в начале прохода рендера
- `VK_ATTACHMENT_LOAD_OP_DONT_CARE`: содержимое буфера не определено; для нас оно не имеет значения


Мы будем использовать `VK_ATTACHMENT_LOAD_OP_CLEAR`, чтобы заполнить фреймбуфер черным цветом перед отрисовкой нового фрейма.

Для `storeOp` возможны только два значения:

- `VK_ATTACHMENT_STORE_OP_STORE`: содержимое буфера сохраняется в память для дальнейшего использования
- `VK_ATTACHMENT_STORE_OP_DONT_CARE`: после рендеринга буфер больше не используется, и его содержимое не имеет значения

Нам нужно вывести отрендеренный треугольник на экран, поэтому перейдем к операции сохранения:

```cpp
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
```

`loadOp` и `storeOp` применяются к буферам цвета и глубины. Для буфера трафарета используются поля `stencilLoadOp`\/`stencilStoreOp`. Мы не используем буфер трафарета, поэтому результаты загрузки и сохранения нас не интересуют.

```cpp
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

Текстуры и фреймбуферы в Vulkan — это объекты [VkImage](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkImage.html) с определенным форматом пикселей, однако layout пикселей в памяти может меняться в зависимости от того, что вы хотите сделать с image.

Вот некоторые из наиболее распространенных layout-ов:

- `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`: images используются в качестве цветового буфера
- `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`: images используются для показа на экране
- `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`: image принимает данные во время операций копирования


Более подробно мы обсудим эту тему в главе, посвященной текстурированию. Сейчас нам важно, чтобы images были переведены в layouts, подходящие для дальнейших операций.

В `initialLayout` указывается layout, в котором будет image перед началом прохода рендера. В `finalLayout` указывается layout, в который image будет автоматически переведен после завершения прохода рендера. Значение `VK_IMAGE_LAYOUT_UNDEFINED` в поле `initialLayout` обозначает, что нас не интересует предыдущий layout, в котором был image. Использование этого значения не гарантирует сохранение содержимого image, но это и не важно, поскольку мы все равно очистим его. После рендеринга нам нужно вывести наш image на экран, поэтому в поле `finalLayout` укажем `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`.

### Подпроходы \(subpasses\)

Один проход рендера может состоять из множества подпроходов \(subpasses\). Подпроходы — это последовательные операции рендеринга, зависящие от содержимого фреймбуферов в предыдущих проходах. К ним относятся, например, эффекты постобработки, применяемые друг за другом. Если объединить их в один проход рендера, Vulkan сможет перегруппировать операции для лучшего сохранения пропускной способности памяти и большей производительности \(прим. переводчика: видимо, имеется в виду тайловый рендеринг\). Однако мы будем использовать для нашего треугольника только один подпроход.

Каждый подпроход ссылается на один или несколько attachment-ов. Эти отсылки представляют собой структуры [VkAttachmentReference](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkAttachmentReference.html):

```cpp
VkAttachmentReference colorAttachmentRef{};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

В поле `attachment` указывается порядковый номер буфера в массиве, на который ссылается подпроход. Наш массив состоит только из одного буфера [VkAttachmentDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkAttachmentDescription.html), его индекс равен `0`. В поле `layout` мы указываем layout буфера во время подпрохода, ссылающегося на этот буфер. Vulkan автоматически переведет буфер в этот layout, когда начнется подпроход. Мы используем attachment в качестве буфера цвета, и layout `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` обеспечит нам самую высокую производительность.

Подпроход описывается с помощью структуры [VkSubpassDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkSubpassDescription.html):

```cpp
VkSubpassDescription subpass{};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
```

Мы должны явно указать, что это графический подпроход, поскольку не исключено, что в будущем Vulkan может поддерживать вычислительные подпроходы. После этого укажем ссылку на цветовой буфер:

```cpp
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```

Директива `layout(location = 0) out vec4 outColor` ссылается именно на порядковый номер буфера в массиве `subpass.pColorAttachments`.

Подпроход может ссылаться на следующие типы буферов:

- `pInputAttachments`: буферы, содержимое которых читается из шейдера
- `pResolveAttachments`: буферы, которые используются для цветовых буферов с мультисэмплингом
- `pDepthStencilAttachment`: буферы глубины и трафарета
- `pPreserveAttachments`: буферы, которые не используются в текущем подпроходе, но данные которых должны быть сохранены

### Проход рендера \(render pass\)

Теперь, когда буфер и подпроход, ссылающийся на этот буфер, описаны, мы можем создать сам проход рендера. Перед `pipelineLayout` создадим новую переменную-член класса для хранения объекта [VkRenderPass](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkRenderPass.html):

```cpp
VkRenderPass renderPass;
VkPipelineLayout pipelineLayout;
```

Теперь создадим объект прохода рендера. Для этого заполним структуру [VkRenderPassCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkRenderPassCreateInfo.html) массивом буферов и подпроходами рендера. Обратите внимание, объекты [VkAttachmentReference](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkAttachmentReference.html) используют индексы из этого массива \(прим. переводчика: видимо, имеется в виду массив `renderPassInfo.pAttachments`\).

```cpp
VkRenderPassCreateInfo renderPassInfo{};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
}
```

Мы будем ссылаться на проход рендера на протяжении всего жизненного цикла программы, поэтому его нужно очистить в самом конце:

```cpp
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ...
}
```

Мы проделали большую работу, и осталось лишь собрать все воедино, чтобы наконец-то создать графический конвейер!

[C++ code](11_render_passes.cpp) \/ [Vertex shader](09_shader_base.vert) \/ [Fragment shader](09_shader_base.frag)


## Заключение

Теперь мы можем объединить все структуры и объекты, чтобы создать графический конвейер!
Давайте вспомним, какие объекты у нас уже есть:

- Шейдеры: шейдерные модули, определяющие функционал программируемых стадий конвейера
- Непрограммируемые стадии: структуры, описывающие работу конвейера на непрограммируемых стадиях, таких как input assembler, растеризатор, вьюпорт и функция смешивания цветов
- Layout конвейера: описание uniform-переменных и push-констант, которые используются конвейером и которые могут обновляться динамически
- Проход рендера \(render pass\): описания буферов \(attachments\), в которые будет производиться рендер


Все эти объекты полностью определяют функционал графического конвейера. С ними можно начать заполнение структуры [VkGraphicsPipelineCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkGraphicsPipelineCreateInfo.html). Сделаем это в конце функции `createGraphicsPipeline`, но перед вызовами [vkDestroyShaderModule](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDestroyShaderModule.html), поскольку шейдерные модули будут использоваться во время создания конвейера.

```cpp
VkGraphicsPipelineCreateInfo pipelineInfo{};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
```

Начнем с указателя на массив структур [VkPipelineShaderStageCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineShaderStageCreateInfo.html).

```cpp
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr; // Optional
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = nullptr; // Optional
```

Затем заполним указатели на все структуры, описывающие непрограммируемые стадии конвейера.

```cpp
pipelineInfo.layout = pipelineLayout;
```

После этого укажем layout конвейера, который является дескриптором Vulkan, а не указателем на структуру.

```cpp
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```

В конце сделаем ссылку на проход \(render pass\) и номер подпрохода \(subpass\), который используется в создаваемом пайплайне. Во время рендера можно использовать и другие объекты прохода, но они должны быть совместимы с нашим `renderPass`. Требования к совместимости вы можете найти [здесь](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#renderpass-compatibility), однако в руководстве мы будем использовать только один проход.

```cpp
pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // Optional
pipelineInfo.basePipelineIndex = -1; // Optional
```

Остались два параметра — `basePipelineHandle` и `basePipelineIndex`. Vulkan позволяет создать производный графический конвейер из существующего конвейера. Суть в том, что создание производного конвейера не требует больших затрат, поскольку большинство функций берется из родительского конвейера. Также переключение между дочерними конвейерами одного родителя осуществляется намного быстрее. В поле `basePipelineHandle` вы можете указать дескриптор существующего конвейера, либо сделать отсылку к другому конвейеру, который будет создан по индексу, в поле `basePipelineIndex`. У нас только один конвейер, поэтому укажем `VK_NULL_HANDLE` и невалидный порядковый номер. Эти значения используются только в том случае, если в [VkGraphicsPipelineCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkGraphicsPipelineCreateInfo.html) в поле `flags` указано `VK_PIPELINE_CREATE_DERIVATIVE_BIT`.

Прежде чем завершить создание конвейера создадим член класса для хранения объекта VkPipeline:

```cpp
VkPipeline graphicsPipeline;
```

И наконец создадим графический конвейер:

```cpp
if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create graphics pipeline!");
}
```

Функция [vkCreateGraphicsPipelines](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateGraphicsPipelines.html) содержит больше параметров, чем обычная функция создания объектов в Vulkan. За один вызов она позволяет создать несколько объектов [VkPipeline](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipeline.html) из массива структур [VkGraphicsPipelineCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkGraphicsPipelineCreateInfo.html).

Параметр [VkPipelineCache](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineCache.html) необязательный, поэтому мы передаем `VK_NULL_HANDLE`. Кэш конвейера может использоваться для хранения и повторного использования данных, связанных с созданием конвейера. Он совместно используется множеством вызовов [vkCreateGraphicsPipelines](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateGraphicsPipelines.html) и даже может быть сохранен на диск для переиспользования при последующих запусках программы. Впоследствии это может значительно ускорить процесс создания конвейера. Мы еще вернемся к этой теме в главе, посвященной кэшу конвейера.

Графический конвейер понадобится для всех операций рисования, поэтому он должен быть уничтожен в самом конце:

```cpp
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

Теперь запустим программу, чтобы убедиться, что конвейер создан успешно! Совсем скоро мы сможем увидеть результат нашей работы на экране. В следующих главах мы создадим фреймбуферы на основе images из swap chain и подготовим команды рисования.

[C++ code](12_graphics_pipeline_complete.cpp) \/ [Vertex shader](09_shader_base.vert) \/ [Fragment shader](09_shader_base.frag)
