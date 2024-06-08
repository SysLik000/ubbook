# Use-after-move

move-семантика C++11 -- важная и нужная фича, позволяющая писать более производительный код, не делающий лишний копий, аллокаций, деаллокаций, а также явно выражать намерение передачи владения ресурсом из одной функции в другую. Все как в, уже многие годы любимом на StackOverflow, Rust. Но по-другому.

Про move-семантику почти наверняка спрашивают на любом сколько-нибудь серьезном собеседовании. Хороший кандидат как-нибудь на пальцах да объяснит что вот, мол, на примере вектора, один объект из другого что-то там забрать может, а эти `&&` ну вот просто синтаксический костыль, потому что `const&` может принять временный объект, но под `const` потом ничего не поменяешь, а `&` принять временный объект не может, а _by value_ с конструктором копирования проблемы... В общем, так получилось.
В конце концов вы с кандидатом, может быть, напишете, простенький `unique_ptr` чтоб он точно продемонстрировал, как умеет воровать указатели из одного объекта в другой. И в теории этого должно хватать в 99% случаев.

А на практике потом встречается 1% интересного. Об этом интересном и пойдет речь далее.

move-семантика в C++ хоть и достаточно эффективна, но все же не до конца. Ее прилепили сверху как неплохой workaround, но оставили существенную проблему.

Давайте глянем на простенький `unique_ptr`:

```C++
template<class T>
class UniquePtr {
public:
    explicit UniquePtr(T* raw) : _ptr {raw} {}
    UniquePtr() = default;
    ~UniquePtr() {
        if (_ptr != nullptr) {
            delete _ptr;
        }
    }
    UniquePtr(const UniquePtr&) = delete;
    UniquePtr(UniquePtr&& other) noexcept : _ptr { std::exchange(other._ptr, nullptr) } {}
    UniquePtr& operator=(const UniquePtr&) = delete;
    UniquePtr& operator=(UniquePtr&& other) noexcept {
        UniquePtr tmp(std::move(other));
        std::swap(this->_ptr, tmp._ptr);
        return *this;
    } 
private:
    T* _ptr = nullptr;
};

....

UniquePtr<MyType> a = ...;
...
// что-нибудь важное с a
...
UniquePtr<MyType> b = std::move(a);
// а тут ничего не мешает сделать
// a = fun();

```

`std::move`, [как известно, ничего не перемещает](https://medium.com/@dhaneshvb/c-pitfalls-std-move-is-not-moving-anything-c9c073422b83). Это просто преобразование ссылок, чтоб при вызове конструктора или оператора присваивания была выбрана нужная перегрузка с rvalue ссылкой.   
Исходный объект, из которого произвели перемещение, никуда не девается (в отличие от Rust -- там объект после перемещения становится недоступен для использования). У него  когда-нибудь будет вызван деструктор. Потому мы обязаны оставить этот объект в каком-то адекватном для вызова деструктора состоянии -- в нашем `UniquePtr` -- оставить там `nullptr`, как это сделано в move-конструкторе.

Но что же происходит в операторе move-присваивания!

```C++
    UniquePtr& operator=(UniquePtr&& other) noexcept {
        UniquePtr tmp(std::move(other));
        std::swap(this->_ptr, tmp._ptr);
        return *this;
    } 
```
Тут зачем-то используется move(copy)-and-swap... Ну как зачем: мы же, наверное, хотим грохнуть старый объект (`T`, а не указатель), и забрать владение новым.
Или не хотим? Если нет, то почему бы не реализовать оператор перемещения так:
```C++
    UniquePtr& operator=(UniquePtr&& other) noexcept {
        std::swap(this->_ptr, other._ptr);
        return *this;
    } 
```
1. Владение данными передано? Передано
2. Старый объект-указатель в адекватном состоянии для вызова деструктора? Да, не хуже, чем тот, куда присваивали!

Все отлично с точки зрения семантики перемещения C++!

Но такое поведение для `UniquePtr` как минимум неожиданное. Потому в стандартной реализации `std::unique_ptr` все-таки зануляет исходный указатель. 
Тоже верно и для `std::shared_ptr`, `std::weak_ptr`. И это гарантируется стандартом...

И тут скрывается главная ловушка: если пустое _moved-out_ состояние для умных указателей гарантированно, то про другие объекты из стандартной библиотеки (и не только её) это вообще-то не так! Совсем не так!

## std::vector и другие похожие контейнеры

Поведение move-оператора перемещения для вектора описывается очень хитро и учитывает параметр, о котором вспоминают только те, кто о нем знает и заинтересован в его настройке: аллокатор.

В каждом экземпляре `std::vector` запрятан объект-аллокатор. Это может быть, как по умолчанию (`std::allocator`), пустой объект использующий глобальные `malloc/operator new`, так и что-то более специфичное: например, вы хотите чтоб каждый ваш вектор использовал свой уникальный предвыделенный кусок одного большого буфера, который полностью под вашим контролем.

Стандартная библиотека просит от типа-аллокатора определить свойство `propagate_on_container_move_assignment`, влияющее на то, как будет вести себя move-присваивание. 
Если вы пишете `A = std::move(B)` есть три варианта:

1. `propagate_on_container_move_assignment{} == true` (да, это не константа, а структура как `false_type`/`true_type`):
Вектор `A` деаллоцируется, аллокатор перемещается (опять-таки с помощью move-присваивания, так что тут уж надо позаботиться о каких-то гарантиях) и содержимое забирается целиком из `B`. `B` будет пуст.
2. `propagate_on_container_move_assignment{} == false` и аллокатор в `A` и `B` один и тот же (`A.get_allocator() == B.get_allocator()`):
 `A` деаллоцируется, аллокатор остается на месте. Содержимое забирается из `A` в `B`
3. `propagate_on_container_move_assignment{} == false` и `A.get_allocator() != B.get_allocator()`. Вот тут начинается самое интересное:
    Забрать ни аллокатор, ни данные целиком `A` не может. Единственный вариант -- переносить каждый элемент отдельно. Но опустошать и деаллоцировать `B` необязательно. Достаточно только перенести элементы. И в этом случае можно получить полный вектор, состоящий из moved-out элементов

В реализации вектора в libc++ в третьем случае как раз-таки вектор остается не пуст. В libstdc++ же воткнут вызов `clear()`. 

В этом можно убедиться на [примере](https://godbolt.org/z/jWM3K15Wv)
```C++
template <class T>
struct MyAlloc {
    using value_type = T;
    using size_type = size_t;
    using difference_type = ptrdiff_t;
    using propagate_on_container_move_assignment = std::false_type;

    T* allocate(size_t n) {
        return static_cast<T*>(malloc(n * sizeof(T)));
    }

    void deallocate(T* ptr, size_t n) {
        free(static_cast<void*>(ptr));
    }


   using is_always_equal = std::false_type;
   bool operator == (const MyAlloc&) const {
        return false;
   }
};

int main() {
    using VectorString = std::vector<std::string, MyAlloc<std::string>>;

    {
        VectorString v = {
            "hello", "world", "my"
        };
        VectorString vv =  std::move(v);
        std::cout << v.size() << "\n";
        // выведет 0. Это был move-конструктор
    }

    {
        VectorString v = {
            "hello", "world", "my"
        };
        VectorString vv;
        vv = std::move(v);
        std::cout << v.size() << "\n";
        // выведет 3. Было move-присваивание
        for (auto& x : v) {
            // но каждый элемент был перемещен -- тут пусто
            std::cout << x;
        }
    }
}
```

Обратите внимание: проблема только с move-присваиванием!
Ну а еще это замечательный пример того, как разрыв объявления и инициализации переменной может менять поведение C++ программы!

Кстати, элементами вектора были строки. И последний цикл обращается к moved-out строкам!

## std::string

Moved-out состояние строк также не специфицировано.

На разных ресурсах, посвященных C++, можно найти пример, выдающий неожиданный результат при компиляции старым Clang 3.7 c libc++:
```C++
void g(std::string v) {
  std::cout << v << std::endl;
}
 
void f() {
  std::string s;
  for (unsigned i = 0; i < 10; ++i) {
    s.append(1, static_cast<char>('0' + i));
    g(std::move(s));
  }
}
```
Начиная с C++11 строки в реализации тройки основных компиляторов используют SSO (small string optimization) -- короткая строка хранится не в куче, а в самом объекте-строке (вместо/поверх указателей -- `union`). И ее копирование становится тривиальным. А тривиальные объекты (примитивы, структуры из примитивов) еще и перемещаются тривиально -- простым копированием. С современными версиями GCC и Clang, с libc++, c lidstdc++, строка остается пустой после move. Но полагаться на это всё же не стоит.

## Что же делать?

С moved-out состоянием объектов может быть четыре уровня гарантий

1. **Destructor only**. moved-out объект годится только на то, чтоб быть уничтоженным. И больше не использоваться. Никак. Это базовая гарантия, которую вы должны обеспечить, если уж решили добавлять move-семантику к своему объекту, чтоб весь механизм автовызова деструкторов не отстрелил никому ноги.
2. **Destructor & assignment**. Теперь еще и можно переиспользовать объект, присвоив ему новое значение (а потом уже пользуйтесь нормально). Объект, который можно перемещать, но нельзя потом переприсваивать -- это очень редкое явление. Поэтому обычно эту гарантию объединяют с предыдущей.
3. **Valid, but unspecified**. Можно пользоваться, вызывать методы, не требующие предусловий, но что там внутри -- черт его знает.
4. **Valid, well defined**. Всё и так ясно.

Читайте документацию, в общем, прежде чем переиспользовать незнакомый moved-out объект! А лучше в принципе так не делать. И многие линтеры и статические анализаторы способны выдать предупреждение, если у вас произошло обращение к moved-out объекту в функции, где вы вызвали на нем `std::move`.

А еще, при реализации оператора перемещения, стоит использовать _move_and_swap_ паттерн (как в `UniquePtr` в самом начале) -- так у вас больше шансов без больших усилий оставлять свои объекты в действительно пустом состоянии.

## Полезные ссылки
1. https://wiki.sei.cmu.edu/confluence/display/cplusplus/EXP63-CPP.+Do+not+rely+on+the+value+of+a+moved-from+object
2. https://stackoverflow.com/a/17735913
3. https://en.cppreference.com/w/cpp/memory/allocator
4. https://www.foonathan.net/2016/07/move-safety/
5. https://medium.com/@dhaneshvb/c-pitfalls-std-move-is-not-moving-anything-c9c073422b83