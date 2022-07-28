# Proxy объекты и неявные ссылки

Мы в C++ очень любим generic код. Да и не только в C++. Чтоб все было удобно, переиспользуемо и гибко. На то нам шаблоны и даны!

Давайте напишем немного такого generic кода

```C++
template <class T>
auto pop_last(std::vector<T>& v) {
    assert(!v.empty());
    auto last = std::move(v.back());
    v.pop_back();
    return last;
}
```

Вполне разумно завести себе подобную функцию, ведь имеющиеся `pop_back` у многих контейнеров и адаптеров (`stack`, `queue`) до C++20 возвращают `void`. Что очень неудобно — чаще всего мы хотим изъять последний элемент контейнера и что-то с ним сделать, а не просто выкинуть.

Все ли хорошо с этой функцией? Конечно, на пустом векторе будет неопределенное поведение, но мы же написали `assert`, так что дальше все на откуп пользователю. Пусть просто пишет корректный код, а некорректный не пишет... А в остальном вроде все в порядке, да?

Что ж, давайте ею пользоваться!

```C++
    std::vector<bool> v(65, true);
    auto last = pop_last(v);
    std::cout << last;
```

Все хорошо? Ну вроде бы. Ничего не падает. Можно пойти с разными компиляторами [попроверять](https://godbolt.org/z/7ovG8qEhd).
Неужели никакого подвоха?

На самом деле подвох есть. Число 65 выбрано не случайно и, скорее всего (зависит от реализации), в коде неопределенное поведение, которое никак не проявляется потому что так устроены деструкторы тривиальных типов. Но обо всем по порядку.

## Паттерн Proxy и proxy-объекты

Подробно о разных паттернах проектирования мы говорить не будем. Для этого есть отдельные хорошие [книжки](https://ru.wikipedia.org/wiki/Design_Patterns). Но в общих чертах: Proxy (иногда переводят как Заместитель) — объект, который перехватывает обращения к другому объекту с тем же самым (или похожим) интерфейсом, чтобы сделать что-то. Что именно — зависит от конкретной задачи и реализации.

В стандартной библиотеке C++ есть самые разные proxy-объекты (иногда не чистые proxy, а c добавлением функционала):
    
- `std::reference_wrapper`
- `std::in_ptr, std::inout_ptr` в C++23
- `std::osyncstream` в C++20
- арифметические операции над [valarray](https://en.cppreference.com/w/cpp/numeric/valarray) могут возвращать proxy-объекты.
- `std::vector<bool>::reference`

Вот последний нам и нужен.

В стандарте C++98 приняли ужасное решение, казавшееся тогда разумным: сделать специализацию для `std::vector<bool>`. Обычно `sizeof(bool) == sizeof(char)`, но вообще для `bool` достаточно одного бита. Но адресовать память по одному биту 99.99% всех возможных платформ не могут. Давайте, для более эффективной утилизации памяти, в `vector<bool>` будем паковать биты и иметь `CHAR_BIT` (обычно 8) булевых значений на один байт (`char`).

Это вылилось в то, что работать с `std::vector<bool>` нужно совершенно по-особому:

- В нем нельзя взять адрес (указатель) на конкретный элемент
- Соседние элементы налезают друг на друга
- `reference` это не `bool&`
- При доступе к элементам используются похожие на `bool` proxy-объекты (знающие, к какому биту в байте обращаться). А значит, нужно быть аккуратным с автовыводом типов.


`reference` для `vector<bool>` выглядит примерно так

```C++
class reference {
public:
    operator bool() const { return (*concrete_byte_ptr) & (1 << bitno); }
    reference& operator=(bool) {...}
    ...
private:
    uchar8_t* concrete_byte_ptr;
    uchar8_t  bitno;
}
```

В строке

```C++
auto last = std::move(v.back());
```

`auto` [отбрасывает](decltype_auto_and_explicit_types.md) ссылки, да. Но только настоящие C++ ссылки. `T&` и `T&&` превращаются в `T`. `reference` в `bool` тут сам по себе никак не превратится, даже несмотря на наличие неявного `operator bool`!

И что же получается:

```C++
auto pop_last(std::vector<bool>& v) {
    // v.size() == 65
    auto last = std::move(v.back());
    // last это vector<bool>::reference_t; != bool
    v.pop_back();
    // v.size() == 64
    // мы полностью выкинули последний uint8/uint32/uint64 (зависит от реализации) из вектора.
    // last продолжает ссылаться на выброшенный элемент.
    // если vector<bool> при выбрасывании этого элемента вызвал (псевдо)деструктор,
    // то далее при обращении через last к этому элементу мы нарушаем объектную
    // модель C++, получая доступ к уничтоженному объекту -> UB.
    return last;
}
```

Но мы этого не почувствовали и не увидели при запусках, поскольку:
- `pop_back` не реаллоцирует внутренний буфер вектора
- `~bool` ничего не делает.

Если же мы получим элемент из `pop_last()`, сохраним его, а потом сделаем с вектором еще что-то, что приведет к реаллокации буфера,
UB начнёт проявляться.

## Больше неожиданностей!

```C++
int main() {
    std::vector<bool> v;
    v.push_back(false);
    std::cout << v[0] << " ";
    const auto b = v[0];
    auto c = b;
    c = true;
    std::cout << c << " " << b;
}
```
Данный код [выводит](https://godbolt.org/z/ncxEh39M7) `0 1 1`.
Несмотря на `const`, значение `b` поменялось. Но ведь это же очевидно, да? Ведь `b` это не ссылка, но объект, который ведет себя как ссылка!

Этот код станет еще более внезапным и интересным в C++23: если при переносе новинок в
[cppreference](https://en.cppreference.com/w/cpp/container/vector_bool/reference) не ошиблись, нас ждет перегрузка операции присваивания через `const reference_t`. И можно будет написать даже так:

```C++
int main() {
    std::vector<bool> v;
    v.push_back(false);
    std::cout << v[0] << "\t"; // 0
    const auto b = v[0];
    b = true;
    std::cout << v[0]; // 1
}
```


Такое поведение вполне определено, но может быть неожиданным, если вы пишете какой-нибудь универсальный шаблонный код. Опытные C++ программисты с опаской относятся к явному использованию `vector<bool>`... Но всегда ли они проверяют в шаблонной функции, принимающей `vector<T>`, что `T != bool`?
Да скорее всего почти никогда (если только они не пишут публичную библиотеку).

Ну ладно, понятно с этим вектором всё. В остальных-то случаях все хорошо же?

Конечно!

Давайте возьмем совершенно невинную функцию (спасибо [@sgshulman](https://github.com/sgshulman) за пример)

```C++
template <class T>
T sum(T a, T b)
{   
    T res;
    res = a + b;
    return res;
}
```

И случайно засунем в нее... правильно, какой-нибудь proxy-тип (что же это может быть?)

```C++
    std::vector<bool> v{true, false};
    std::cout << sum(v[0], v[1]) << std::endl;
```

Если нам повезет, мы получим ошибку компиляции — так, например, в реализации msvc у `vector<bool>::reference` нет конструктора по умолчанию. А gcc и clang спокойненько компилируют [нечто](https://godbolt.org/z/x1T5d9veY), падающее с ошибками обращения к памяти: `T res` ссылается на несуществующий вектор.

Также стоит отметить то, как удивительно здесь работают неявные вызовы операторов приведения типов! Ведь на `vector<bool>::reference` не определен `+`. И `return a + b;` не скомпилируется.
Здесь `a` и `b` приводятся к `bool`, затем к `int`, чтобы просуммироваться и потом обратно привестись к `bool`.


## Что еще, и почему нужно быть бдительным

`std::vector<bool>` это просто самый известный пример объекта, порождающего proxy. Вы можете всегда написать свой класс и, если он будет эмулировать поведение тривиальных типов, устроить кому-нибудь (например, коллегам) развлечение.

Стандарт может разрешать возвращать proxy и для других типов и операций. И разработчики стандартной библиотеки могут этим воспользоваться. А могут и нет. В любом случае мы можем случайно или специально написать код, поведение которого будет зависеть от версии библиотеки.
Например, согласно документации, операторы `std::operator*` у `valarray` в 
[`libstdc++` v12.1](https://gcc.gnu.org/onlinedocs/gcc-12.1.0/libstdc++/api/a01589.html#ga51238588f2e0972914177fd7f9a12e15)
и [Visual Studio 2022](https://docs.microsoft.com/en-us/cpp/standard-library/valarray-operators?view=msvc-170#op_star) имеют разные типы возвращаемого значения.

В сторонних библиотеках также могут применяться proxy-объекты. И уж тем более в сторонних библиотеках их использование может меняться от версии к версии.

Например, proxy-объекты используются для операций над матрицами в библиотеке [Eigen](https://eigen.tuxfamily.org).
Результатом произведения двух матриц оказывается не матрица, а специальный proxy-объект [Eigen::Product](https://eigen.tuxfamily.org/dox/classEigen_1_1Product.html),
транспонирование матрицы возвращает [Eigen::Transpose](https://eigen.tuxfamily.org/dox/classEigen_1_1Transpose.html), и многие другие операции порождают proxy-oбъекты.
И если вы на одной версии написали
```C++
    const auto c = op(a, b);
    b = d;
    do_something(c);
```
и все работало, то при обновлении все вполне может сломаться. Вдруг `op` теперь возвращает ленивый proxy, а следующей строкой вы испортили один из аргументов?

## Что делать и как бороться

В C++ — на самом деле никак. Только повышенной внимательностью.
А также тщательно описывать ограничения, накладываемые на типы в шаблонах, — желательно в виде концептов C++20.

Если вы дизайните библиотеку, то подумайте дважды, если хотите добавить в публичный API неявные proxy. Если очень сильно хочется добавить, то подумайте, нельзя ли обойтись без неявных преобразований. Очень многие проблемы, которые мы тут рассмотрели, происходят от неявных преобразований. Может быть лучше сделать API чуть менее удобным и более многословным, но и более безопасным?

Если вы пользуетесь библиотекой, то, может, лучше все-таки указать тип переменной явно? Если вы хотите `bool` — укажите `bool`. Хотите тип элемента вектора? Укажите 
`vector<T>::value_type`. `auto` это очень удобно, но только если вы знаете, что делаете. 

## Полезные ссылки
1. https://stackoverflow.com/questions/17794569/why-isnt-vectorbool-a-stl-container
2. https://www.researchgate.net/publication/220803585_Performance_of_C_bit-vector_implementations
3. https://eigen.tuxfamily.org/dox/TopicWritingEfficientProductExpression.html
4. https://eigen.tuxfamily.org/dox/TopicLazyEvaluation.html