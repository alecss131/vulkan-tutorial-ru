# Vulkan. Руководство разработчика. Рисуем треугольник

## Базовый код

### Общая структура

В предыдущей главе мы рассказали, как создать проект для Vulkan, правильно его настроить и протестировать с помощью фрагмента кода. В этой главе мы начнем с самых азов.

Рассмотрим следующий код:

```cpp
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

Сначала мы подключаем заголовочный файл Vulkan из LunarG SDK. Заголовочные файлы `stdexcepts` и `iostream` используются для обработки ошибок и их распространения. Заголовочный файл `cstdlib` предоставляет макросы `EXIT_SUCCESS` и `EXIT_FAILURE`.

Сама программа обернута в класс HelloTriangleApplication, в котором мы будем хранить объекты Vulkan в качестве приватных членов класса. Туда же мы добавим функции для инициализации каждого объекта, вызываемые из функции `initVulkan`. После этого создадим основной цикл для рендеринга кадров. Для этого заполним функцию `mainLoop`, где цикл будет выполняться до закрытия окна. После закрытия окна и выхода из `mainLoop` ресурсы должны быть освобождены. Для этого заполним `cleanup`.

Если во время работы возникнет критическая ошибка, мы будем генерировать исключение `std::runtime_error`, которое будет перехвачено в функции `main`, а описание будет выведено в `std::cerr`. Одной из таких ошибок может быть, например, сообщение о том, что требуемое расширение не поддерживается. Для обработки множества стандартных типов исключений мы перехватываем более общий `std::exception`.

Почти в каждой последующей главе будут добавляться новые функции, вызываемые из `initVulkan`, и новые объекты Vulkan, которые необходимо освободить в `cleanup` по окончании работы программы.

### Управление ресурсами

Если объекты Vulkan больше не нужны, их необходимо уничтожить. C++ позволяет автоматически освобождать ресурсы, используя RAII или умные указатели, предоставляемые заголовочным файлом `<memory>`. Однако, в этом руководстве мы решили явно прописать, когда выделять и освобождать объекты Vulkan. Ведь в этом и заключается особенность работы Vulkan – подробно расписывать каждую операцию во избежание возможных ошибок.

Ознакомившись с руководством, вы сможете реализовать автоматическое управление ресурсами, написав классы C++, которые получают объекты Vulkan в конструкторе и освобождают их в деструкторе. Вы также можете реализовать собственный deleter для `std::unique_ptr` или `std::shared_ptr`, в зависимости от требований. Концепцию RAII рекомендуется использовать для более крупных программ, но не будет лишним узнать о ней больше.

Объекты Vulkan создаются напрямую с помощью функции вида [vkCreateXXX](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateXXX.html), либо выделяются через другой объект с помощью функции вида [vkAllocateXXX](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkAllocateXXX.html). Убедившись, что объект больше нигде не используется, вы должны уничтожить его с помощью [vkDestroyXXX](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDestroyXXX.html) или [vkFreeXXX](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkFreeXXX.html). Параметры для этих функций обычно различаются в зависимости от типа объекта, но есть один общий параметр: `pAllocator`. Это необязательный параметр, который позволяет использовать обратные вызовы (callbacks) для кастомного выделения памяти. В руководстве он нам не понадобится, в качестве аргумента мы будем передавать `nullptr`.

### Интеграция GLFW

Vulkan прекрасно работает без создания окна при использовании внеэкранного рендеринга, но намного лучше, когда результат виден на экране.
Для начала замените строку `#include <vulkan/vulkan.h>` следующим:

```cpp
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

Добавьте функцию `initWindow` и добавьте её вызов из метода run перед другими вызовами. Мы будем использовать `initWindow` для инициализации GLFW и создания окна.

```cpp
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
    void initWindow() {

    }
```

Самым первым вызовом в `initWindow` должна быть функция `glfwInit()`, которая инициализирует библиотеку GLFW. GLFW была изначально разработана для работы с OpenGL. Контекст OpenGL нам не нужен, поэтому укажите, что его создавать не надо, используя следующий вызов:

```cpp
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

Временно отключим возможность изменять размер окна, поскольку обработка этой ситуации требует отдельного рассмотрения:

```cpp
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

Осталось создать окно. Для этого добавьте приватный член `GLFWwindow* window;` и инициализируйте окно с помощью:

```cpp
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

Первые три параметра определяют ширину, высоту и название окна. Четвертый параметр необязательный, он позволяет указать монитор, на котором будет отображаться окно. Последний параметр относится к OpenGL.

Было бы не плохо использовать константы для ширины и высоты окна, т. к. эти значения понадобятся нам в других местах. Добавим следующие строки перед определением класса `HelloTriangleApplication`:

```cpp
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;
```

и заменим вызов создания окна на

```cpp
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

У вас должна получиться следующая функция `initWindow`:

```cpp
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

Опишем главный цикл в методе mainLoop, чтобы поддерживать работу приложения до закрытия окна:

```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

Этот код не должен вызвать вопросов. Он обрабатывает такие события, как, например, нажатие кнопки X до закрытия окна пользователем. Также из этого цикла мы будем вызывать функцию для рендеринга отдельных кадров.

После закрытия окна нам нужно освободить ресурсы и завершить GLFW. Для начала добавим в cleanup следующий код:

```cpp
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

В результате после запуска программы вы увидите окно с именем Vulkan, которое будет отображаться вплоть до закрытия программы. Теперь, когда у нас есть скелет для работы с Vulkan, давайте перейдем к созданию первого объекта Vulkan!

[Код C++](00_base_code.cpp)

## Экземпляр (instance)

### Создание экземпляра

Первое, что вам нужно сделать, — это создать экземпляр для инициализации библиотеки. Экземпляр — это связующее звено между вашей программой и библиотекой Vulkan, и для его создания потребуется предоставить драйверу некоторые данные о вашей программе.

Добавьте метод `createInstance` и вызовите ее из функции `initVulkan`.

```cpp
void initVulkan() {
    createInstance();
}
```

Добавьте член instance в наш класс для хранения дескриптора экземпляра:

```cpp
private:
VkInstance instance;
```

Теперь надо заполнить информацией о программе специальную структуру. Технически, данные указывать необязательно, однако, это позволит драйверу получить полезную информацию для оптимизации работы с вашей программой. Эта структура называется `VkApplicationInfo`:

```cpp
void createInstance() {
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
}
```

Как уже было сказано, многие структуры в Vulkan требуют явного определения типа в члене `sType`. Также эта структура, как и многие другие, содержит элемент `pNext`, который позволяет предоставить сведения для расширений. Мы используем `value initialization`, чтобы заполнить структуру нулями.

Большая часть информации в Vulkan передается через структуры, поэтому вам необходимо заполнить еще одну структуру, чтобы предоставить достаточно информации для создания экземпляра. Следующая структура обязательная, она указывает драйверу, какие глобальные расширения и слои валидации мы хотим использовать. «Глобальные» обозначает, что расширения применяются ко всей программе, а не к конкретному устройству.

```cpp
VkInstanceCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
createInfo.pApplicationInfo = &appInfo;
```

Первые два параметра не вызывают вопросов. Следующие два члена структуры определяют необходимые глобальные расширения. Как вы уже знаете, Vulkan API полностью независим от платформы. Это значит, что вам необходимо расширение для взаимодействия с оконной системой. GLFW имеет удобную встроенную функцию, которая возвращает список необходимых расширений.

```cpp
uint32_t glfwExtensionCount = 0;
const char** glfwExtensions;

glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

createInfo.enabledExtensionCount = glfwExtensionCount;
createInfo.ppEnabledExtensionNames = glfwExtensions;
```

Последние два члена структуры определяют, какие глобальные слои валидации требуется включить. Более подробно мы поговорим о них в следующей главе, поэтому пока оставьте эти значения пустыми.

```cpp
createInfo.enabledLayerCount = 0;
```

Теперь вы сделали всё необходимое для создания экземпляра. Выполните вызов `vkCreateInstance`:

```cpp
VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```

Как правило, параметры функций для создания объектов идут в таком порядке:

- Указатель на структуру с необходимой информацией
- Указатель на кастомный аллокатор
- Указатель на переменную, куда будет записан дескриптор нового объекта

Если все выполнено верно, дескриптор экземпляра сохранится в [instance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstance.html). Почти все функции Vulkan возвращают значение типа [VkResult](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkResult.html), которым может быть либо `VK_SUCCESS`, либо код ошибки. Нам не нужно хранить результат, чтобы убедиться в том, что экземпляр был создан. Используем простую проверку:

```cpp
if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
    throw std::runtime_error("failed to create instance!");
}
```

Теперь запустите программу, чтобы убедиться, что экземпляр был создан успешно.

### Проверка поддерживаемых расширений

Если мы посмотрим в документацию к [Vulkan](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkCreateInstance.html), то можем обнаружить, что одним из возможных кодов ошибки является `VK_ERROR_EXTENSION_NOT_PRESENT`. Мы можем просто указать необходимые расширения и прекратить работу, если они не поддерживаются. Это имеет смысл для основных расширений, например, для интерфейса оконной системы, но что, если мы хотим проверить опциональные возможности?

Чтобы получить список поддерживаемых расширений до создания экземпляра, используйте функцию [vkEnumerateInstanceExtensionProperties](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkEnumerateInstanceExtensionProperties.html). Первый параметр функции необязательный, он позволяет фильтровать расширения по конкретному слою валидации, поэтому мы пока оставим его пустым. Также функция требует указатель на переменную, куда будет записано количество расширений и указатель на область памяти, куда следует писать информацию о них.

Чтобы выделить память для хранения сведений о расширении, сначала необходимо узнать количество расширений. Чтобы запросить количество расширений, оставьте последний параметр пустым:

```cpp
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```

Выделите массив для хранения сведений о расширениях \(не забудьте про `\#include <vector>`\):

```cpp
std::vector<VkExtensionProperties> extensions(extensionCount);
```

Теперь вы можете запросить сведения о расширениях.

```cpp
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

Каждая структура [VkExtensionProperties](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkExtensionProperties.html) содержит имя и версию расширения. Их можно перечислить с помощью простого цикла for \(`\t` здесь — это таб для отступов\):

```cpp
std::cout << "available extensions:\n";

for (const auto& extension : extensions) {
    std::cout << '\t' << extension.extensionName << '\n';
}
```

Вы можете добавить этот код в функцию `createInstance`, чтобы получить дополнительную информацию о поддержке Vulkan. Также можете попробовать создать функцию, которая будет проверять, все ли расширения, возвращаемые функцией `glfwGetRequiredInstanceExtensions`, включены в список поддерживаемых расширений.

### Очистка

[VkInstance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkInstance.html) необходимо уничтожить перед самым закрытием программы. Это можно сделать в cleanup с помощью функции [VkDestroyInstance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDestroyInstance.html):

```cpp
void cleanup() {
    vkDestroyInstance(instance, nullptr);
    glfwDestroyWindow(window);
    glfwTerminate();
}
```

Параметры функции [vkDestroyInstance](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/vkDestroyInstance.html) не требуют объяснений. Как уже было сказано в предыдущей главе, функции выделения и освобождения в Vulkan принимают необязательные указатели на кастомные аллокаторы, которые мы не используем и передаем `nullptr`. Все остальные ресурсы Vulkan нужно очистить до уничтожения экземпляра.

Прежде чем перейти к более сложным действиям нам необходимо настроить слои валидации для удобства отладки.

[Код C++](01_instance_creation.cpp)
