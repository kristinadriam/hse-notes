# 75. SFINAE — основы

## Вопросы

* Определение SFINAE для функций и специализаций шаблонов, исходное применение
* Отличия hard compilation error от SFINAE, работа внутри псевдонимов типов и шаблонных переменных, примеры
  неработающего SFINAE
* SFINAE по возвращаемому типу: без запятой, с оператором `,`, с возвратом `void`
* `enable_if`: требования, реализация, использование для SFINAE в возвращаемом типе
* SFINAE в фиктивных параметрах шаблона
    * В значении по умолчанию, проблемы с переопределением функций
    * В типе
    * Использование `void_t` вместо `enable_if_t` для проверки корректности выражений

---

## Определение SFINAE для функций и специализаций шаблонов, исходное применение

_Substitution Failure Is Not An Error (SFINAE)_ – ошибка подстановки это не ошибка.

**Стандартные шутки:**

1. Segmentation Fault Is Not An Error
2. SFINAE отбивная

SFINAE нужно, когда мы инстанцируем шаблонную функцию / класс: если перегрузка получается инстанцированием шаблона
функции,
но компилятор не смог вывести типы сигнатуры функции / класса, то такая перегрузка не считается ошибкой, а молча
отбрасывается
(даже без предупреждения).

Пусть у нас здесь есть две вот такие функции.

```c++
template<typename T>
void duplicate_element(T &container, typename T::iterator iter) {
    container.insert(iter, *iter);
}

template<typename T>
void duplicate_element(T *array, T *element) {
   assert(array != element);
   *(element - 1) = *element;
}
```

При выборе перегрузки мы должны проверить, есть ли у `int[]` тип `int[]::iterator`, которого не существует.

Но это не ошибка компиляции. В стандарте прописано, что мы просто игнорируем эту перегрузку.

```c++
std::vector a{1, 2, 3};
duplicate_element(a, a.begin() + 1);
assert((a == std::vector{1, 2, 2, 3}));

int b[] = {1, 2, 3};
duplicate_element(b, b + 1);  // Not a compilation error when we "try" the first
// overload even though `int[]::iterator` does not exist. Substitution `T=int[]` failure, not an error.
```

## Отличия hard compilation error от SFINAE, работа внутри псевдонимов типов и шаблонных переменных, примеры неработающего SFINAE


SFINAE ––> ошибка при непосредственной _подстановке_ типа `T` или при использовании `using ... = ...`.

**Пример SFINAE:**

```c++
#include <iostream>

struct BotvaHolder {
    using botva = int;
};

template<typename T> using GetBotva = typename T::botva;

template<typename T>
void foo(T, GetBotva<T>) {  // GetBotva is an alias, substitution failure happens 'here'.
    std::cout << "1\n";
}

template<typename T>
void foo(T, std::nullptr_t) {
    std::cout << "2\n";
}

int main() {
    foo(BotvaHolder(), 10);  // 1
    foo(BotvaHolder(), nullptr);  // 2
    foo(10, nullptr); // 2
    // foo(10, 10);  // CE
}
```

Hard compilation error ––> ошибка, которая раскрывается _вне сигнатуры шаблонной функции_.

**Примеры:**

Чтобы понять, кто такой `GetBotva<T>::type` в первой перегрузке, нужно пройти два
уровня: `foo` – `GetBotva<T>` – `type`. Если `T::botva` не существует, то ошибка компиляции появляется не внутри
шаблонной функции, поэтому это не SFINAE.

```c++
#include <iostream>

struct BotvaHolder {
    using botva = int;
};

template<typename T>
struct GetBotva {
    using type = typename T::botva;  
};

template<typename T>
void foo(T, typename GetBotva<T>::type) {
    std::cout << "1\n";
}

template<typename T>
void foo(T, std::nullptr_t) {
    std::cout << "2\n";
}

int main() {
    foo(BotvaHolder(), 10);  // 1
    foo(BotvaHolder(), nullptr);  // 2
    foo(10, nullptr); // CE, but should be 2
    // foo(10, 10);  // CE
}
```

И с переменными будет (не) работать похожим образом, так как у нас не type alias.

```c++
struct BotvaHolder {
    static constexpr int botva = 10;
};

template<typename T>
constexpr int GetBotva = T::botva;  // hard compilation error
```

Чтобы SFINAE работало, надо писать вот так:

```c++
#include <iostream>

struct BotvaHolder {
    static constexpr int botva = 10;
};

template<int> struct Foo {};

template<typename T>
void foo(T, Foo<T::botva>) {  // SFINAE is enabled
    std::cout << "1\n";
}

template<typename T>
void foo(T, std::nullptr_t) {
    std::cout << "2\n";
}

int main() {
    foo(BotvaHolder(), Foo<10>{});  // 1
    foo(BotvaHolder(), nullptr);  // 2
    foo(10, nullptr); // 2
    // foo(10, Foo{});  // CE
    // foo(BotvaHolder(), Foo<11>{});  // CE
}
```

## SFINAE по возвращаемому типу: без запятой, с оператором `,`, с возвратом `void`


### Без запятой

```c++
struct A { int val = 0; };
struct B { int val = 0; };
struct C { int val = 0; };
struct D { int val = 0; };
```

Хотим, чтобы можно было бы складывать между собой объекты класса A и B и результат печатался на экран. Две опции:

```c++
template<typename T, typename U>
auto operator+(const T &a, const U &b) -> decltype(a.val + b.val) {  // !! Option 1
    return a.val + b.val;
}

template<typename T, typename U>
decltype(std::declval<T>().val2 + std::declval<U>().val2) operator+(const T &a, const U &b) {  // !! Option 2
    return a.val2 + b.val2;
}
```

### С оператором `,`

А теперь хотим, чтобы нам возвращалось значение.

Удивительно, но и решение (почти) то же. Пользуемся оператором `,`, перед которым просто проверяем,
возможно ли (в данном случае) вывести в `std::cout`.

```c++
template<typename T>
auto println(const T &v) -> decltype(std::cout << v.val, void()) {
    std::cout << v.val << "\n";
}
```

### С возвратом `void`

Можно написать вот так:

```c++
template<typename T>
decltype(std::cout << std::declval<T>().val2, void()) println(const T &v) {
    std::cout << v.val2 << "\n";
}
```

## `enable_if`: требования, реализация, использование для SFINAE в возвращаемом типе

Хотим навесить ограничения на то, какие структуры мы можем складывать между собой.

Напишем шаблонную переменную для ограничения на тип.

```c++
template<typename T>
constexpr bool good_for_plus = std::is_same_v<A, T> ||
                               std::is_same_v<B, T> ||
                               std::is_same_v<C, T> ||
                               std::is_same_v<D, T>;
```

А дальше пишем `enable_if`, внутри которого есть поле `type`, только если первый параметр шаблона `== true`.

В C++ есть стандартный `std::enable_if_t`.

```c++
template<bool Cond, typename T>
struct enable_if {};

template<typename T>
struct enable_if<true, T> { using type = T; };

template<bool Cond, typename T>
using enable_if_t = typename enable_if<Cond, T>::type;

// Check out error message in GCC vs Clang

template<typename T, typename U>
enable_if_t<good_for_plus<T> && good_for_plus<U>, int> operator+(const T &a, const U &b) {
// std::enable_if_t<good_for_plus<T> && good_for_plus<U>, int> operator+(const T &a, const U &b) {
    return a.val + b.val;
}
```

## SFINAE в фиктивных параметрах шаблона

### В значении по умолчанию, проблемы с переопределением функций

У этой штуки возникает проблема со значениями по умолчанию, которые не являются частью сигнатуры,
будет redefinition.

```c++
template<typename T, typename U, typename /* Dummy */ = std::enable_if_t<good_for_plus<T> && good_for_plus<U>>>
int operator+(const T &a, const U &b) {
    return a.val + b.val;
}

// Compilation error: redefinition. Default values is not a part of the signature.
template<typename T, typename U, typename /* Dummy */ = std::enable_if_t<good_for_plus<T> && std::is_integral_v<U>>>
int operator+(const T &a, const U &b) {
    return a.val + b;
}
```

**Решение:**

Сделаем разные типы у последнего параметра.

```c++
// We get: `template<typename U, typename U, void* = nullptr>
template<typename T, typename U, std::enable_if_t<good_for_plus<T> && good_for_plus<U>>* /* ptr */ = nullptr>
int operator+(const T &a, const U &b) {
return a.val + b.val;
}

template<typename T, typename U, std::enable_if_t<good_for_plus<T> && std::is_integral_v<U>>* /* ptr */ = nullptr>
int operator+(const T &a, const U &b) {
return a.val + b;
}
```

### В типе

Хотим сделать перегрузки в шаблонном классе.

В первом случае нет SFINAE, так как это не шаблонный метод.

Во втором случае нет SFINAE, так как шаблонный тип должен быть взять не из класса или откуда-то свыше,
а прямо приписан к самому методу.

В третьем все ок.

```c++
template<typename T>
struct MyClass {
#if 0
    // Checking signature on class instantiation fails. No SFINAE: not a template
    std::enable_if_t<std::is_same_v<T, int>, int> foo_bad_1() {
        return 1;
    }
    int foo_bad_1() const {
        return 2;
    }
#endif

#if 0
    // Checking signature on class instantiation fails. No SFINAE: the error does not depend on template parameter
    template<typename = void>
    std::enable_if_t<std::is_same_v<T, int>, int> foo_bad_2() {
        return 1;
    }
    int foo_bad_2() const {
        return 2;
    }
#endif

    template<typename U = T>  // OK
    std::enable_if_t<std::is_same_v<U, int>, int> foo() {
        static_assert(std::is_same_v<T, int>);
        return 10;
    }
    int foo() const {
        return 20;
    }
};
```

Работает :)

```c++
{
    MyClass<int> a;
    std::cout << a.foo() << "\n";  // 10

    const auto &b = a;
    std::cout << b.foo() << "\n";  // 20
}
{
    MyClass<double> a;
    std::cout << a.foo() << "\n";  // 20

    const auto &b = a;
    std::cout << b.foo() << "\n";  // 20
}
```

### Использование `void_t` вместо `enable_if_t` для проверки корректности выражений

Принимает произвольное количество аргументов произвольных типов и возвращает `void`. Можно вместо `enable_if_t`,
это подчеркивает, что все, что мы передаем, используется чисто для мета-программирования.

```c++
using void_t = void;

// Similar to std::void_t

template<typename T, typename U, void_t<decltype(std::declval<T>().val + std::declval<U>().val)>* /* ptr */ = nullptr>
int operator+(const T &a, const U &b) {
    return a.val + b.val;
}
```