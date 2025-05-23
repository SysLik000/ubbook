# strict aliasing и type punning

Особенно ревностные фанаты C и C++ любят говорить, что эти языки позволяют им все контролировать. Даже малейшие прикосновения к памяти. Нужно просто научиться правильно пользоваться указателями и знать, как устроена память и как работают компьютеры! Байтики они всегда байтики... Вот только злобные разработчики компиляторов понапридумывают своих странных оптимизаций, которые ломают наш прекрасный и 100% правильный код!
А когда не ломают, то ничего не оптимизируют!


При обсуждении правил алиасинга и неопределенного поведения при их нарушении, обычно демонстрируют какую-то странную не очень реалистичную функцию от двух указателей и показывают, какой ужас, ее результат не соответствует ожиданиям при включении оптимизаций. Я, пожалуй, отойду от этой традиции и начну с функции, у которой никаких проблем нет.

```C++
// Не владеющий view над непрерывным массивом,
// например, для итерации по столбцу матрицы
template <class T>
struct stride_view {
    T* data;
    size_t step;
    size_t len;

T& operator[](int idx) {
    return this->data[idx * step]; 
}
};
template <class T>
struct Data {
    stride_view<T> data;
    int counter;
};

void process(Data<uint32_t>* data, int idx) {
    data->data[idx] -= 1;
    if (data->data[idx] == 0) {
        data->counter -= 1;
    }
}
void process(Data<uint64_t>* data, int idx) {
    data->data[idx] -= 1;
    if (data->data[idx] == 0) {
        data->counter -= 1;
    }
}
```

Совершенно нормальные функции. Мы даже исполнять их не будем. Давайте их просто скомпилируем (GCC 14):
`g++ -std=c++26 -O3`

И посмотрим на сгенерированный [код](https://godbolt.org/z/KP8bE3h1v):

```
process(Data<unsigned int>*, int):
        movsx   rsi, esi
        imul    rsi, QWORD PTR [rdi+8]
        mov     rax, QWORD PTR [rdi]
        lea     rdx, [rax+rsi*4]
        sub     DWORD PTR [rdx], 1
        jne     .L1
        sub     DWORD PTR [rdi+24], 1
.L1:
        ret
process(Data<unsigned long>*, int):
        mov     rdx, QWORD PTR [rdi+8]
        movsx   rsi, esi
        mov     rax, QWORD PTR [rdi]
        sal     rsi, 3
        imul    rdx, rsi
        sub     QWORD PTR [rax+rdx], 1
        imul    rsi, QWORD PTR [rdi+8]
        cmp     QWORD PTR [rax+rsi], 0
        jne     .L4
        sub     DWORD PTR [rdi+24], 1
.L4:
        ret
```

И вот же странная картина! Версия с `uint32` явно оптимизирована чуть получше версии с `uint64`! 
```
// можно заметить, что код operator[]  (конкретно умножение на шаг) присутствует дважды
// индекс пересчитывается дважды и обращений к памяти два!
        imul    rdx, rsi
        sub     QWORD PTR [rax+rdx], 1
        imul    rsi, QWORD PTR [rdi+8]
        cmp     QWORD PTR [rax+rsi], 0
```

Ну, мы так и написали же в коде...

Да. Вот только в версии с `uint32` обращение лишь одно. А в коде на C++ же два!
```
        imul    rsi, QWORD PTR [rdi+8]
        mov     rax, QWORD PTR [rdi]
        lea     rdx, [rax+rsi*4]
        sub     DWORD PTR [rdx], 1
```

Обращения к памяти в основном очень дороги. А значит от оптимизации их все будут только в плюсе! Зачем перечитывать повторно значение из памяти в регистр, если нам заранее известно, что оно там не поменялось?!

Но на основе чего можно получить такую информацию, чтоб иметь право выполнить оптимизацию?! Языки C и C++ же низкоуровневые. Полный контроль! Можно делать что угодно с памятью и указателями...

НЕТ. НЕЛЬЗЯ. Поприветствуем правила строгого алиасинга!

Если у вас есть указатели двух **разных** типов `A* a` и `B* b`, то запись значений через указатель `a` **не влияет** на чтение значений через указатель `b`, во всех случаях, кроме нескольких исключений (*алиасинг*):
- `A` и `B` — это *совместимые*, signed/unsigned версии одного и того же типа 
- `A` — это тип, *совместимый* с каким-либо из подобъектов внутри `B`: элемент структуры или объединения, элемент массива
- `A` или `B` — `char`, `unsigned char` или `std::byte`. Этот вариант существует как раз для того чтоб можно было работать с сырыми байтиками. Заметьте, что `signed char` не считается.

В C (не в C++, но компиляторы поддерживают!) допустимо еще:
- `A` и `B` — это *соместимые* структуры/объединения: у них совпадает размер, порядок и имена полей и типы полей *совместимы*. Например `struct Vector { int32_t x; int32_t y; }` и struct `Point { uint32_t x; uint32_t y; }`

Ну, звучит не страшно. Даже логично и правильно. Если у меня есть массив целых чисел, а еще рядом массив чисел с плавающей точкой, то **в корректной** программе, от изменения числа в первом массиве во втором ничего не должно поменяться.

Теперь можно вернуться к примеру и понять, что пошло не так
```C++
struct stride_view<uint32_t> {
    uint32_t* data;
    size_t step;
    size_t len;

uint32_t& operator[](int idx) {
    return this->data[idx * step]; 
}
};
...
void process(Data<uint32_t>* data, int idx) {
    data->data[idx] -= 1; // ссылка uint32_t&. Запись через нее.
    // внутри stride_view нет полей типов int32.
    if (data->data[idx] == 0) {  // Запись выше не влияет на индекс и можно оптимизировать
        data->counter -= 1;
    }
}

struct stride_view<size_t> {
    size_t* data; // !!!
    size_t step;
    size_t len;

T& operator[](int idx) {
    return this->data[idx * step]; 
}
};
void process(Data<size_t>* data, int idx) {
    data->data[idx] -= 1; // ссылка size_t&. Запись через нее.
    // step имеет тип size_t. Ссылка теоретически может алиасить это поле!
    if (data->data[idx] == 0) {  // Запись выше влияет на индекс. Нельзя оптимизировать!
        data->counter -= 1;
    }
}
```

Пытливый читатель, наверное, уже догадался, что случай с указателем `char*` совершенно восхитительно запрещает оптимизацию почти всегда. Ведь он может алиасить что угодно!

Строгий алиасинг это особенность не только С и C++. Rust, например, тоже им следует, но еще более строго, включающая информацию о уникальных и разделяемых ссылках

```Rust
pub struct StrideView<'a, T> {
    pub data: &'a mut [T],
    pub step: usize,
}
impl <'a, T> StrideView<'a, T> {
    unsafe fn get<'b>(&'b mut self, idx: usize) -> &'b mut T {
        self.data.get_unchecked_mut(self.step * idx) // unsafe, нас не интересует код обработки паник
    }
}
pub struct Data<'a, T> {
    pub data: StrideView<'a, T>,
    pub counter: i32,
}
#[no_mangle]
pub unsafe fn process(data: &mut Data<'_, usize>, idx: usize) {
    *data.data.get(idx) -= 1; // ссылка &mut usize 
    // она не может алиасить поле step
    // так как иначе это бы означало, что data содержит ссылку на саму себя
    // что невозможно для обычных ссылок в safe Rust
    if (*data.data.get(idx) == 0) { // можно оптимизировать!
        data.counter -= 1;
    }
}
```
```
// https://godbolt.org/z/dc8Mcajhe
process:
        imul    rsi, qword ptr [rdi + 16]
        mov     rax, qword ptr [rdi]
        dec     qword ptr [rax + 8*rsi]
        je      .LBB0_1
        ret
.LBB0_1:
        dec     dword ptr [rdi + 24]
        ret
```

Ну хорошо, существует такое правило. Оптимизировать позволяет. Отлично. А где неопределенное поведение?

А оно начинается тогда, когда разработчик творит безобразие с преобразованием указателей!

Вот теперь можно и классическую странную функцию показать!

```C++
#include <stdio.h>
int foo(int* x, short* y) {
    *x = 5;
    *y = 10;
    return *x;
}

int main() {
    int f = 0;
    printf("%d\n", foo(&f, (short*)&f));
}
```

В зависимости от уровня оптимизаций, на little endian платформах, [результат будет 5 или 10](https://godbolt.org/z/7YozoYKeb).

Функция, конечно, совершенно надуманная, но фокус в том, что она иллюстрирует простейший пример так называемого **type punning** — обращения с объектом одного типа, как с совершенно другим. 

А вот он уже в реальных программах распространен! Быстрый обратный квадратный корень из Quake III — классический пример
```C
float Q_rsqrt( float number )
{
	int32_t i;
	float x2, y;
	const float threehalfs = 1.5F;

	x2 = number * 0.5F;
	y  = number;
	i  = * (int32_t * ) &y;						// evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

	return y;
}
```
Такой лобовой type punning нарушает предположения строгого алиасинга. Нарушение объявлено неопределенным поведением. И результат можно получить самый странный.

Type punning очень распространен в библиотеках сериализации и десериализации для эффективного разбора сырых байтов. В реализации сетевых протоколов: например, всякие структуры [in_addr](https://learn.microsoft.com/en-us/windows/win32/api/in6addr/ns-in6addr-in6_addr) используют `union` для type punning.
В ядре Linux также можно найти [примеры](https://www.yodaiken.com/2018/06/07/torvalds-on-aliasing/).

### Что же делать?

Вся эта красота в любой момент может сломаться, если она сделана неправильно. Поэтому у GCC и Clang есть флаг `-fno-strict-aliasing`, чтоб отключить оптимизации на основе строгого алиасинга и позволить себе стрелять по ногам чуть более предсказуемо.

Туpe punning с помощью `reinterpret_cast` — почти всегда выводит на поле неопределенное поведение.
Язык C (не C++!) позволяет выполнять type punning с помощью union.
В C++ до C++20 нужно было использовать `memcpy` и... копировать. Компиляторы при этом часто способны такие копии ради type punning оптимизировать и убирать. В C++20 добавили `std::bit_cast` для той же цели.
В С++23 еще появится `start_lifetime_as` — который поможет для наиболее частого случая type punning: интерпретировать массив принятых байт (например, по сети) как осмысленную структуру/массив структур.


Последнее что нужно отметить: преобразование указателей **само по себе не является неопределенным поведением**. Так что, к примеру, стандартная цепочка `T*` -> `void*` -> `T*` при передаче объектов и функций между границами библиотек не нарушает правила алиасинга.
A вот использование указателя `A*`, чтоб через него прочитать/изменить объект другого типа `B` — проблемы начинаются здесь. 


## Полезные ссылки
1. https://en.cppreference.com/w/cpp/language/reinterpret_cast
2. https://en.cppreference.com/w/c/language/type#Compatible_types
3. https://en.wikipedia.org/wiki/Fast_inverse_square_root
4. https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8
5. https://www.yodaiken.com/2018/06/07/torvalds-on-aliasing/
6. https://en.cppreference.com/w/cpp/memory/start_lifetime_as