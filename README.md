# SimpleProtection2.exe | Решение

#### Портнов Пётр Владимирович

#### МГТУ им Н.Э.Баумана, ИУ8-15, 2019-2020

### <a name="contents"/>Оглавление

- [Преамбула](#preamble)
- [Програмное обеспечение](#software)
- [Разбор поведения](#analyzis)
- [Алгоритм](#algorithm)
- [Простейший keygen](#simple-keygen)
- [Предложения по оптимизации](#optimization-suggestions)
- [Альтернативное решение](#alternative-sollution)

### <a name="preamble"/>Преамбула

> Да простится мне нейминга со смесью венгерской и индусской нотаций
>
> Я сам сугубо за long-but-obvious нейминг, но Cutter не выравнивает настолько длинные строки с комментариями, так что приходится выкручиваться ©

### <a name="software"/>Программное обеспечение

- ОС Microsoft windows 8.1
- Cutter 1.9.0

### <a name="analyzis"/>Разбор поведения

###### Сразу оговорюсь, что опущены организационные и явные пункты

Для начала находим синтетически-сгенерированный метод `main` со стандартной сигнатурой подобного (`<код выхода> (<число аргументов командной строки>, <аргументы командной строки>, <переменные среды)> `).

> Ремарка: многие дальнейшие названия будут сразу представлены в изменённой форме, поскольку их предназначение было мной определено, что, однако, никак не влияет на поведение программы (поскольку многие из этих данных даже не хранятся в ассемблерном представлении), однако упрощает восприятие.

```assembly
;-- eip:
(fcn) main 364
  int main (int argc, char **argv, char **envp);
```

Сразу после объявления Cutter описывает локальные переменные, используемые далее в пределах функции:

```assembly
; var int32_t var_10h @ ebp-0x10
; var int32_t var_ch @ ebp-0xc
; var signed int var_8h @ ebp-0x8
; var int32_t var_1h @ ebp-0x1
```

Вспомним стандартные обозначения регистров:

![](http://flint.cs.yale.edu/cs421/papers/x86-asm/x86-registers.png)

Таким образом, первые три инструкции *настраивают* стек для данной функции: 

```assembly
0x00402c60      push    ebp        ; Сохранение предыдущего указателя на стеке
0x00402c61      mov     ebp, esp   ; Запись текущего указателя в ebp
0x00402c63      sub     esp, 0x20  ; Сдвиг указателя стека на 32 вверх (максимально возможное заполнение стека локальными данными)
```

В связи с этим, легко находится и конец функции:

```assembly
0x00402d05      mov     esp, ebp   ; Запись текущего указателя обратно в esp
0x00402d07      pop     ebp        ; Взятие со стека в ebp предыдущего указатель
0x00402d08      ret                ; Выход из функции
```

(на самом деле, достаточно было найти инструкцию `ret`)

> Под предыдущим указателем тут подразумевается указатель на предыдущий фрейм, а под текущим - соответственно, на текущий, то есть фрейм функции, в которой мы находимся (в данном случае, `int main (int argc, char **argv, char **envp)`)

Стоит также вспомнить, что аргументы передаются в функцию на стеке в обратном порядке (это будет важно при дальнейшем анализе).

Следующие две группы инструкций, судя по исходному названию вызываемой в них функции (`sub.KERNEL32.dll_HeapAlloc_40cdec`) и тому, что (судя по дальнейшему коду) они используются при вводе данных пользователя, являются, скорее всего, аллокаций пространства в куче для вводимых данных:

```assembly
0x00402d16      push    7          ; 7 ; Укладка на стек размера аллоцируемой области
0x00402d18      call    allocate_in_heap ; Вызов функции аллокации памяти в куче
0x00402d1d      add     esp, 4     ; Сдвиг указателя стека вниз на размер переданного в функцию параметра
0x00402d20      mov     dword [username_adr], eax ; Запись адреса выделенной области кучи в username_adr
0x00402d23      xor     eax, eax   ; Обнуление eax
0x00402d25      mov     ecx, dword [username_adr] ; Запись указателя из username_adr в ecx
0x00402d28      mov     dword [ecx], eax ; Зануление с 0 по 3 байт имени пользователя
0x00402d2a      mov     word [ecx + 4], ax ; Зануление с 4 по 5 байт имени пользователя
0x00402d2e      mov     byte [ecx + 6], al ; Зануление 6 байта имени пользователя
```

```assembly
0x00402d31      push    0xe        ; 14 ; Укладка на стек размера аллоцируемой области
0x00402d33      call    allocate_in_heap ; Вызов функции аллокации памяти в куче
0x00402d38      add     esp, 4     ; Сдвиг указателя стека вниз на размер переданного в функцию параметра
0x00402d3b      mov     dword [password_adr], eax ; Запись адреса выделенной области кучи в password_adr
0x00402d3e      xor     edx, edx   ; Обнуление edx
0x00402d40      mov     eax, dword [password_adr] ; Запись указателя из password_adr в eax
0x00402d43      mov     dword [eax], edx ; Зануление с 0 по 3 байт пароля
0x00402d45      mov     dword [eax + 4], edx ; Зануление с 4 по 7 байт пароля
0x00402d48      mov     dword [eax + 8], edx ; Зануление с 8 по 11 байт пароля
0x00402d4b      mov     word [eax + 0xc], dx ; Зануление с 12 по 13 байт пароля
```

> Название функции `allocate_in_heap` введено мной здесь для удобства чтения, как и новые названия для переменных, что также будет применяться для всех дальнейших функций и переменных с очевидными поведением и предназначением. Также это поможет избежать повторного определения их смысла в случае, если они снова встретятся.
>
> | Сгенерированное название переменной | Название после переименования |
> | ----------------------------------- | ----------------------------- |
> | `var10h`                            | `password_adr`                |
> | `var_ch`                            | `username_adr`                |

> `xor R R`, где `R` - некий регистр - инструкция обнуления значения этого регистра, поскольку XOR даёт нуль при равенстве операндов

Следующий набор инструкций, судя по укладке на стек некого непонятного значения (получаемого вызовом другой функции), строкового литерала (который соответствует выводу в консоль при старте программы) и некой магической константы - это вызов функции стандартного текстового вывода (e.g. `std::cout.operator<<(*char)`), за которой следует функция переноса строки, а первый *непонятный* оператор - это, скорее всего, получение потока вывода или шаблон выводимых данных, что не столь важно, поскольку подобный сниппет встречается для всех участков с выводом текста в консоль. Также можно увидеть во всех подобных грппах инструкций последующий вызов некой функции, вероятнее всего, перевода строки.

```assembly
0x00402d4f      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; Получение потока вывода
0x00402d54      push    str.Enter_your_username: ; 0x425d90 ; "Enter your username:" ; Выводимый текст
0x00402d59      push    0x42a0d8   ; *Магическая константа*
0x00402d5e      call    console_output_text ; Вывод текста в консоль
0x00402d63      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402d66      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402d68      call    console_new_line ; Перенос строки в консоли
```

Как было сказано, аналогичные блоки также разбросаны по другим частям функции main и соответствуют выводу текста в консоль в случае ввода различных данных:

```assembly
0x00402d9e      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402da3      push    str.Type_your_license_number: ; 0x425da8 ; "Type your license number:" ; Выводимый текст
0x00402da8      push    0x42a0d8   ; *Магическая константа*
0x00402dad      call    console_output_text ; Вывод текста в консоль
0x00402db2      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402db5      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402db7      call    console_new_line ; Перенос строки в консоли
```

```assembly
0x00402de2      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402de7      push    str.License_Key_Accepted ; 0x425dc4 ; "License Key Accepted!" ; Выводимый текст
0x00402dec      push    0x42a0d8   ; *Магическая константа*
0x00402df1      call    console_output_text ; Вывод текста в консоль
0x00402df6      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402df9      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402dfb      call    console_new_line ; Перенос строки в консоли
```

```assembly
0x00402e04      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402e09      push    str.Invalid_License_Key ; 0x425ddc ; "Invalid License Key!" ; Выводимый текст
0x00402e0e      push    0x42a0d8   ; *Магическая константа*
0x00402e13      call    console_output_text ; Вывод текста в консоль
0x00402e18      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402e1b      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402e1d      call    console_new_line ; Перенос строки в консоли
```

```assembly
0x00402e31      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402e36      push    str.Hint:_license_number_looks_like:_XXXX_XXXX_XXXX ; 0x425df4 ; "Hint: license number looks like: XXXX-XXXX-XXXX" ; Выводимый текст
0x00402e3b      push    0x42a0d8   ; *Магическая константа*
0x00402e40      call    console_output_text ; Вывод текста в консоль
0x00402e45      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402e48      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402e4a      call    console_new_line ; Перенос строки в консоли
```

```assembly
0x00402e55      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402e5a      push    str.You_have_only_3_tries ; 0x425e24 ; "You have only 3 tries!" ; Выводимый текст
0x00402e5f      push    0x42a0d8   ; *Магическая константа*
0x00402e64      call    console_output_text ; Вывод текста в консоль
0x00402e69      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402e6c      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402e6e      call    console_new_line ; Перенос строки в консоли
```

Аналогично определяем два блока кода, читающих пользовательский ввод:

- Имени пользователя (единоразово):

```assembly
0x00402d6d      mov     ecx, dword [username_adr] ; Запись указателя на имя пользователя в ecx
0x00402d70      push    ecx        ; Укладка на стек указателя из ecx
0x00402d71      push    0x42a1f0   ; *Магическая константа*
0x00402d76      call    console_read ; Чтение значения в кучу из консоли
0x00402d7b      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
```

- Пароля (циклически):

```assembly
0x00402dbc      mov     eax, dword [password_adr] ; Укладка адреса пароля на стек ...
0x00402dbf      push    eax        ; ... через eax
0x00402dc0      push    0x42a1f0   ; *Магическая константа*
0x00402dc5      call    console_read ; Чтение значения в кучу из консоли
0x00402dca      add     esp, 8     ; Сдвиг указателя стека на 8 вниз (размер положенных для вызова функций значений)
```

Как можно увидеть, вторая из перечисленных групп инструкций вывода сообщений соответствует сообщению о верно введённых данных, поэтому изучим, в каком случае исполнение доходит именно до неё.

Инструкция `0x00402de0   jle   0x402e04` пропускает обработку верно введённых данных, однако для понимания её поведения необходимо понять значение стека и регистров, с которыми работают предыдущие инструкции, так что вернёмся к инструкциям, следующим за первым выводом в консоль.

Также есть смысл обратиться к графовому представлению последовательности инструкций.

![](./images/graph0.png)

Как видно, до этой проверки присутствуют и другие, изучим происходящее там.

В первую очередь, в глаза бросается пара инструкций, выполняющих прыжок в конец программы в случае равенства локальной переменной `3`.

```assembly
0x00402d94      cmp     dword [attempt_id], 3 ; Сравнение значения счётчика попыток с 3
0x00402d98      jge     0x402e78   ; Переход (в конец функции) в случае достижения счётчиком попыток значения 3
```

Судя по тому, что эта переменная инициализируется выше значением `0` и инкрементируется далее для каждой случая, когда она не достигла этого значения, это счётчик попыток ввода пароля:

```assembly
0x00402e22      mov     eax, dword [attempt_id] ; Запись счётчика попыток в eax для изменения
0x00402e25      add     eax, 1     ; Инкремент счётчика попыток внутри eax
0x00402e28      mov     dword [attempt_id], eax ; Запись инкрементированного значения счётчика попыток обратно в локальную переменную
```

Выделим сразу несколько блоков, связанных с обработкой конкретных значений этой переменной после инкремента, но до проверки:

Для `2`, блок, выводящий шаблон пароля:

```assembly
0x00402e2b      cmp     dword [attempt_id], 2 ; Сравнение значения счётчика попыток с 2
0x00402e2f      jne     0x402e4f   ; Пропуск вывода подсказки в случае неравенства значения счётчика 2
0x00402e31      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402e36      push    str.Hint:_license_number_looks_like:_XXXX_XXXX_XXXX ; 0x425df4 ; "Hint: license number looks like: XXXX-XXXX-XXXX" ; Выводимый текст
0x00402e3b      push    0x42a0d8   ; *Магическая константа*
0x00402e40      call    console_output_text ; Вывод текста в консоль
0x00402e45      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402e48      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402e4a      call    console_new_line ; Перенос строки в консоли
```

Для `3` - сообщение о закончившихся попытках:

```assembly
0x00402e4f      cmp     dword [attempt_id], 3 ; Сравнение значения счётчика попыток с 3
0x00402e53      jne     0x402e73   ; Прыжок к верхушке цикла в случае, если счётчик не равен 3
0x00402e55      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; *Магическая константа*
0x00402e5a      push    str.You_have_only_3_tries ; 0x425e24 ; "You have only 3 tries!" ; Выводимый текст
0x00402e5f      push    0x42a0d8   ; *Магическая константа*
0x00402e64      call    console_output_text ; Вывод текста в консоль
0x00402e69      add     esp, 8     ; Сдвиг указателя стека на 8 вниз
0x00402e6c      mov     ecx, eax   ; Запись результата вызова функции в ecx
0x00402e6e      call    console_new_line ; Перенос строки в консоли
```

> Обозначим локальную переменную, на основе которой, в итоге, определяется результат, как `hash`, потому что она мутная и непонятная, а всё, что мутное и непонятное - это хэш. ©

Заметим, вообще, что вся эта ветка обрабатывается в том случае, когда результат вызова некой функции меньше либо равен нулю. Эта функция принимает на вход адрес введённого для данной итерации пароля и байтового значения, порождаемого функцией выше (обозначим его, как `name_hash`, поскольку функция, возвращающая его, принимает на вход именно имя пользователя):

```assembly
0x00402dcd      movzx   ecx, byte [name_hash] ; Укладка name-hash'а (как 0-filled 32-битного значения) на стек ...
0x00402dd1      push    ecx        ; ... через ecx
0x00402dd2      mov     edx, dword [password_adr] ; Укладка адреса пароля на стек ...
0x00402dd5      push    edx        ; ... через edx
0x00402dd6      call    check_password ; Вызов функции обработки введённых данных
0x00402ddb      add     esp, 8     ; Сдвиг указателя стека на размер переданных в функцию параметров
0x00402dde      test    eax, eax   ; Проверка на равенство значения в eax 0
0x00402de0      jle     0x402e04   ; Прыжок к обработке неверного пароля в случае, когда значение в eax оказалось меньше либо равно 0
```

Обратимся, для начала, к функции, генерирующей `name_hash`, тем более, что в пределах `main`, она вызывается всего один раз, сразу после ввода имени пользователя, обозначив её за `hash_username`:

```assembly
0x00402d7e      mov     edx, dword [username_adr] ; Запись в edx адреса имени пользователя в куче
0x00402d81      push    edx        ; Укладка на стек адреса имени пользователя из edx
0x00402d82      call    hash_username ; Вызов функции обработки введённого имени пользователя
0x00402d87      add     esp, 4     ; Сдвиг указателя стека на размер переданных в функцию параметров
0x00402d8a      mov     byte [name_hash], al ; Запись результата вызова hash_username в локальную переменную name_hash
```

Кстати, сразу после неё следует установка начального значения счётчика цикла:

```assembly
0x00402d8d      mov     dword [attempt_id], 0 ; Установка номера попытки в 0
```

Так вот, Cutter определяет её, как `sub.Username_cannot_be_empty_402bc0`, что намекает на то, что она лишь проверяет, что строка, содежащая имя пользователя, не пустая. Обратимся к её коду:
Сигнатура + локальные переменные (все уже проименованы мной):

```assembly
(fcn) hash_username 158
  hash_username (uint32_t username_adr);
; var uint32_t username_len @ ebp-0x18
; var int32_t nxt_char_adr @ ebp-0x14
; var int32_t hash @ ebp-0x10
; var int32_t loop_counter @ ebp-0xc
; var uint32_t tst_char_adr @ ebp-0x8
; var uint32_t tst_char @ ebp-0x1
; arg uint32_t username_adr @ ebp+0x8
```

Стандартные начало:

```assembly
0x00402bc0      push    ebp        ; Сохранение предыдущего указателя на стеке
0x00402bc1      mov     ebp, esp   ; Запись текущего указателя в ebp
0x00402bc3      sub     esp, 0x18  ; Сдвиг указателя стека на 24 вверх (максимально возможное заполнение стека локальными данными)
```

и конец функции:

```assembly
0x00402c5a      mov     esp, ebp   ; Запись текущего указателя обратно в esp
0x00402c5c      pop     ebp        ; Взятие со стека в ebp предыдущего указатель
0x00402c5d      ret                ; Выход из функции
```

И, действительно, первая проверка, которую мы встречаем, оказывается проверка строки на пустоту и возврат, в таком случае, из функции результата в виде байта `0`:

```assembly
0x00402bc6      cmp     dword [username_adr], 0 ; Сравнение 0'го символа строки с '\0' (конец строки)
0x00402bca      jne     0x402bee   ; Прыжок к длинному пути, в случае непустой строки
0x00402bcc      push    console_output_last_arg ; 0x4041d0 ; "U\x8b\xecj\n\x8bE\b\x8b\b\x8bU\b\x03Q\x04\x8b\xca\xe8I\v" ; Получение потока вывода
0x00402bd1      push    str.Username_cannot_be_empty ; 0x425d74 ; "Username cannot be empty!!!" ; Выводимый текст
0x00402bd6      push    0x42a0d8   ; *Магическая константа*
0x00402bdb      call    console_output_text ; Вывод текста в консоль
0x00402be0      add     esp, 8
0x00402be3      mov     ecx, eax   ; Запись из eax в ecx
0x00402be5      call    console_new_line ; Перенос строки в консоли
0x00402bea      xor     al, al     ; Запись в байт результата 0
0x00402bec      jmp     0x402c5a   ; Прыжок в конец функции
```

Однако, в противном случае, нас встречает циклическая структура, вычисляющая в `al` некоторое более сложное значение.

![](./images/graph1.png)

Можно попробовать изучить её, однако, возможно, стоит довериться названию и предположить, что длинный путь лишь проверяет иные случаи пустоты строки (спойлер: нет).

Посмотрим лучше, как пользуется этим значением функция проверки пароля, которую мы обнаружили ранее:

Шаблонно, сигнатуры и локальные переменные:

```assembly
(fcn) check_password 169
  check_password (int32_t password_adr, int32_t name_hash);
; var int32_t __result_cpy @ ebp-0x20
; var uint32_t password_len @ ebp-0x1c
; var int32_t nxt_char_adr @ ebp-0x18
; var uint32_t hash @ ebp-0x14
; var int32_t result @ ebp-0x10
; var int32_t loop_counter @ ebp-0xc
; var int32_t tst_char_adr @ ebp-0x8
; var uint32_t tst_char @ ebp-0x1
; arg int32_t password_adr @ ebp+0x8
; arg int32_t name_hash @ ebp+0xc
```

Пролог:

```assembly
0x00402c60      push    ebp        ; Сохранение предыдущего указателя на стеке
0x00402c61      mov     ebp, esp   ; Запись текущего указателя в ebp
0x00402c63      sub     esp, 0x20  ; Сдвиг указателя стека на 32 вверх (максимально возможное заполнение стека локальными данными)
```

Эпилог:

```assembly
0x00402d02      mov     eax, dword [result] ; Запись в eax результата работа функции
0x00402d05      mov     esp, ebp   ; Запись текущего указателя обратно в esp
0x00402d07      pop     ebp        ; Взятие со стека в ebp предыдущего указатель
```

Неожиданно, здесь нас встречает практически идентичный набор инструкций во всё той же циклической структуре:

![](./images/graph2.png)

Поэтому пойдём с конца.

В самом низу нас ожидает проверка, выбирающая, какой *статус-код* будет возвращён в результате вызова этой функции:

![](./images/graph3.png)

Необходимая нам ветка (левая на рисунке)

```assembly
0x00402ce6      mov     dword [result], 1 ; Запись результата 1
0x00402ced      mov     eax, dword [result] ; Запись результата в eax
0x00402cf0      mov     dword [result_cpy], eax ; Запись результата из eax в result_cpy
0x00402cf3      jmp     0x402d02   ; Прыжок в конец функции
```

достигается в том случае, когда значение некой локальной переменной (обозначенной за `hash`) принимает значение равное `0x100`:

```assembly
0x00402cdd      cmp     dword [hash], 0x100 ; Сравнение хэша с 256
0x00402ce4      jne     0x402cf5   ; Прыжок к обработке неверного пароля в случае неравенства хэша 256
```

В ином случае мы получаем результат `-1`, который, как мы уже видели выше, приведёт к обработке ошибочного ввода:

```assembly
0x00402cf5      mov     dword [result], 0xffffffff ; -1 ; Запись результата -1
0x00402cfc      mov     ecx, dword [result] ; Запись результата в ecx
0x00402cff      mov     dword [result_cpy], ecx ; Запись результата из ecx в result_cpy
0x00402d02      mov     eax, dword [result] ; Запись в eax результата работа функции
```

> Фактически, запись в `result_cpy` здесь не имеет никакого смысла, поскольку эта переменная никак не используется. Вероятно, компилятор сгенерировал её, поскольку она присутствовала в исходном коде, где та использовалась для того, чтобы точка выхода из функции была одна, и не удалил её, в качестве оптимизации.

Итак, цель ясна, необходимо понять, каким образом мы можем получить в `hash` значение 256.

Теперь уж нам придётся рассмотреть то многоблочную структуру, поскольку путь к записи этой переменной идёт именно через неё. Имеем три блока:

```assembly
0x00402c86      mov     ecx, dword [password_adr] ; Запись в tst_char_adr адреса нулевого символа ...
0x00402c89      mov     dword [tst_char_adr], ecx ; ... через ecx ...
0x00402c8c      mov     edx, dword [tst_char_adr] ; ... и его копия в edx
0x00402c8f      add     edx, 1     ; Запись адреса следующего за 0'м символа в nxt_char_adr ...
0x00402c92      mov     dword [nxt_char_adr], edx ; ... через инкремент edx
```

```assembly
0x00402c95      mov     eax, dword [tst_char_adr] ; Запись в tst_char проверяемого сейчас символа ...
0x00402c98      mov     cl, byte [eax] ; ... через запись адреса в eax, ...
0x00402c9a      mov     byte [tst_char], cl ; ... а значения в cl
0x00402c9d      add     dword [tst_char_adr], 1 ; Инкремент адреса проверяемого символа
0x00402ca1      cmp     byte [tst_char], 0 ; Проверка проверяемого символа на равенство 0 (конец строки)
0x00402ca5      jne     0x402c95   ; Повтор блока в том случае, если конец строки не достигнут
```

```assembly
0x00402ca7      mov     edx, dword [tst_char_adr] ; Запись в password_len длины строки ...
0x00402caa      sub     edx, dword [nxt_char_adr] ; ... за счёт разности адреса конца строки, увеличенного на 1, и адреса первого символа ...
0x00402cad      mov     dword [password_len], edx ; ... через edx
0x00402cb0      mov     eax, dword [loop_counter] ; Запись в eax счёчика цикла ...
0x00402cb3      cmp     eax, dword [password_len] ; ... для сравнения с длиной строки
0x00402cb6      jae     0x402cdd   ; Выход из цикла, в случае, когда все символы перебраны (не считая '\0')
```

Понимание того, как обычно хранятся строки, позволяет осознать, что, фактически, это код вычисления длины строки и сравнения её с неким другим числом (в случае равенства (или превышения, которое, в прочем, практически недостижмо) происходит выход из цикла).

Судя по множеству особенностей кода, исходный код программы был написан на C++ или его родственнике, потому что функция вычисления длины строки не только идентична его стандартной, но также заинлайнена, что описано в нативном методе. Это же и объясняет наличие идентичного блока в `hash_username`.

Определить роль локальной переменной, с которой мы сравниваем строку, можно исходя из того, что, вообще, код очень похож на шаблонный перебор символов строки, впрочем эту идею подтверждает также и блок инкремента:

```assembly
0x00402c7d      mov     eax, dword [loop_counter] ; Инкремент счётчика цикла ...
0x00402c80      add     eax, 1     ; ... через eax ...
0x00402c83      mov     dword [loop_counter], eax ; ... в локальную переменную
```

И её начальное зануление (с пропуском, собственно, инкремента):

```assembly
0x00402c66      mov     dword [result], 0 ; Обнуление локальной переменной для хранения результата
0x00402c6d      mov     dword [hash], 0 ; Установка начального значения хэша в 0
0x00402c74      mov     dword [loop_counter], 0 ; Обнуление локальной переменной для хранения счётчика итераций
0x00402c7b      jmp     0x402c86   ; Прыжок к началу цикла
```

Итак, локальная переменная (обозначенная мной как `loop_counter`) есть ничто иное, как счётчик цикла, а весь код выше, перебором символов строки вплоть до её конца.

Дело остаётся за малым, анализом двух блоков просчитки хэша, основанных на очевидной адресной арифметике (получение i'го символа строки) и XOR'е:

Проверка i'го символа на неравенство символу `'-'` (ещё одно подтверждение гипотезы о том, что происходит в функции):

```assembly
0x00402cb8      mov     ecx, dword [password_adr] ; Запись в ecx i'го символа строки ...
0x00402cbb      add     ecx, dword [loop_counter] ; ... за счёт увеличения адреса 0'го символа на счётчик
0x00402cbe      movsx   edx, byte [ecx] ; Запись в edx i'го символа ...
0x00402cc1      cmp     edx, 0x2d  ; 45 ; ... для сравнения i'го символа с '-'
0x00402cc4      je      0x402cdb   ; Пропуск блока и переход к инкременту в случае, когда i'й символ - это '-'
```

И, непосредственно, высчитывание хэша:

```assembly
0x00402cc6      mov     eax, dword [password_adr] ; Запись в eax i'го символа строки ...
0x00402cc9      add     eax, dword [loop_counter] ; ... за счёт увеличения адреса 0'го символа на счётчик
0x00402ccc      movsx   ecx, byte [eax] ; Запись в ecx i'го символа
0x00402ccf      movzx   edx, byte [name_hash] ; Запись в edx name-hash'а
0x00402cd3      xor     ecx, edx   ; Инкремент хэша ...
0x00402cd5      add     ecx, dword [hash] ; ... на XOR i'го символа и name-hash'а ...
0x00402cd8      mov     dword [hash], ecx ; ... через ecx
0x00402cdb      jmp     0x402c7d   ; Промежуточный прыжок к инкрементору цикла
```

> Легко заметить, что в двух блоках дублируются инструкции получения i'го символа. Вероятно, было бы эффективнее один раз записывать его в локальную переменную, а далее проводить все манипуляции с ней.

Здесь мы уже встречаем саму хэш-функцию, для которой нам нужно определить правильные значения. Осталось определить алгоритм нахождения `name_hash`.

Вернёмся к `hash_username`.

Снова встречаем начало циклического перебора:

```assembly
0x00402bee      mov     dword [hash], 0 ; Установка начального значения хэша в 0
0x00402bf5      mov     dword [loop_counter], 0 ; Обнуление счётчика цикла
0x00402bfc      jmp     0x402c07   ; Прыжок к сложной логике
```

Инкремент:

```assembly
0x00402bfe      mov     eax, dword [loop_counter] ; Инкремент счётчика цикла ...
0x00402c01      add     eax, 1     ; ... через eax ...
0x00402c04      mov     dword [loop_counter], eax ; ... в локальную переменную
```

Вычисление строки в 3 блока с проверкой цикла на достижение своей границы:

```assembly
0x00402c07      mov     ecx, dword [username_adr] ; Запись в tst_char_adr адреса нулевого символа ...
0x00402c0a      mov     dword [tst_char_adr], ecx ; ... через ecx ...
0x00402c0d      mov     edx, dword [tst_char_adr] ; ... и его копия в edx ...
0x00402c10      add     edx, 1     ; Запись адреса следующего за 0'м символа в nxt_char_adr ...
0x00402c13      mov     dword [nxt_char_adr], edx ; ... через edx
```

```assembly
0x00402c16      mov     eax, dword [tst_char_adr] ; Запись в tst_char проверяемого сейчас символа ...
0x00402c19      mov     cl, byte [eax] ; ... через запись адреса в eax, ...
0x00402c1b      mov     byte [tst_char], cl ; ... а значения в cl
0x00402c1e      add     dword [tst_char_adr], 1 ; Инкремент адреса проверяемого символа
0x00402c22      cmp     byte [tst_char], 0 ; Проверка проверяемого символа на равенство 0 (конец строки)
0x00402c26      jne     0x402c16   ; Повтор блока в том случае, если конец строки не достигнут
```

```assembly
0x00402c28      mov     edx, dword [tst_char_adr] ; Запись в username_len длины строки ...
0x00402c2b      sub     edx, dword [nxt_char_adr] ; ... за счёт разности адреса конца строки, увеличенного на 1, и адреса первого символа ...
0x00402c2e      mov     dword [username_len], edx ; ... через edx
0x00402c31      mov     eax, dword [loop_counter] ; Запись в eax счёчика цикла ...
0x00402c34      cmp     eax, dword [username_len] ; ... для сравнения с длиной строки
0x00402c37      jae     0x402c57   ; Выход из цикла, в случае, когда все символы перебраны (не считая '\0')
```

В блоке же просчёта хэша мы обнаруживаем интересную особенность, которая нам на руку: вычисление переменной `hash` никогда не производится с учётом её предыдущего значения, то естб хэш каждого символа перетирает собой значение предыдущего, а значит хэш username'а есть ничто иное, как хэш его последнего символа.

Сам же он высчитывается за счёт арифметических и побитовых операций:

```assembly
0x00402c39      mov     ecx, dword [username_adr] ; Запись в edx i'го символа строки ...
0x00402c3c      add     ecx, dword [loop_counter] ; ... за счёт увеличения в ecx адреса 0'го символа на счётчик цикла ...
0x00402c3f      movsx   edx, byte [ecx] ; ... и получения значения (с знаковым заполнением) по этому адресу
0x00402c42      lea     eax, [edx + edx + 0x17] ; Запись в eax значения, равного i'му символу, умноженному на 2 и увеличенному на 23
0x00402c46      and     eax, 0x8000007f ; Применение на eax маски (most-significant бит и 7 last-significant битов)
0x00402c4b      jns     0x402c52   ; Пропуск блока, если первый бит eax - 0, т.е. исходный символ был меньше 127
```

```assembly
0x00402c4d      dec     eax        ; Декремент eax
0x00402c4e      or      eax, 0xffffff80 ; 4294967168 ; Заполнение 26 most-significant битов единицами
0x00402c51      inc     eax        ; Инкремент eax
```

Собственно, та самая перезапись хэша:

```assembly
0x00402c52      mov     dword [hash], eax ; Запись в хэш значения из eax
0x00402c55      jmp     0x402bfe   ; Прыжок к инкременту цикла
```

В конце же метода, нас встречает вполне очевидная запись из поля байта в последний байт регистра `eax`:

```assembly
0x00402c57      mov     al, byte [hash] ; Запись в байт результата значения хэша
```

В общем-то, на этом анализ программы завершается, поскольку теперь мы обладаем знаниями достатычными для генерации пароля на основании имени пользователя. 

### <a name="algorithm"/>Алгоритм

Итак, алгоритм проверки соответствия имени пользователя и пароля следующий:

1. Вычисление хэша `username`'а (`username_hash`):
   1. В случае пустоты возвращается `0` и выводится сообщение об ошибке
   2. В противном, берётся последний символ и для него находится значение хэш-функции, являющейся значением хэш-функции всего имени пользователя:
      1. Преобразовании из `int8` (ASCII) в `int32` с знаковым заполнением 24 most-significant битов данного символа
      2. Вычисление суммы двух полученных величин и увеличение результата на `0x17`
      3. Применение к полученной сумме маски `0x8000007F`
      4. В случае равенства most-significant бита `1`, также уменьшение на `1`, дизъюнкция с `0xFFFFFF80` и увеличение на `1`
2. Вычисление хэш-функции (`hash`) от `password` и `username_hash`:
   1. Сумма хэш-функций каждого символа пароля:
      1. Преобразование из `int8` (ASCII) в `int32` с знаковым заполнением 24 most-significant битов данного символа
      2. Преобразование из `int8` (ASCII) в `int32` с ззаполнением `0` 24 most-significant битов данного символа
      3. XOR над полученными значениями
3. Сравнение `hash` с `0x100` (`256`)
   1. В случае равенства - код `1`
   2. В противном случае - код `-1`

По итогу, положительное значение кода указывает на соответствие имени пользователя и пароля, а всякое иное - на несоответствие.

### <a name="simple-keygen"/>Простейший keygen

Опуская вопросы оптимизации генератора ключей, нам достаточно просто, для введённого имени пользователя, определять его хэш, а далее перебирать (благо, число символов является фиксированным) возможные комбинации символов в промежутке от `0x20` (невключительно, поскольку, пробел не распознаётся, как допустимый символ) до `0x80` (невключительно, дабы не упереться в несовпадение кодировок), высчитывая для них соответствие хэшей.

*Ленивая* реализация на Java (благо, какие-то моменты могут быть оптимизированы JIT, однако, всё равно, это отнюдь не эффективное решение, а лишь элементарная демонстрация идеи):

```java
package ru.progrm_jarvis.bmstu.ti.lab3;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

import static java.lang.System.in;
import static java.lang.System.out;

public class KeyGenMain {
  private static final int REQUIRED = 0x100, MIN = 0x21, MAX = 0x80;

  public static void main(String[] args) {
    String username;
    out.println("Insert wanted username:");
    try (final BufferedReader reader = new BufferedReader(new InputStreamReader(in))) {
      do {
        username = reader.readLine();
      } while (username == null || username.isEmpty());
    } catch (final IOException e) {
      throw new RuntimeException("An exception occurred while trying to read console input", e);
    }

    int usernameHash = username.charAt(username.length() - 1) << 24 >> 24;
    usernameHash = (usernameHash + usernameHash + 0x17) & 0x8000007F;
    if (usernameHash >>> 31 == 1) usernameHash = ((usernameHash - 1) | 0xFFFFFF80) + 1;
    usernameHash = usernameHash & 0xFF;

    String result = null;
    loop:
    {
      for (int char1 = MIN; char1 < MAX; char1++) {
        final int char1hash = (char1 << 24 >> 24) ^ usernameHash;
        for (int char2 = MIN; char2 < MAX; char2++) {
          final int char2hash = (char2 << 24 >> 24) ^ usernameHash;
          for (int char3 = MIN; char3 < MAX; char3++) {
            final int char3hash = (char3 << 24 >> 24) ^ usernameHash;
            for (int char4 = MIN; char4 < MAX; char4++) {
              final int char4hash = (char4 << 24 >> 24) ^ usernameHash;
              for (int char5 = MIN; char5 < MAX; char5++) {
                final int char5hash = (char5 << 24 >> 24) ^ usernameHash;
                for (int char6 = MIN; char6 < MAX; char6++) {
                  final int char6hash = (char6 << 24 >> 24) ^ usernameHash;
                  for (int char7 = MIN; char7 < MAX; char7++) {
                    final int char7hash = (char7 << 24 >> 24) ^ usernameHash;
                    for (int char8 = MIN; char8 < MAX; char8++) {
                      final int char8hash = (char8 << 24 >> 24) ^ usernameHash;
                      for (int char9 = MIN; char9 < MAX; char9++) {
                        final int char9hash = (char9 << 24 >> 24) ^ usernameHash;
                        for (int char10 = MIN; char10 < MAX; char10++) {
                          final int char10hash = (char10 << 24 >> 24) ^ usernameHash;
                          for (int char11 = MIN; char11 < MAX; char11++) {
                            final int char11hash = (char11 << 24 >> 24) ^ usernameHash;
                            for (int char12 = MIN; char12 < MAX; char12++) {
                              final int char12hash = (char12 << 24 >> 24) ^ usernameHash;
                              if (char1hash + char2hash + char3hash + char4hash
                                  + char5hash + char6hash + char7hash + char8hash
                                  + char9hash + char10hash + char11hash + char12hash
                                  == REQUIRED) {
                                result = ""
                                    + (char) char1
                                    + (char) char2
                                    + (char) char3
                                    + (char) char4
                                    + '-'
                                    + (char) char5
                                    + (char) char6
                                    + (char) char7
                                    + (char) char8
                                    + '-'
                                    + (char) char9
                                    + (char) char10
                                    + (char) char11
                                    + (char) char12;
                                break loop;
                              }
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
    out.println(result == null
        ? "Could not generate password for you, so sorry" : "Generated password: `" + result + '`');
  }
}
```

> Так писать код не надо

Вывод keygen'а:

```log
Insert wanted username:
ADDMINS
Generated password: `!!!!-!!!!-!9==`

Process finished with exit code 0
```

Победная проверка:

![](./images/success.png)



### <a name="optimization-suggestions"/>Предложения по оптимизации

Как было отмечено, в самой программе есть несколько моментов, которые можно оптимизировать, а именно можно:

- Кэшировать значение функции, высчитывающей длину строки, вместо того, чтобы высчитывать его для каждой итерации
- Кэшировать i'е значение строки, вместо того, чтобы доставать его из кучи 2 раза подряд
- Не использовать временную переменную для хранения результата вызова функции проверки пароля в её конце, т.к. она не используется
- Выводить сообщение, сообщающее о пустом username'е, не внутри функции высчитывания хэша этой строки, а в вызывающем методе, тем самым не нарушая single-responsibility принцип и допуская более гибкую обработку этой ошибки

### <a name="alternative-sollution"/>Альтернативное решение

Стоит также отметить, что, имея допущение в виде разрешения редактировать исходную программу, все манипуляции можно было свести к простейшему инвертированию одной из инструкций прыжка, за счёт чего валидными стали бы любые данные, кроме тех, которые являются валидными в немодифицированной программе.

Например можно было заменить:

```assembly
0x00402de0      jle     0x402e04   ; Прыжок к обработке неверного пароля в случае, когда значение в eax оказалось меньше либо равно 0
```

на

```assembly
0x00402de0      jg      0x402e04   ; Прыжок к обработке неверного пароля в случае, когда значение в eax оказалось больше 0
```