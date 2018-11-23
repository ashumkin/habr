## Автоматизированная сборка Delphi-приложения

Я довольно часто сталкивался с тем, что разработчики на Delphi (можно сказать традиционно) компилируют свои приложения "ручками", что далеко не production-решение, а со стороны выглядит кустарщиной и *"делаем на-коленке"*, хотя продукты бывают весьма серьёзными и продаваемыми. Вероятно, это пошло ещё с тех пор, когда для автоматизации нужно было придумывать свои *батнички*, которые запускали компилятор командной строки `dcc32` с нужными параметрами. *Некоторые даже сделали свой "Публикатор" - Delphi-expert, который делает работу сервера сборок: компилирует (правда, открытый в IDE) проект, выставляя ему взятый из какой-то БД инкрементированный номер версии, записывает некий changelog и копирует это куда-то в сетевой каталог*.
Я не буду вдаваться в исторический экскурс *как было раньше*. Я расскажу как есть/можно сейчас, и как это использовать для повышения эффективности своей работы.

Файл проекта современной версии Delphi - это `.dproj`-файл (здесь и далее я буду ориентироваться на Delphi 10 Rio, но с небольшими отличиями это верно для всех более ранних версий Delphi, начиная с 2007). В нём хранятся все настройки проекта, которые обычно изменяют в IDE (меню `Project - Options (Ctrl+Shift+F11)`). В рамках данной статьи я сконцентрируюсь на "основных", которые понадобятся для демонстрации общих принципов: это `Config` - конфигурация, `Platform` -  платформа, `OutputDirectory` - путь выходного файла и `ConditionalDefines` (директивы условной компиляции). Остальные настройки, если таковые нужно менять при сборке, я предлагаю выявить самостоятельно. Этот же `.dproj`-файл, если в него заглянуть обычным текстовым редактором, является ничем иным как скриптом сборки [MSBuild](https://ru.wikipedia.org/wiki/MSBuild) (давайте создадим простое консольное приложение и назовём его [DelphiAutomatedBuild](https://github.com/ashumkin/habr-delphi-automate-build-demo)):

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <PropertyGroup>
        <ProjectGuid>{6880AD8E-6CB3-47B9-B8E3-7304CF6E9735}</ProjectGuid>
        <ProjectVersion>18.1</ProjectVersion>
        <FrameworkType>None</FrameworkType>
        <MainSource>DelphiAutomatedBuild.dpr</MainSource>
        <Base>True</Base>
        <Config Condition="'$(Config)'==''">Debug</Config>
        <Platform Condition="'$(Platform)'==''">Win32</Platform>
        <TargetedPlatforms>1</TargetedPlatforms>
        <AppType>Console</AppType>
    </PropertyGroup>
    ...

```

*Скрипты сборки MSBuild также используются для описания проектов, например, Visual Studio. Я коснусь некоторых деталей MSBuild, но я предлагаю читателю самостоятельно освоить его азы*. Что нам это даёт? Это позволяет нам выполнить сборку Delphi-проекта из командной строки одной строчкой (*что, в свою очередь, позволяет автоматизировать сборку проекта, в том числе перейти на DevOps*)

```
msbuild /t:build DelphiAutomatedBuild.dproj
```

*Где взять MSBuild? Если установлена Delphi, то MSBuild уже тоже есть, и Delphi его использует. Скорее всего, это каталог `%WINDIR%\Microsoft.Net\Framework\v3.5`, либо найти в каталоге .Net 4.0/4.5/4.6. Но можно и скачать отдельным приложением с сайта [Microsoft](https://www.microsoft.com/ru-ru/download/details.aspx?id=48159). Далее нам понадобится MSBuild минимум 4.0, но пока хватит и того, что по умолчанию*

Если же читатель откроет командную строку в каталоге с проектом (hint: это можно *быстро* сделать, щёлкнув правой кнопкой мыши (ПКМ) на проекте в IDE - *Show in Explorer*, затем в *Проводнике* ПКМ - *Открыть окно команд*), то вышеприведённая команда не сработает: 

```
...>msbuild /t:build DelphiAutomatedBuild.dproj
"msbuild" не является внутренней или внешней
командой, исполняемой программой или пакетным файлом.
```

т.к по умолчанию, пути к MSBuild-у в `PATH` нет. Так что добавим его туда:

```
set PATH=%WINDIR%\Microsoft.Net\Framework\v3.5;%PATH%
```

Теперь повторим:

```
...>msbuild /t:build DelphiAutomatedBuild.dproj
Microsoft (R) Build Engine версии 12.0.21005.1
[Microsoft .NET Framework версии 4.0.30319.42000]
(C) Корпорация Майкрософт (Microsoft Corporation). Все права защищены.

Сборка начата 24.11.2018 0:12:14.
Проект "Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj" в узле 1 (целевые объекты build).
Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj : error MSB4057: в проекте нет целевого объекта "build".
Сборка проекта "Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj" завершена (целевые объекты build) с ошибкой.

Ошибка сборки.

"Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj" (целевой объект build ) (1) ->
  Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj : error MSB4057: в проекте нет целевого объекта "build".

    Предупреждений: 0
    Ошибок: 1

Затраченное время: 00:00:00.04

```

Сборка запустилась, но завершилась с ошибкой. В чём же дело? Почему нет задачи `build`? 

Тут мы заглянем в `.dproj`-файл, там мы найдём следующее:

```xml
...
    <Import Project="$(BDS)\Bin\CodeGear.Delphi.Targets" Condition="Exists('$(BDS)\Bin\CodeGear.Delphi.Targets')"/>
...
```

И если мы откроем файл в каталоге Delphi 
`c:\Program Files\Embarcadero\Studio\20.0\Bin\CodeGear.Delphi.Targets`, то мы увидим там ещё один MSBuild-скрипт, в котором объявлена задача `Build`:

```xml
<Target Name="Build"...
```

Т.е. нужно задать переменную окружения `BDS` (`$(VAR)` в MSBuild разыменовывает как свойство (Property) `VAR`, заданное в скрипте, так и одноимённую переменную окружения), указать в ней путь к той версии Delphi, которая будет компилировать проект (да-да, один и тот же проект можно компилировать разными версиями Delphi, лишь заменив значение переменной окружения `BDS`). Тогда скрипт проекта разыменует `$(BDS)`, найдёт общий `.Targets` файл из каталога Delphi и запустит задачу `Build`.
Сделаем это:

```
set BDS=c:\Program Files\Embarcadero\Studio\20.0
```

ещё раз:

```
...>msbuild /t:build DelphiAutomatedBuild.dproj
Microsoft (R) Build Engine версии 12.0.21005.1
[Microsoft .NET Framework версии 4.0.30319.42000]
(C) Корпорация Майкрософт (Microsoft Corporation). Все права защищены.

Сборка начата 24.11.2018 0:20:40.
Проект "Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj" в узле 1 (целевые объекты build).
CreateProjectDirectories:
  Создание каталога ".\Win32\Debug".
BuildVersionResource:
  C:\Program Files\Embarcadero\Studio\20.0\bin\cgrc.exe -c65001 DelphiAutomatedBuild.vrc -foDelphiAutomatedBuild.res 
  CodeGear Resource Compiler/Binder
  Version 1.2.2 Copyright (c) 2008-2012 Embarcadero Technologies Inc.
  
  Microsoft (R) Windows (R) Resource Compiler Version 6.0.5724.0
  
  Copyright (C) Microsoft Corporation.  All rights reserved.
  
  
  Файл "DelphiAutomatedBuild.vrc" удаляется.
_PasCoreCompile:
  C:\Program Files\Embarcadero\Studio\20.0\bin\dcc32.exe -$O- -$W+ --no-config -B -Q -TX.exe -AGenerics.Collections=System.Generics.Collections;Generics.Defaults=System.Generics.Defaults;WinTypes=Winapi.Windows;WinProcs=Winapi.Windows;DbiTypes=BDE;DbiProcs=BDE;DbiErrs=BDE -DDEBUG -E.\Win32\Debug -I"c:\program files\embarcadero\studio\20.0\lib\Win32\debug";"c:\program files\embarcadero\studio\20.0\lib\Win32\release";C:\Users\USER\Documents\Embarcadero\Studio\20.0\Imports;"C:\Program Files\Embarcadero\Studio\20.0\Imports";"C:\Users\Public\Documents\RAD Studio\5.0\Dcp";"C:\Program Files\Embarcadero\Studio\20.0\include";C:\Users\USER\AppData\Local\Programs\TestInsight\Source -LE"C:\Users\Public\Documents\RAD Studio\5.0\Bpl" -LN"C:\Users\Public\Documents\RAD Studio\5.0\Dcp" -NU.\Win32\Debug -NSWinapi;System.Win;Data.Win;Datasnap.Win;Web.Win;Soap.Win;Xml.Win;Bde;System;Xml;Data;Datasnap;Web;Soap; -O"c:\program files\embarcadero\studio\20.0\lib\Win32\release";C:\Users\USER\Documents\Embarcadero\Studio\20.0\Imports;"C:\Program Files\Embarcadero\Studio\20.0\Imports";"C:\Users\Public\Documents\RAD Studio\5.0\Dcp";"C:\Program Files\Embarcadero\Studio\20.0\include";C:\Users\USER\AppData\Local\Programs\TestInsight\Source -R"c:\program files\embarcadero\studio\20.0\lib\Win32\release";C:\Users\USER\Documents\Embarcadero\Studio\20.0\Imports;"C:\Program Files\Embarcadero\Studio\20.0\Imports";"C:\Users\Public\Documents\RAD Studio\5.0\Dcp";"C:\Program Files\Embarcadero\Studio\20.0\include";C:\Users\USER\AppData\Local\Programs\TestInsight\Source -U"c:\program files\embarcadero\studio\20.0\lib\Win32\debug";"c:\program files\embarcadero\studio\20.0\lib\Win32\release";C:\Users\USER\Documents\Embarcadero\Studio\20.0\Imports;"C:\Program Files\Embarcadero\Studio\20.0\Imports";"C:\Users\Public\Documents\RAD Studio\5.0\Dcp";"C:\Program Files\Embarcadero\Studio\20.0\include";C:\Users\USER\AppData\Local\Programs\TestInsight\Source -CC -V -VN -NB"C:\Users\Public\Documents\RAD Studio\5.0\Dcp" -NH"C:\Users\Public\Documents\RAD Studio\5.0\hpp\Win32" -NO.\Win32\Debug   DelphiAutomatedBuild.dpr   
  Embarcadero Delphi for Win32 compiler version 30.0
  Copyright (c) 1983,2015 Embarcadero Technologies, Inc.
  19 lines, 0.27 seconds, 100748 bytes code, 26044 bytes data.
Сборка проекта "Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj" завершена (целевые объекты build).

Сборка успешно завершена.
    Предупреждений: 0
    Ошибок: 0

Затраченное время: 00:00:01.32
```

Та-дам! Проект скомпилировался. В выходном каталоге `Win32\Debug` лежит наш `DelphiAutomatedBuild.exe`.

Но это отладочная сборка (по умолчанию, новый проект активируется в Debug-конфигурации), а мы хотим для выпуска релиза собирать Release-конфигурацию ([подробнее про конфигурации](http://docwiki.embarcadero.com/RADStudio/Tokyo/en/Build_Configurations_Overview)). В IDE это сделать легко, но это ручная работа, и это то, чего мы хотим избежать, то ради чего мы читаем эту статью. Заглянем опять в `.dproj`-файл, и заметим в его начале такую строку

```xml
...
        <Config Condition="'$(Config)'==''">Debug</Config>
...
```

Мы ж программисты, и понимаем, что если свойство/переменная `Config`, не задана, то по умолчанию она принимается равной `Debug`. *Это как раз то, что мы меняем в IDE (поменяйте в IDE текущую конфигурацию на Release и сохраните проект - строка сменится на*

```xml
...
        <Config Condition="'$(Config)'==''">Release</Config>
...
```
в коде для контроля исполняемого файла добавим такое:
```
    {$IFDEF RELEASE}
    WriteLn('This is RELEASE build');
    {$ENDIF RELEASE}
    {$IFDEF DEBUG}
    WriteLn('This is DEBUG build');
    {$ENDIF DEBUG}
```
и убедимся, что conditional defines в настройках проекта для Release и Debug-конфигураций содержат RELEASE и DEBUG, соответственно

Так что нужно лишь задать свойство `Config` в нужное нам значение, и собираться будет нужная конфигурация:

```
...>msbuild /t:build DelphiAutomatedBuild.dproj /p:Config=Release
Microsoft (R) Build Engine версии 12.0.21005.1
[Microsoft .NET Framework версии 4.0.30319.42000]
(C) Корпорация Майкрософт (Microsoft Corporation). Все права защищены.

Сборка начата 24.11.2018 0:48:30.
Проект "Z:\habr\delphi-automate-build\DelphiAutomatedBuild.dproj" в узле 1 (целевые объекты build).
CreateProjectDirectories:
  Создание каталога ".\Win32\Release".
...
```
```
...>Win32\Debug\DelphiAutomatedBuild.exe
This is DEBUG build

...>Win32\Release\DelphiAutomatedBuild.exe
This is RELEASE build
```

Замечательно!

Часто разработчики указывают путь отладочной (а то и релизной) сборки в какой-то каталог на своём диске, но мы автоматизируем сборку и  подразумеваем, что выполняться она будет на сервере сборок, а получать выходные файлы непонятно где в файловой системе сервера - как-то неправильно. Значит, мы должны уметь задавать этот выходной путь. Заглянем опять в `.dproj`:

```xml
...
        <DCC_ExeOutput>.\$(Platform)\$(Config)</DCC_ExeOutput>
...
```

но что это? тут нет условия (если не задано), и свойство задаётся всегда, сможем ли мы его переопределить? попробуем

```
...>msbuild /t:build DelphiAutomatedBuild.dproj /p:DCC_ExeOutput=binaries
```

Та-дам! Появился каталог `binaries`, в котором - наш `DelphiAutomatedBuild.exe`. Как же так? Тот, кто уже освоил `MSBuild`, знает, что свойства, заданные при запуске `MSBuild`-а, имеют высший приоритет, и уже не могут быть переопределены в скрипте. Сейчас нас это устраивает. Но с этим мы ещё столкнёмся...

Выходной каталог мы менять научились. Теперь нужно собирать сразу и релизную, и отладочную версии (надеюсь, не надо объяснять зачем такое надо). Конечно, можно запустить сначала с одним параметром `Config` - `Debug`, затем - с другим - `Release`, но это потребует, во-первых, дублирования остальных параметров (например, `DCC_ExeOutput` и параметра версии сборки (об этом - ниже)), а во-вторых, это придётся учитывать и при конфигурировании сервера сборок, что влечёт дублирование и там (либо написание очередного *батничка*, что лишает встроенной поддержки MSBuild-а сервером сборок). Так что требуется выполнить всё ту же одну команду

```
...>msbuild /t:build DelphiAutomatedBuild.dproj /p:DCC_ExeOutput=binaries
```

но она бы выполнила сборку обеих конфигураций. Можно так? Конечно!
Напишем свою задачу `Build`. Поскольку есть нежелание менять что-то в файле, который меняет IDE (часто самым дурацким образом; кстати, есть три замечательных инструмента  от автора эксперта [MMX](https://www.mmx-delphi.de/): [DProjNormalizer](https://www.uweraabe.de/Blog/2017/01/18/dproj-changed-or-not-changed/), [DProjSplitter](https://www.uweraabe.de/Blog/2017/08/02/working-in-a-team-dprojsplitter-might-be-helpful/) и сумма их - [ProjectMagician](https://www.uweraabe.de/Blog/2018/05/17/keep-your-project-files-clean-with-project-magician/) - для удобства отслеживания изменений `.dproj`-файлов), то сделаем отдельный файл проекта. Назовём его **DAB.ciproj** (CI-project, от CI - Continuous Integration):

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Target Name="Build">
        <MSBuild Projects="DelphiAutomatedBuild.dproj"
            Targets="Build"
            Properties="Config=Debug"/>
        <MSBuild Projects="DelphiAutomatedBuild.dproj"
            Targets="Build"
            Properties="Config=Release"/>
    </Target>
</Project>

```

Запускаем

```
...>msbuild /t:build DAB.ciproj /p:DCC_ExeOutput=binaries
```

и... получаем один файл `DelphiAutomatedBuild.exe` в `binaries`, той конфигурации, что собралась последней:

```
...>binaries\DelphiAutomatedBuild.exe
This is RELEASE build
```

`DCC_Exeoutput` задался и для каждой задачи **MSBuild** - это хорошо, но каждая конфигурация скомпилировала файл в один и тот же каталог. Тогда зададим подкаталоги соответственно конфигурации:

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Target Name="Build">
        <MSBuild Projects="DelphiAutomatedBuild.dproj"
            Targets="Build"
            Properties="Config=Debug;DCC_Exeoutput=$(DCC_ExeOutput)\Debug"/>
        <MSBuild Projects="DelphiAutomatedBuild.dproj"
            Targets="Build"
            Properties="Config=Release;DCC_Exeoutput=$(DCC_ExeOutput)\Release"/>
    </Target>
</Project>
```

Запускаем опять

```
...>msbuild /t:build DAB.ciproj /p:DCC_ExeOutput=binaries
```

и теперь на выходе мы имеем два файла

`binaries\Debug\DelphiAutomatedBuild.exe` и `binaries\Release\DelphiAutomatedBuild.exe`.
```
...>binaries\Debug\DelphiAutomatedBuild.exe
This is DEBUG build

...>binaries\Release\DelphiAutomatedBuild.exe
This is RELEASE build
```
Теперь представим, что у нас есть желание/необходимость временно задавать conditional define при сборке проекта (например, у нас есть демо-версия, в которой мы ограничиваем функциональность нашей программы, если задано переменная условной компиляции TRIAL )

В нашем демо-коде это выглядит так
```pascal
    {$IFDEF TRIAL}
    WriteLn('This is TRIAL version');
    {$ENDIF DEBUG}
```
Добавим в Debug-конфигурацию conditional define TRIAL и посмотрим, куда оно прописывается в .dproj:
```
        <DCC_Define>DEBUG;TRIAL;$(DCC_Define)</DCC_Define>
```
Ага, т.е. если задать /p:DCC_Define=TRIAL,
```
...>msbuild /t:build DAB.ciproj /p:DCC_ExeOutput=binaries
```
то
```
...>binaries\Debug\DelphiAutomatedBuild.exe
This is TRIAL version

...>binaries\Release\DelphiAutomatedBuild.exe
This is TRIAL version
```
Сработало, но как-то не так, куда-то делись DEBUG и RELEASE, а нам такого не надо, т.к. у нас там обычно куча полезных define-ов. 
А дело в том, что свойства заданные через командную строку имеют высший приоритет, и переопределяют значения в скриптах. Но выход [есть](https://stackoverflow.com/questions/8507372/can-i-add-conditional-defines-in-the-msbuild-command-line/38003499#38003499).
Определяем **переменную окружения** `DCC_Define`:

```
...>set DCC_Define=TRIAL
...>msbuild /t:build DAB.ciproj /p:DCC_ExeOutput=binaries
...
```
```
...>binaries\Debug\DelphiAutomatedBuild.exe
This is DEBUG build
This is TRIAL version
...>binaries\Release\DelphiAutomatedBuild.exe
This is RELEASE build
This is TRIAL version
```

С компиляцией разобрались, теперь можно настраивать сервер сборок, который бы после каждого изменения в центральном репозитории (я ориентируюсь на Git, но для того же SVN это тоже применимо) собирал нам проект, дабы мы ничего не забыли добавить в исходники, и прогонял тесты, буде таковые у нас есть, и мы всегда будем готовы выпустить релиз или отдать на тестирование уже готовую сборку.
Однако ж, при таких частых сборках может стать проблема нумерации версий. Какая? Каждая новая сборка будет иметь ровно ту версию, которая прописана в свойствах проекта, а менять её с каждым коммитом - как-то рутинно и не "по-нашенски", к тому же, зависит от ~~разработчика~~ человека (а что такое "человеческий фактор" - не мне вам рассказывать).
В рамках обычной для Windows/Delphi-проектов нумерации `Major.Minor.Release.Build`, *нормальный* сервер сборок, как правило, умеет увеличивать для каждой сборке число `Release`, и, естественно, передавать её в скрипты сборки. Однако ж, если мы посмотрим на то, как задаётся информация о версии в .dproj-файле

```
        <VerInfo_Keys>CompanyName=;FileDescription=;FileVersion=1.0.0.0;InternalName=;LegalCopyright=;LegalTrademarks=;OriginalFilename=;ProductName=;ProductVersion=1.0.0.0;Comments=</VerInfo_Keys>
```
мы поймём, что передавать её некуда. И даже если мы додумаемся заменить `FileVersion=0.0.0.0`, например, так `FileVersion=$(Version)`, где `Version` - переменная, которую мы бы передавали при сборке, то сборка из командной строки у нас получится, а вот уже изменять свойства проекта в IDE - нет, т.к. она будет "ругаться" на такое значение. Ну да где наша не пропадала. Видим, что это свойство, значение которого - CSV-список, одно из значений которого нам нужно заменить.
Долго не думая, перейдём сразу к делу. MSBuild 4.0 [умеет понимать скрипты на C#](https://docs.microsoft.com/ru-ru/visualstudio/msbuild/walkthrough-creating-an-inline-task?view=vs-2019), чем мы и воспользуемся (но напомню, что тогда в PATH надо прописать именно его). Накидаем такой файлик `Delphi.Version.Targets`

```xml
<Project xmlns='http://schemas.microsoft.com/developer/msbuild/2003' ToolsVersion="12.0">
  <UsingTask TaskName="__SetFileVersion" TaskFactory="CodeTaskFactory" AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.v12.0.dll">
    <ParameterGroup>
      <VerInfoKeys ParameterType="System.String" Required="true" />
      <VerInfoProperties ParameterType="Microsoft.Build.Framework.ITaskItem[]" Required="true" />
      <Out ParameterType="System.String" Output="true" />
    </ParameterGroup>
    <Task>
      <Code Type="Fragment" Language="cs"><![CDATA[
        // split values as CSV (by ";")
        String[] verInfoKeysList = VerInfoKeys.Split(';');
        Dictionary<String, String> d = new Dictionary<String, String>();
        foreach (String verInfoValue in verInfoKeysList) {
            // split values as "key=value"
            if (! String.IsNullOrEmpty(verInfoValue)) {
                String[] kv = verInfoValue.Split('=');
                d.Add(kv[0], kv[1]);
            }
        }
        if (VerInfoProperties.Length > 0) {
          foreach (ITaskItem item in VerInfoProperties) {
            String value = item.GetMetadata("Value");
            if (value.Length > 0) {
              Log.LogMessage("{0}: {1}", item.ItemSpec, value);
              d.Remove(item.ItemSpec);
              d.Add(item.ItemSpec, value);
            }
          }
        }

        List<String> L = new List<String>();
        foreach (KeyValuePair<String, String> kv in d) {
            L.Add(kv.Key + "=" + kv.Value);
        }
        _Out = String.Join(";", L.ToArray());
]]></Code>
    </Task>
  </UsingTask>

  <Target Name='_SetVersionCode_Name'>
    <Message Text="$(VerInfo_Keys)" />
    
    <ItemGroup>
      <VerInfoProperties Include="FileVersion">
        <Value>$(FileVersion)</Value>
      </VerInfoProperties>
    </ItemGroup>
    <__SetFileVersion VerInfoKeys="$(VerInfo_Keys)" VerInfoProperties="@(VerInfoProperties)">
      <Output PropertyName="VerInfo_Keys" TaskParameter="Out" />
    </__SetFileVersion>
    <Message Text="$(VerInfo_Keys)" />
  </Target>

  <Target Name='_SetFileVersion' BeforeTargets="_BuildRCFile"
      Condition="'$(FileVersion)' != ''">
     <CallTarget Targets='_SetVersionCode_Name'/>
  </Target>
</Project> 
```
Любознательный читатель наверняка уже догадывается как примерно такое использовать.
Добавим в наш `DelphiAutomatedBuild.dproj`

```xml
...
<Import Project="Delphi.Version.Targets" Condition="$(MSBuildToolsVersion) >= 4.0" />
...
```

*(Условие "$(MSBuildToolsVersion) >= 4.0" необходимо для того, чтобы проект не падал с ошибкой при сборке в IDE, которая, как мы помним, использует MSBuild 3.5, который не поддерживает UsingTask)*
Таким образом, мы импортируем задание `_SetFileVersion`, которое выполнится после задания `_BuildRCFile` (тут я немного срезал угол: это *внутреннее* задание, которое выполняется при сборке проекта - см. `$(BDS)\bin\Codegear.Common.Targets`), и только если задано свойство `FileVersion`. Это задание берёт тот самый CSV-список в переменной `VerInfo_Keys`, разбивает его на пары ключ-значение, заменяет некоторые значения на заданные, и собирает обратно в строку `VerInfo_Keys`. 

Поставим в свойствах проекта "Include version information in project" и добавим вывод текущей версии (оставим это за скобками), и:

```
...>msbuild /t:build DAB.ciproj /p:DCC_ExeOutput=binaries /p:FileVersion=4.3.2.1
```
```
...>binaries\Debug\DelphiAutomatedBuild.exe
This is RELEASE build
This is TRIAL version
This file version is 4.3.2.1
...>binaries\Release\DelphiAutomatedBuild.exe
This is DEBUG build
This is TRIAL version
This file version is 4.3.2.1
```

Profit!

### Заключение

Так мы научились автоматизированно собирать Delphi-приложения одной командой, что экономит нам время и нервы, в том числе, за счёт того, что позволяет переложить компиляцию и выпуск релизов на сервер сборок, и тем самым застраховаться от ситуаций, когда проект собирается только на машине разработчика. К тому же, позволяет автоматизировать простановку версии как каждой сборки (на каждый коммит), так и увеличение релизной версии при выпуске релиза.

В дальнейшем я ещё планирую рассказать

1. как запускать статический анализ кода (на примере FixInsight, не реклама!) во время сборки
2. как писать unit-тесты на Delphi (увы, некоторым приходится объяснять ))). И запускать их в пайплане сборки )
3. как "прикрутить" сборку Delphi-проектов к GitLab CI
4. а также, как можно использовать отладчик WinDbg, например, для поиска причин сбоя/падения приложений из-за библиотек, написанных на Delphi (ну, конечно же, как при этом интегрировать формирование необходимых для этого PDB-файлов в автосборку)