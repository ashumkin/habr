# Как найти зловреда (нет) с WinDbg
## Вступление

В этой статье я покажу, как, например, с помощью WinDbg найти, какой такой зловред (или нет) заменил адрес вызова системной функции в DLL, подгружаемым каким-нибудь приложением. *Так я, к примеру, искал почему не загружается модуль защиты в конфигурацию 1С*.
Для демонстрации нам понадобится приложение, которое загружает пару DLL: одну из них назовём **victim** (*жертва*), другую - ~~хищник~~ **injector**. Последнняя внедряется в жертву, заменяя вызов системной функции (для простоты возьмём **Sleep**), и будет вызывать AV при определённых условиях (что понадобится в следующей статье). 

Т.к. приложения, написанные на Delphi, "не падают" в core-dump из-за необработаных исключений, то наше главное приложение (DLLInjectionDemo) написано на С, линковано ранним связыванием с DLL-жертвой, а для простоты воспроизведения ситуации, будет загружать DLL-injector, переданную в опциях при запуске, и вызывать в ней метод, который навредит жертве. *Конкретно для этой статьи нам подошло бы приложение, написанное на любом языке программирования, но убьём двух зайцев сразу.*

[Исходные коды](https://github.com/ashumkin/habr-delphi-dll-injection-demo.git) приложений написаны на C и Delphi 10.3 Rio Community Edition, и компилируются MinGW и Delphi, как для Win32~~, так и для Win64 (а также FPC в Lazarus-е)~~.

Итак компилируем обе DLL и *главное* приложение

<cut/>
```powershell
> msbuild /t:build victim.dproj /p:Platform=Win32;Config=Debug;DCC_Exeoutput=.
> msbuild /t:build injector.dproj /p:Platform=Win32;Config=Debug;DCC_Exeoutput=.
> mingw32-make
```

*для сборки make-ом от MinGW нужно прописать его в PATH, конечно*


и запускаем *главное* приложение без параметров:

```powershell
C:\Users\demo>DLLInjectionDemo.exe
Sleeping 100 milliseconds
Done!
```

Затем запускаем с параметрами `-L injector` (напомню, что это для того, чтобы эмулировать поведение, когда у приложения уже что-то не так перед вызовом нужной функции)

```powershell
c:\Users\demo>DLLInjectionDemo.exe -L injector
Loading injector
Searching...
oleaut32.dll
advapi32.dll
user32.dll
kernel32.dll
kernel32.dll
user32.dll
version.dll
kernel32.dll
kernel32.dll
netapi32.dll
oleaut32.dll
Injected
Sleeping 100 milliseconds
New sleep instead of 100
Done!
```

Тут мы видим, что наше приложение ведёт себя странно, и вместо Sleep происходит что-то неожиданное. Мы смотрим в код нашего приложения и ничего не понимаем (а также представим, что это происходит со слов пользователя программы). Тогда мы просим пользователя снять дамп приложения с помощью менеджера задач (и да, я не сделал GUI-версию, так что, для того, чтобы можно было снять дамп приложения во время его работы, я добавил ключ `-i`(*interactive*)). Так что запускаем

```powershell
c:\Users\demo>DLLInjectionDemo.exe -i -L injector
Loading injector
Searching...
oleaut32.dll
advapi32.dll
user32.dll
kernel32.dll
kernel32.dll
user32.dll
version.dll
kernel32.dll
kernel32.dll
netapi32.dll
oleaut32.dll
Injected
Sleeping 100 milliseconds
New sleep instead of 100
Done!
Press ENTER...
```

 Приложение работает (в данном случае просто ждёт ввода пользователя). Снимаем дамп этого процесса: вызываем диспетчер задач, закладку процессы, находим процесс, правая кнопка мыши, в меню выбираем "Создать файл дампа процесса...", ждём показа сообщения о завершении снятия дампа и из него копируем или запоминаем путь к файлу дампа.

Затем берём в руки отладчик WinDbg, и открываем в нём этот дамп:

из меню `File` - `Open Crash Dump (Ctrl+D)`

или из командной строки сразу:

`%PROGRAM FILES%\Windows Kits\10\Debuggers\x64\windbg.exe" -z C:\Users\demo\App
Data\Local\Temp\DLLInjectionDemo.DMP`

```
Loading Dump File [C:\Users\alex\AppData\Local\Temp\DLLInjectionDemo.DMP]
User Mini Dump File with Full Memory: Only application data is available

Symbol search path is: srv*
Executable search path is: 
Windows 7 Version 7601 (Service Pack 1) MP (4 procs) Free x64
Product: WinNt, suite: SingleUserTS
Machine Name:
Debug session time: Thu Jul  4 08:46:18.000 2019 (UTC + 3:00)
System Uptime: 17 days 12:25:45.404
Process Uptime: 0 days 0:00:10.000
........................
For analysis of this file, run !analyze -v
ntdll!NtRequestWaitReplyPort+0xa:
00000000`77bcddfa c3              ret

```

подсказке `For analysis of this file, run !analyze -v` пока следовать не будем (хотя любопытствующие могут и последовать, ничего страшного не произойдёт).

Нас интересует IAT - таблица адресов импорта (Import Address Table) - DLL, в которой произошла подмена адреса (почему? потому что мы знаем, что сами хотим заменить адрес в IAT ;) ). К счастью, для анализа всех PE-структур загруженных модулей (будь то DLL или процесс) отладчик WinDbg из Windows 10 SDK (который легко и непринуждённо ставится в том числе и на Windows 7) уже имеет встроенные средства, не вынуждая нас самим отправляться на "[странные берега](https://habr.com/ru/post/266831/)", как бы мы это сделали в WinDbg 6, например.

Так что воспользуемся своим счастьем. Сначала найдём адреса загруженных модулей:

`lm`

```
0:000> lm
start             end                 module name
00000000`003b0000 00000000`003db000   injector   (deferred)             
00000000`00400000 00000000`00412000   DLLInjectionDemo   (deferred)             
00000000`00520000 00000000`00616000   victim     (deferred)             
00000000`6e580000 00000000`6e5b4000   libmingwex_2   (deferred)             
00000000`72870000 00000000`72886000   netapi32   (deferred)             
...
```

*Nota bene: На 32-битной ОС адреса, естественно отличаются от 64-битных...*

Убеждаемся, что наши DLL загружены, нам нужна IAT жертвы, для этого достаточно выполнить команду `!dh` с ключом `-a`:

`!dh  009c0000 -a` 

или можно прям
<spoiler title="!dh victim -a">

```
0:000> !dh victim -a

File Type: DLL
FILE HEADER VALUES
     14C machine (i386)
       A number of sections
5E5F4251 time date stamp Wed Mar  4 09:53:21 2020

       0 file pointer to symbol table
       0 number of symbols
      E0 size of optional header
    A18E characteristics
            Executable
            Line numbers stripped
            Symbols stripped
            Bytes reversed
            32 bit word machine
            DLL

OPTIONAL HEADER VALUES
     10B magic #
    2.25 linker version
   CE400 size of code
   1BA00 size of initialized data
       0 size of uninitialized data
   CF79C address of entry point
    1000 base of code
         ----- new -----
0000000000400000 image base
    1000 section alignment
     200 file alignment
       2 subsystem (Windows GUI)
    5.00 operating system version
    0.00 image version
    5.00 subsystem version
   F6000 size of image
     400 size of headers
       0 checksum
0000000000000000 size of stack reserve
0000000000000000 size of stack commit
0000000000100000 size of heap reserve
0000000000001000 size of heap commit
       0  DLL characteristics
   DD000 [      A4] address [size] of Export Directory
   DA000 [    105C] address [size] of Import Directory
   F3000 [    2A00] address [size] of Resource Directory
       0 [       0] address [size] of Exception Directory
       0 [       0] address [size] of Security Directory
...
```
</spoiler>
листаем до таблиц импорта kernel32.dll (в которой находится нужная нам функция **Sleep**)

*Nota bene: На 32-битной ОС количество секций_IMAGE_IMPORT_DESCRIPTOR отличается от таковых на 64-битной. Последнее, вероятно, связано с разницей компиляторов Delphi для этих платформ.*

```
...
  _IMAGE_IMPORT_DESCRIPTOR 00000000005fa03c
    kernel32.dll
              005FA398 Import Address Table
              005FA11C Import Name Table
                     0 time date stamp
                     0 Index of first forwarder reference

       003C7C7C    0 Sleep
       76EE184A    0 VirtualFree
       76EE1832    0 VirtualAlloc
       76EE16DC    0 lstrlenW
       76EE4422    0 VirtualQuery
       76EE110C    0 GetTickCount
...
```

Видим, что базовый адрес этой функции отличается от
<spoiler title="базового адреса `kernel32.dll`">

```
0:000> !dh kernel32

File Type: DLL
FILE HEADER VALUES
     14C machine (i386)
       4 number of sections
5708A7E3 time date stamp Sat Apr  9 10:57:39 2016

...
         ----- new -----
0000000076ed0000 image base
...
```
т.е. 0000000076ed0000

</spoiler>

Найдём куда же ведёт этот адрес, другими словами - какому модулю он принадлежит?

В WinDbg команда `lm` имеет опцию `а` для этого:

```
0:000> lma 003C7C7C    
Browse full module list
start             end                 module name
00000000`003b0000 00000000`003db000   injector   (deferred)             
```

Ага, вот кто виноват!  Только нам как будто непонятно что это за модуль. Посмотрим пристальней (укажем ключ `v` для `lm`)

```
0:000> lma 003C7C7C v
Browse full module list
start             end                 module name
00000000`003b0000 00000000`003db000   injector   (deferred)             
    Image path: z:\habr\DLLInjectionDemo\injector.dll
    Image name: injector.dll
    Browse all global symbols  functions  data
    Timestamp:        Wed Mar  4 09:53:12 2020 (5E5F4248)
    CheckSum:         00000000
    ImageSize:        0002B000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables:
```

А вот где находится наш виновник!

`Image path: C:\Users\demo\injector.dll`

Дальше выясняем уже откуда он взялся и как его удалить, но это уже история нас не касается.



Вот так и мне в реальных условиях достаточно быстро удалось найти на предустановленной ОС от производителя некую странную DLL, которая перехватывала вызов функции `WriteFile`.


Приложение из этой статьи я использую в следующей, в которой хочу показать читателю, как можно с помощью WinDbg выяснить почему (или хотя бы примерно где) приложение упало с Access Violation, произошедшим в DLL, написанном на Delphi, когда удалённая отладка недоступна по какой-либо причине. При том, что Delphi не имеет средств анализа дампа приложения, и не умеет генерировать PDB-файлы.

