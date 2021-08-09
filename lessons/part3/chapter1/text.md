# Vulkan. Руководство разработчика. Вершинные буферы

## Вступление

В следующих главах мы заменим данные вершин, встроенные в вершинный шейдер, на вершинный буфер в памяти. Начнем с самого простого — создадим буфер видимый для CPU и используем `memcpy`, чтобы напрямую копировать в него данные вершин. Затем рассмотрим, как использовать промежуточный буфер для копирования данных вершин в высокопроизводительную память.

## Вершинный шейдер

Сначала изменим код вершинного шейдера и уберем из него данные вершин. Используем ключевое слово `in`, чтобы шейдер принимал данные из вершинного буфера.

```glsl
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

Переменные `inPosition` и `inColor` — атрибуты вершин. Это свойства, которые указываются для каждой вершины в буфере точно так же, как мы указывали координаты и цвет с помощью массива в шейдере. Не забудьте заново скомпилировать вершинный шейдер!

Точно так же, как это было у `fragColor`, `layout (location = x)` присваивает индекс каждому атрибуту, чтобы в дальнейшем можно было ссылаться на них. Важно знать, что некоторые типы, например 64-битные векторы `dvec3`, используют несколько слотов. Это значит, что следующий индекс после него должен быть как минимум на 2 больше:

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

Дополнительную информацию о layout квалификаторе можно найти в OpenGL [wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL)).

## Данные вершин

Переместим данные вершин из кода шейдера в массив в коде нашей программы. Сначала подключим библиотеку GLM, которая предоставляет векторные и матричные типы. Мы будем использовать эти типы для координат и цвета.

```cpp
#include <glm/glm.hpp>
```

Создадим новую структуру `Vertex` с двумя атрибутами, которые соответствуют входным данным вершинного шейдера:

```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

GLM очень кстати предоставляет типы C ++, которые в точности соответствуют векторным типам, используемым в языке шейдеров.

```cpp
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

Используя структуру `Vertex`, укажем массив данных вершин. Мы используем те же значения координат и цвета, что и раньше, но теперь они объединены в один массив вершин.

## Описание привязки \(binding\)

Следующий шаг — сообщить Вулкану, как передавать такой формат данных в вершинный шейдер после того, как данные оказались в графической памяти. Для этого нужны 2 типа структур.

Первая структура — [VkVertexInputBindingDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkVertexInputBindingDescription.html). Добавим метод в структуру `Vertex`, чтобы заполнить ее нужными данными.

```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};

        return bindingDescription;
    }
};
```

Эта структура определяет, как данные располагаются в памяти. В ней мы указываем расстояние между элементами данных в байтах и то, когда следует переходить к следующей записи данных — после каждой вершины или после каждого экземпляра \(instance\).

```cpp
VkVertexInputBindingDescription bindingDescription{};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

Все наши данные собраны в один массив, поэтому у нас будет только одна привязка. Параметр `binding` указывает номер привязки в массиве. Параметр `stride` указывает расстояние между элементами данных. А параметр `inputRate` может иметь одно из следующих значений:

 - `VK_VERTEX_INPUT_RATE_VERTEX`: переход к следующему элементу данных происходит после каждой вершины
 - `VK_VERTEX_INPUT_RATE_INSTANCE`: переход к следующему элементу данных происходит после каждого экземпляра (instance)

Мы будем использовать `VK_VERTEX_INPUT_RATE_VERTEX`.

## Описание атрибутов

Вторая структура — [VkVertexInputAttributeDescription](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkVertexInputAttributeDescription.html). Она описывает, как интерпретировать входные данные вершин. Добавим еще одну вспомогательную функцию в `Vertex`, чтобы заполнить структуры.

```cpp
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

    return attributeDescriptions;
}
```

Как указывает прототип функции, таких структур будет две. Каждая из структур описывает, как получить отдельный атрибут из куска данных, извлеченного из буфера для вершины. У нас два атрибута — координаты и цвет, поэтому нам нужны две такие структуры.

```cpp
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```

Параметр `binding` сообщает Vulkan, от какой привязки поступают данные для вершины. Параметр `location` ссылается на директиву `location` в вершинном шейдере. В нашем шейдере значение `location = 0` соответствует координатам вершины. Координаты представлены двумя 32-разрядными числами типа float.

Параметр `format` описывает тип данных для атрибута. Может показаться странным, что для format используется такое же перечисление, что и для форматов цвета. Обычно используют такие соответствия типа и формата:

 - `float`: `VK_FORMAT_R32_SFLOAT`
 - `vec2`: `VK_FORMAT_R32G32_SFLOAT`
 - `vec3`: `VK_FORMAT_R32G32B32_SFLOAT`
 - `vec4`: `VK_FORMAT_R32G32B32A32_SFLOAT`

Как вы видите, необходимо использовать формат, в котором количество цветовых каналов совпадает с количеством компонентов в типе данных шейдера. Можно использовать больше каналов, чем количество компонентов в шейдере, но они будут автоматически отброшены. Если количество каналов меньше количества компонентов, компоненты GBA будут использовать значения по умолчанию `(0, 0, 1)`. Тип цвета `(SFLOAT, UINT, SINT)` и разрядность также должны соответствовать типу входных данных шейдера. Ниже представлены примеры:

 - `ivec2`: `VK_FORMAT_R32G32_SINT`, двухкомпонентный вектор 32-битных целых чисел со знаком
 - `uvec4`: `VK_FORMAT_R32G32B32A32_UINT`, 4-компонентный вектор 32-битных целых чисел без знака
 - `double`: `VK_FORMAT_R64_SFLOAT`, число с плавающей запятой двойной точности \(64-битное\)


Параметр `format` неявно определяет размер данных атрибута в байтах, а параметр `offset` указывает смещение данных атрибута от начала считанного для вершины куска. Данные для каждой из вершин загружаются в виде структуры `Vertex`, а атрибут `pos` смещен на `0` байт от начала этой структуры. Можно использовать макрос `offsetof`, чтобы делать эти расчеты автоматически.

```cpp
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```

Аналогично описывается атрибут цвета.

## Входные данные пайплайна

Теперь изменим метод `createGraphicsPipeline`, добавив ссылки на наши структуры, чтобы пайплайн принимал данные вершин в нужном формате. Найдем `vertexInputInfo` и добавим туда ссылки:

```cpp
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

Теперь конвейер может принимать данные вершин и передавать их в наш вершинный шейдер. Но если запустить программу с включенными слоями валидации, вы увидите предупреждение, что буфер вершин, относящийся к привязке, отсутствует. Следующим шагом будет создание вершинного буфера и перемещение в него данных вершин, чтобы GPU мог получить к ним доступ.

[Код C++](17_vertex_input.cpp) / [Вершинный шейдер](17_shader_vertexbuffer.vert) / [Фрагментный шейдер](17_shader_vertexbuffer.frag)
