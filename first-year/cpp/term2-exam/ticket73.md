### 73. Perfect forwarding
* Использование perfect forwarding: `std::forward`, синтаксис forwarding reference
* Теория perfect forwarding: правила вывода для forwarding reference, правила reference collapse
* Когда `T&&` не является forwarding reference, perfect forwarding в методах шаблонных классов
* Perfect forwarding для множества параметров функции
* Оператор `decltype` (два режима работы)
* Синтаксис `decltype(auto)` для объявления переменных и в возвращаемом типе функций и лямбд
    * Невозможность объявить переменную/поле типа `void` (практика `28-220523`, задание `01-memorizer`)
* Отличия `ref`/`cref` и perfect forwarding