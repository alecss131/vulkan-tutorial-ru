# Vulkan. Руководство разработчика. Непрограммируемые стадии конвейера

Ранние графические API-интерфейсы использовали состояние по умолчанию для большинства стадий графического конвейера. В Vulkan же все состояния должны описываться явно, начиная с размера вьюпорта и заканчивая функцией смешивания цветов. В этой главе мы выполним настройку непрограммируемых стадий конвейера.

## Входные данные вершин

Структура [VkPipelineVertexInputStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineVertexInputStateCreateInfo.html) описывает формат данных вершин, которые передаются в вершинный шейдер. Есть два типа описаний:

- Описание атрибутов: тип данных, передаваемый в вершинный шейдер, привязка к буферу данных и смещение в нем
- Привязка \(binding\): расстояние между элементами данных и то, каким образом связаны данные и выводимая геометрия \(повершинная привязка или per-instance\) \(см. [Geometry instancing](https://en.wikipedia.org/wiki/Geometry_instancing)\)


Поскольку данные вершин мы жестко прописали в вершинном шейдере, укажем, что данных для загрузки нет. Для этого заполним структуру `VkPipelineVertexInputStateCreateInfo`. Мы вернемся к этому вопросу позже, в главе, посвященной вершинным буферам.

```cpp
VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // Optional
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // Optional
```

Члены `pVertexBindingDescriptions` и `pVertexAttributeDescriptions` указывают на массив структур, которые описывают вышеупомянутые данные для загрузки атрибутов вершин. Добавьте эту структуру в функцию `createGraphicsPipeline` сразу после `shaderStages`.

## Input assembler

Структура [VkPipelineInputAssemblyStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineInputAssemblyStateCreateInfo.html) описывает 2 вещи: какая геометрия образуется из вершин и разрешен ли рестарт геометрии для таких геометрий, как line strip и triangle strip. Геометрия указывается в поле topology и может иметь следующие значения:

- `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`: геометрия отрисовывается в виде отдельных точек, каждая вершина — отдельная точка
- `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`: геометрия отрисовывается в виде набора отрезков, каждая пара вершин образует отдельный отрезок
- `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`: геометрия отрисовывается в виде непрерывной ломаной, каждая последующая вершина добавляет к ломаной один отрезок
- `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`: геометрия отрисовывается как набор треугольников, причем каждые 3 вершины образуют независимый треугольник
- `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP`: геометрия отрисовывается как набор связанных треугольников, причем две последние вершины предыдущего треугольника используются в качестве двух первых вершин для следующего треугольника

Обычно вершины загружаются последовательно в том порядке, в котором вы их расположите в вершинном буфере. Однако с помощью индексного буфера вы можете изменить порядок загрузки. Это позволяет выполнить оптимизацию, например, повторно использовать вершины. Если в поле `primitiveRestartEnable` задать значение `VK_TRUE`, можно прервать отрезки и треугольники с топологией `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP` и `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP` и начать рисовать новые примитивы, используя специальный индекс `0xFFFF` или `0xFFFFFFFF`.

В руководстве мы будем рисовать отдельные треугольники, поэтому будем использовать следующую структуру:

```cpp
VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## Вьюпорт и scissors

Вьюпорт описывает область фреймбуфера, в которую рендерятся выходные данные. Почти всегда для вьюпорта задаются координаты от `(0, 0)` до `(width, height)`.

```cpp
VkViewport viewport{};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

Помните, что размер swap chain и images может отличаться от значений `WIDTH` и `HEIGHT` окна. Позже images из swap chain будут использоваться в качестве фреймбуферов, поэтому мы должны использовать именно их размер.

`minDepth` и `maxDepth` определяют диапазон значений глубины для фреймбуфера. Эти значения должны находиться в диапазоне `[0,0f, 1,0f]`, при этом `minDepth` может быть больше `maxDepth`. Используйте стандартные значения — `0.0f` и `1.0f`, если не собираетесь делать ничего необычного.

Если вьюпорт определяет, как изображение будет растянуто во фрэймбуфере, то scissor определяет, какие пиксели будут сохранены. Все пикселы за пределами прямоугольника отсечения \(scissor rectangle\) будут отброшены во время растеризации. Прямоугольник отсечения используется для обрезки изображения, а не для его трансформации. Разница показана на картинках ниже. Обратите внимание, прямоугольник отсечения слева — лишь один из многих возможных вариантов для получения подобного изображения, главное, чтобы его размер был больше размера вьюпорта.

![](0.png)

В этом руководстве мы хотим отрисовать изображение во весь фреймбуфер, поэтому укажем, чтобы прямоугольник отсечения \(scissor rectangle\) полностью перекрывал вьюпорт:

```cpp
VkRect2D scissor{};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

Теперь нужно объединить информацию о вьюпорте и сциссоре, используя структуру [VkPipelineViewportStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineViewportStateCreateInfo.html). На некоторых видеокартах можно использовать одновременно несколько вьюпортов и прямоугольников отсечения, поэтому информация о них передается в виде массива. Для использования сразу нескольких вьюпортов нужно включить соответствующую опцию GPU.

```cpp
VkPipelineViewportStateCreateInfo viewportState{};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

## Растеризатор

Растеризатор преобразует геометрию, полученную из вершинного шейдера, во множество фрагментов. Здесь также выполняется [тест глубины](https://en.wikipedia.org/wiki/Z-buffering), [face culling](https://en.wikipedia.org/wiki/Back-face_culling), scissor тест и настраивается способ заполнения полигонов фрагментами: заполнение всего полигона, либо только ребра полигонов \(каркасный рендеринг\). Все это настраивается в структуре [VkPipelineRasterizationStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineRasterizationStateCreateInfo.html).

```cpp
VkPipelineRasterizationStateCreateInfo rasterizer{};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

Если в поле `depthClampEnable` установить `VK_TRUE`, фрагменты, которые находятся за пределами ближней и дальней плоскости, не отсекаются, а пододвигаются к ним. Это может пригодиться, например, при создании карты теней. Для использования этого параметра нужно включить соответствующую опцию GPU.

```cpp
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

Если для `rasterizerDiscardEnable` задать `VK_TRUE`, стадия растеризации отключается и выходные данные не передаются во фреймбуфер.

```cpp
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

`polygonMode` определяет, каким образом генерируются фрагменты. Доступны следующие режимы:

- `VK_POLYGON_MODE_FILL`: полигоны полностью заполняются фрагментами
- `VK_POLYGON_MODE_LINE`: ребра полигонов преобразуются в отрезки
- `VK_POLYGON_MODE_POINT`: вершины полигонов рисуются в виде точек

Для использования этих режимов, за исключением `VK_POLYGON_MODE_FILL`, нужно включить соответствующую опцию GPU.

```cpp
rasterizer.lineWidth = 1.0f;
```

В поле `lineWidth` задается толщина отрезков. Максимальная поддерживаемая ширина отрезка зависит от вашего оборудования, а для отрезков толще `1.0f` требуется включить опцию GPU `wideLines`.

```cpp
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

Параметр `cullMode` определяет тип отсечения \(face culling\). Вы можете совсем отключить отсечение, либо включить отсечение лицевых и\/или нелицевых граней. Переменная frontFace определяет порядок обхода вершин \(по часовой стрелке или против\) для определения лицевых граней.

```cpp
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f; // Optional
rasterizer.depthBiasClamp = 0.0f; // Optional
rasterizer.depthBiasSlopeFactor = 0.0f; // Optional
```

Растеризатор может изменить значения глубины, добавив постоянное значение или сместив глубину в зависимости от наклона фрагмента. Обычно это используется при создании карты теней. Нам это не нужно, поэтому для `depthBiasEnable` установим `VK_FALSE`.

## Мультисэмплинг

Структура [VkPipelineMultisampleStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineMultisampleStateCreateInfo.html) настраивает мультисэмплинг — один из способов сглаживания \([anti-aliasing](https://en.wikipedia.org/wiki/Multisample_anti-aliasing)\). Он работает главным образом на краях, комбинируя цвета разных полигонов, которые растеризуются в одни и те же пиксели. Это позволяет избавиться от наиболее заметных артефактов. Основное преимущество мультисэмплинга в том, что фрагментный шейдер в большинстве случаев выполняется только один раз на пиксель, что гораздо лучше, например, чем рендеринг в большем разрешении с последующим уменьшением размеров. Чтобы использовать мультисэмплинг, необходимо включить соответствующую опцию GPU.

```cpp
VkPipelineMultisampleStateCreateInfo multisampling{};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // Optional
multisampling.pSampleMask = nullptr; // Optional
multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

Пока не будем его включать, мы вернемся к нему одной из следующих статей.

## Тест глубины и тест трафарета

При использовании буфера глубины и\/или трафаретного буфера нужно настроить их с помощью [VkPipelineDepthStencilStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineDepthStencilStateCreateInfo.html). У нас пока нет в этом необходимости, поэтому мы просто передадим nullptr вместо указателя на эту структуру. Мы вернемся к этому в главе, посвященной буферу глубины.

## Смешивание цветов

Цвет, возвращаемый фрагментным шейдером, нужно объединить с цветом, уже находящимся во фреймбуфере. Этот процесс называется смешиванием цветов, и есть два способа его сделать:

- Смешать старое и новое значение, чтобы получить выходной цвет
- Объединить старое и новое значение с помощью побитовой операции

Используется два типа структур для настройки смешивания цветов: структура [VkPipelineColorBlendAttachmentState](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineColorBlendAttachmentState.html) содержит настройки для каждого подключенного фреймбуфера, структура [VkPipelineColorBlendStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineColorBlendStateCreateInfo.html) — глобальные настройки смешивания цветов. В нашем случае используется только один фреймбуфер:

```cpp
VkPipelineColorBlendAttachmentState colorBlendAttachment{};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

Структура `VkPipelineColorBlendAttachmentState` позволяет настроить смешивание цветов первым способом. Все производимые операции лучше всего демонстрирует следующий псевдокод:

```
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

Если для `blendEnable` установлено `VK_FALSE`, цвет из фрагментного шейдера передается без изменений. Если установлено `VK_TRUE`, для вычисления нового цвета используются две операции смешивания. Конечный цвет фильтруется с помощью colorWriteMask для определения, в какие каналы выходного изображения идет запись.

Чаще всего для смешивания цветов используют альфа-смешивание, при котором новый цвет смешивается со старым цветом в зависимости от прозрачности. `finalColor` вычисляется следующим образом:

```
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

Это может быть настроено с помощью следующих параметров:

```cpp
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

Все возможные операции вы можете найти в перечислениях [VkBlendFactor](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkBlendFactor.html) и [VkBlendOp](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkBlendOp.html) в спецификации.

Вторая структура ссылается на массив структур для всех фреймбуферов и позволяет задавать константы смешивания, которые можно использовать в качестве коэффициентов смешивания в приведенных выше вычислениях.

```cpp
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

Если вы хотите использовать второй способ смешивания \(побитовая операция\), установите `VK_TRUE` для `logicOpEnable`. После этого вы сможете указать побитовую операцию в поле `logicOp`. Обратите внимание, что первый способ автоматически становится недоступным, как если бы в каждом подключенном фреймбуфере для `blendEnable` было установлено `VK_FALSE`! Обратите внимание, `colorWriteMask` используется и для побитовых операций, чтобы определить, содержимое каких каналов будут изменено. Вы можете отключить оба режима, как это сделали мы, в этом случае цвета фрагментов будут записаны во фреймбуфер без изменений.

## Динамическое состояние

Некоторые состояния графического конвейера можно изменять, не создавая конвейер заново, например, размер вьюпорта, ширину отрезков и константы смешивания. Для этого заполните структуру [VkPipelineDynamicStateCreateInfo](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineDynamicStateCreateInfo.html):

```cpp
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_LINE_WIDTH
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = 2;
dynamicState.pDynamicStates = dynamicStates;
```

В результате значения этих настроек не учитываются на этапе создания конвейера, и вам необходимо указывать их прямо во время отрисовки. Мы вернемся к этому в следующих главах. Вы можете использовать `nullptr` вместо указателя на эту структуру, если не хотите использовать динамические состояния.

## Layout конвейера

В шейдерах вы можете использовать `uniform`-переменные — глобальные переменные, которые можно изменять динамически для изменения поведения шейдеров без необходимости пересоздавать их. Обычно они используются для передачи матрицы преобразования в вершинный шейдер или для создания сэмплеров текстуры во фрагментном шейдере.

Эти uniform-переменные необходимо указать во время создания конвейера с помощью объекта [VkPipelineLayout](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkPipelineLayout.html). Несмотря на то, что мы пока не будем использовать эти переменные, нам все равно нужно создать пустой layout конвейера.

Создадим член класса для хранения объекта, поскольку позже мы будем ссылаться на него из других функций:

```cpp
VkPipelineLayout pipelineLayout;
```

Затем создадим объект в функции `createGraphicsPipeline`:

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // Optional
pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optional

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

В структуре также указываются push-константы, которые представляют собой еще один способ передачи динамических переменных шейдерам. С ними мы познакомимся позже. Мы будем пользоваться конвейером на протяжении всего жизненного цикла программы, поэтому его нужно уничтожить в самом конце:

```cpp
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

## Заключение

Это все, что нужно знать о непрограммируемых состояниях! Потребовалось немало усилий, чтобы настроить их с нуля, зато теперь вы знаете практически все, что происходит в графическом конвейере!

Для создания графического конвейера осталось создать последний объект — проход рендера.

[Код C++](10_fixed_functions.cpp)/[Вершинный шейдер](09_shader_base.vert)/[Фрагментный шейдер](09_shader_base.frag)
