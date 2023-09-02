# 70. Parameter pack — основы

## Вопросы

* Функции с произвольным числом аргументов: синтаксис function parameter pack
* Template parameter pack, variadic template (в том числе специализации — у них может быть несколько паков)
* Значения вместо типов в template parameter pack
* Pack expansion, в том числе параллельный для нескольких паков
* Автовывод типов, автовывод нескольких template parameter pack из аргументов функции
* Fold expression (бинарный и унарный)
* Эмуляция цикла по элементам parameter pack при помощи лямбда-функции, fold expression и `operator,`

---

## Функции с произвольным числом аргументов: синтаксис function parameter pack

**Задача:** есть функция `print`, который мы хотим передать произвольное количество аргументов, чтобы она вывела всех их
в `cout` через пробел. До C++11 писали несколько перегрузок (где-то, кстати, так до сих пор пишут).

```c++
void print() {
}

template<typename T>
void print(const T &v) {
    std::cout << v;
}

template<typename T1, typename T2>
void print(const T1 &v1, const T2 &v2) {
    std::cout << v1 << " " << v2;
}

template<typename T1, typename T2, typename T3>
void print(const T1 &v1, const T2 &v2, const T3 &v3) {
    std::cout << v1 << " " << v2 << " " << v3;
}
```

В С++11 появилась возможность писать обобщенный код и принимать произвольное количество элементов.

Напишем функцию, печатающее количество переданных параметров. Заведем variadic template – любой template с
template parameter pack.

_template parameter pack_: `...Ts` – либо 0, либо 1, либо сколько угодно типов.

_function parameter pack_: `...vs` – произвольное количество аргументов типа из template parameter pack Ts.

**Для запоминания:** `...vs` – (точки слева) – объявление нового, `vs...` – (точки справа) – раскрытие старого.

```c++
template<typename /* template parameter pack */ ...Ts>
// Mnemonics: ... on the left of declared pack.
void println(const Ts &...vs) {  // "function parameter pack"
    // Mnemonics: ... on the right of the used pack.
    // No way to "index" an element because they may have different types.
    // No simple way to force all elements to have the same type.
    print(vs...);  // pack expansion
    std::cout << " - " << sizeof...(vs) << " element(s)\n";
}
```

При вызове можно не указывать типы, а можно и указывать.

```c++
println(10, 20, 'x');
println(10);
println<int, double>(10, 20);  // explicit template arguments
```

## Template parameter pack, variadic template (в том числе специализации — у них может быть несколько паков)

Очень похоже на функции. Не можем завести в классе сразу много полей, но можем завести `tuple`.

```c++
// "Variadic template" is a template which has a "template parameter pack"
template<typename A, int N, typename ...Ts>
struct Foo {
    // Ts... xs;  // Does not work, use std::tuple instead:
    std::tuple<Ts...> xs;
};
```

Теперь можем заводить классы. Компилятор строже, чем в функциях: нельзя сделать parameter pack не последним аргументом.

```c++
Foo<int, 10> f1;                                        // sizeof... = 0
Foo<int, 10, int, int> f2;                              // sizeof... = 2
Foo<int, 10, int, Foo<int, 10, char>, std::string> f3;  // sizeof... = 3

// error: parameter pack 'Ts' must be at the end of the template parameter list
// template<typename... Ts, typename T> struct Bar {};
```

Кстати, `tuple` примерно так и устроен.

Можно сделать структуру от двух `tuple`. И тогда можно использоваться несколько parameter pack. Это стандартный способ
использования.

```c++
template<typename, typename>
struct Foo {
};

template<typename ...As, typename ...Bs>  // Ok, multiple parameter packs.
struct Foo<std::tuple<As...>, std::tuple<Bs...>> {  // Similar to template type deduction.
};
```


## Значения вместо типов в template parameter pack

## Pack expansion, в том числе параллельный для нескольких паков

Использование такой штуки весьма ограничено, например, нельзя получить элемент по номеру (в языке такой конструкции
нет).

Все, что можно сделать – передавать дальше целиком. При этом нельзя даже требовать, чтобы все элементы были одного типа.

Раскрытие pack:

```c++
print(vs...);  // pack expansion
```

Можно получить количество элементов в pack:

```c++
sizeof...(vs)
```

На самом деле, в С и С++ есть _variadic functions_, который работают так: передаем в функцию какое-то количество
аргументов
(синтаксис: `void simple_printf(const char* fmt, ...)`), они как-то неявно конвертируются в `int` / `double` / ..., а затем с
помощью макросов и прочего мы можем с ними работать
(но нельзя вообще ожидать, аргументы каких типов нам прилетят). В С++ не рекомендуется использовать.

Раскрывать можно хитрее: можно раскрыть дважды (`239, vs..., 17, vs...`), можно применить что-то к
выражению (`(vs + 10)...` – паттерн), можно раскрыть сразу несколько
паков (`(vs + vs)...`), если они имеют одинаковую длину (иначе ошибка компиляции).

```c++
void print(int a, int b, int c, int d) {
    std::cout << a << " " << b << " " << c << " " << d << "\n";
}

template<typename ...Ts>
void print_twice(const Ts &...vs) {
    print(239, vs..., 17, vs...);
}

template<typename ...Ts>
void print_plus_10(const Ts &...vs) {
    print((vs + 10)...);
}

template<typename ...Ts>
void print_double(const Ts &...vs) {
    print((vs + vs)...);  // expanded in parallel, should have the same length.
}

int main() {
    print_twice(1);
    print_plus_10(1, 2, 3, 4);
    print_double(1, 2, 3, 4);
}
```

Можно раскрывать pack в куче мест. Например, можно передать в функцию, в параметры конструктора (`()`, `{}`),
можно создать массив (`= {}`), если все аргументы одного типа.

```c++
template<typename ...Ts>
void foo(Ts...) {}

template<typename ...Ts>
void bar(Ts ...vs) {
    foo(vs...);  // function call
    // Different order of evaluation:
    [[maybe_unused]] std::tuple t1(vs...);  // direct initialization
    [[maybe_unused]] std::tuple t2{vs...};  // direct list initialization
    [[maybe_unused]] int data[] = {vs...};  // array initialization
    ...
```

Можно использовать в захвате лямбд (по значению/по ссылке), но
с помощью `std::move` нельзя.

```c++
    ...
[x, vs...]() {
};  // works with captures
[x, &vs...]() {};  // works with captures
// [x, vs = std::move(vs)...]() {};  // no simple way to "move" parameter pack
[x, vs = std::make_tuple(std::move(vs)...)]() {};  // only via std::tuple (it moves/copies arguments inside a tuple), may need std::apply inside.
}
```

## Автовывод типов, автовывод нескольких template parameter pack из аргументов функции



## Fold expression (бинарный и унарный)

Появилось в C++17. Синтаксис: `(vs + ...)` (вместо `+` может быть любой унарный оператор) – _унарная правая свертка_.
Аналогично есть _унарная левая свертка_. Работает с любым типом, у которого есть `+` (в данном контексте).

```c++
// Fold expression should always be inside its own parenthesis.

template<typename ...Ts>
auto sum1(const Ts &...vs) {
   return (vs + ...);  // unary right fold: v0 + (v1 + (v2 + v3))
    // return (... + vs);  // unary left fold: ((v[n-4] + v[n-3]) + v[n-2]) + v[n-1]
}
```

Бинарная правая свертка:

```c++
template<typename ...Ts>
auto sum2(const Ts &...vs) {
    return (vs + ... + 0);  // binary right fold: v0 + (v1 + (v2 + 0))
}
```

Можно даже так:

```c++
template<typename ...Ts>
auto sum_twice(const Ts &...vs) {
    return ((2 * vs) + ... + 0);  // patterns are allowed
}
```

Это тоже забавная бинарная левая свертка:

```c++
template<typename ...Ts>
void print(const Ts &...vs) {
    // binary left fold: (std::cout << v0) << v1
    (std::cout << ... << vs) << "\n";
    // (std::cout << vs << ...) << "\n";  // incorrect
    // (0 + vs + ...)  // incorrect too
    // std::cout << ... << vs << "\n";  // incorrect fold expression
    // std::cout << ... << (vs << "\n");  // `v0 << "\n"` is not a valid expression
}
```

**Допустимые вызовы:**

```c++
int main() {
    // std::cout << sum1() << "\n";  // compilation error
    std::cout << sum1(1, 2, 3) << "\n";
    std::cout << sum1("hello", " ", std::string("world")) << "\n";

    std::cout << sum2() << "\n";
    std::cout << sum2(1, 2, 3) << "\n";

    std::cout << sum_twice(1, 2, 3) << "\n";

    print(1, "hello");
}
```

## Эмуляция цикла по элементам parameter pack при помощи лямбда-функции, fold expression и `operator,`

Хотим что-то вроде цикла. Не можем писать цикл, так как аргументы разных типов, но можно завести вспомогательную
функцию, которая будет по очереди обрабатывать все элементы.

`(f(vs), ...)` – применение функции `f` к каждому элементу pack, некий лайфхак с C++17.

```c++
template<typename ...Ts>
void println(const Ts &...vs) {
    bool first = true;
    auto f = [&](const auto &v) {  // In C++11: use a templated class.
        if (first) {
            first = false;
        } else {
            std::cout << " ";
        }
        std::cout << v;
    };
    (f(vs), ...);  // C++17 lifehack.
    // (f(12), (f("hello"), f("world")));

    // int dummy[] = { f(vs)... };  // C++14 lifehack, make sure `f` returns 0.
    // int dummy[] = { f(12), f("hello"), f("world") };
    std::cout << "\n";
}

int main() {
    println(12, "hello", "world");
}
```