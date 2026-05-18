# WinAPIHash

У бинарника нормальной таблицы импорта почти нет, поэтому адреса нужных функций он получает не обычным способом, а ищет во время выполнения.

В начале функции `start` сразу бросаются в глаза вызовы двух внутренних функций:

```
call sub_1400010C0
call sub_140001000
```

После просмотра кода стало понятно:

Обе функции работают не со строками, а с числами- хешами. То есть по смыслу это аналоги:

- `GetModuleHandle`
- `GetProcAddress`

только скрытые через хеширование.

## Поиск DLL

В начале `start` перед вызовом `sub_1400010C0` в `ECX` передаются два значения:

```
mov ecx, 41A815F7h
call sub_1400010C0

mov ecx, 9CD6557Dh
call sub_1400010C0
```

После остановки в x64дбг и проверки возвращаемых адресов стало видно, что это:

- `KERNEL32.DLL`
- `NTDLL.DLL`

Это также совпало со списком загруженных модулей в карте памяти процесса.

## Поискк функций

Дальше программа много раз вызывает `sub_140001000`.

Перед вызовом в `EDX` передается хэш функции, например:

```
mov edx, CA5BB4D9h
call sub_140001000
```

Я поставил брекпоинт внутри `sub_140001000` в месте, где функция уже найдена. После инструкции:

```
add rax, r8
```

в `RAX` появляется полный адрес АПИ-функции, и x64дбг сам подписывает ее имя.

Так в процессе выполнения удалось увидеть, например:

- `kernel32.GetProcessHeap`
- `ntdll.RtlZeroMemory`
- `ntdll.RtlCopyMemory`
- `kernel32.GetModuleFileNameW`
- `kernel32.CreateFileW`
- `kernel32.SetFileInformationByHandle`
- `kernel32.CloseHandle`
- `kernel32.GetStdHandle`
- `kernel32.WriteConsoleA`
- `kernel32.lstrlenA`
- `kernel32.SetConsoleCursorPosition`
- `kernel32.FlushConsoleInputBuffer`
- `kernel32.Sleep`

## Вопросики к `.rdata`

Во время просмотра файла я заметил, что в секции `.rdata` рядом лежит группа значений, очень похожих на заранее сохраненные хеши АПИ-функций.

Например, рядом находились значения:

```
CA5BB4D9
47DC14B7
C7DF3EA2
```

Они уже совпадали с функциями, которые я видел во время отладки:

```
CA5BB4D9 = GetProcessHeap
47DC14B7 = RtlZeroMemory
C7DF3EA2 = RtlCopyMemory
```

Было найдено еще одно значение, которое до этого мы явно не видели:

```
A427F6B3
```

Раз оно находилось в той же группе, проверяем его тем же алгоритмом хэширования. Это хеш функции:

```
HeapAlloc
```


## Найденные DLL

- `KERNEL32.DLL`
- `NTDLL.DLL`

## Все найденные функции

### `KERNEL32.DLL`

1. `GetProcessHeap`
2. `HeapAlloc`
3. `GetModuleFileNameW`
4. `CreateFileW`
5. `SetFileInformationByHandle`
6. `CloseHandle`
7. `GetStdHandle`
8. `WriteConsoleA`
9. `lstrlenA`
10. `SetConsoleCursorPosition`
11. `FlushConsoleInputBuffer`
12. `Sleep`

### `NTDLL.DLL`

13. `RtlZeroMemory`
14. `RtlCopyMemory`
