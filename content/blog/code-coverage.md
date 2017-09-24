---
title: "Code Coverage"
date: 2017-09-23T11:28:11+03:00
draft: false
---

# Введение
------------------------------
Code coverage, или покрытие кода, это мера, используемая при тестировании программного обеспечения. Она показывает процент, насколько исходный код программы был протестирован (wikipedia). Code coverage удобно использовать как вспомогательный инструмент, чтобы понять какой код в данном случае исполнятеся. Для примера можно привести различные JavaScript-движки: JSC, ChakraCore, V8. В этой статье будут показаны примеры сборки JSC и ChakraCore с опцией покрытия кода и получения статистики с использованием компилятора clang.

# Общие вопросы
------------------------------
## Сборка проекта
Опции отвечающая за компиляцию проекта с покрытием кода:
```
-fprofile-instr-generate -fcoverage-mapping
```
Добавить их необходимо как во флаги компиляции, так и линковки. На выходе вы получите "обычный" исполняемый файл, как если бы не применялась опция покрытия кода.

## Получение статистики
В конце работы исполняемого файла будет генерироваться отчёт. По-умолчанию файл будет называться `default.profraw`. Тут важно предупредить, что файл должен завершиться нормально, никаких `SIGSEGV` и прочего, обычный `exit()`. Файл, в который будет записываться статистика, можно указать через переменную окружения `LLVM_PROFILE_FILE`.

Для тестов будем использовать следующий кусок кода:
```js
let o = {};
for (let i in {xx: 0}) {
    o[i];
}
```

Следующим шагом будет использование команды `llvm-profdata`:
```bash
llvm-profdata merge -sparse default.profraw -o default.profdata
```
Если у вас стоит `llvm` до версии `4.0`, то есть `3.8` или `3.9`, что часто встречается на `ubuntu 16.04.3`, то команда использует чуть другие параметры:
```bash
llvm-profdata merge -stats default.profraw -o default.profdata
```

После этого у вас должен получиться файл `default.profdata`. Его уже необходимо скормить на вход программе `llvm-cov` (в случае если у вас версия `4.0`):
```bash
llvm-cov show path_to_binary_or_lib \
-Xdemangler=c++filt \             # Демандглер имён функций. Не доступна на 3.9 и ниже
-instr-profile=default.profdata \ # Путь до файла с предыдущего шага.
-format=html \                    # В каком формате получить отчёт.
-output-dir=output_html           # В какаую директорию сгенерировать файлы отчёта.
```
После этого у вас в директории `output_html` должны находиться файлы отчёта о покрытии кода. Его удобно посмотреть с помощью браузера. Реализация функции покрытие кода с помощью clang активно развивается. Поэтому лучше всего использовать наиболее свежую версия clang/llvm из-за более удобных отчётов. Последняя доступная стабильная версия `5.0`.

# Примеры
--------------------------------
## JavaScriptCore
Для сборки JSC под Linux используется `cmake` (конфиги \*.cmake). Для сборки под macOS используется `xcodebuild` (конфиги \*.xcconfig). Репозиторий кода использующийся при сборке https://github.com/WebKit/WebKit. Если проекти не собирается и выскакивают какие-то ошибки, то можно глянуть билды которые собирались на https://webkit.org/nightly/archives/, а через них уже выйти на последний удачный коммит.

 В `Source/cmake/WebKitCompilerFlags.cmake` определено несколько макросов. Полезный для нас будет `WEBKIT_PREPEND_GLOBAL_COMPILER_FLAGS`, который добовляет указанные нами флаги к `CFLAGS` и `CXXFLAGS`. Добавляем в `Source/cmake/WebKitCompilerFlags.cmake` в секцию с `if (COMPILER_IS_GCC_OR_CLANG)` следующий код:
```cmake
WEBKIT_PREPEND_GLOBAL_COMPILER_FLAGS(-O0 -g -fprofile-instr-generate -fcoverage-mapping)
set(CMAKE_SHARED_LINKER_FLAGS "-fprofile-instr-generate -fcoverage-mapping ${CMAKE_SHARED_LINKER_FLAGS}")
```
После этого запускаем сборку проекта:
```bash
test@ubnt16043x64:~/Desktop/WebKit$ export CC=clang-5.0
test@ubnt16043x64:~/Desktop/WebKit$ export CXX=clang++-5.0
test@ubnt16043x64:~/Desktop/WebKit$ Tools/Scripts/build-webkit --jsc-only
```
Собранный проект находится в директории `WebKitBuild/Release`
```bash
test@ubnt16043x64:~/Desktop/WebKit$ llvm-profdata-5.0 merge -sparse default.profraw -o def.profdata
test@ubnt16043x64:~/Desktop/WebKit$ llvm-cov-5.0 show WebKitBuild/Release/bin/jsc -Xdemangler=c++filt  -instr-profile=def.profdata -format=html -output-dir=output_dir2
warning: 362 functions have mismatched data
```
После этого статистика в формате `html` будет доступна в директории `output_dir2`.

![Покрытие кода для WebKitBuild/Release/bin/jsc](/static/code-coverage/cc_3.png)

Если посмотреть более внимательно будет отсутствовать код JSC-движка. Для того, чтобы получить статистику по покрытию кода для JSC-движка необходимо указывать путь к библиотеке:
```bash
test@ubnt16043x64:~/Desktop/WebKit$ llvm-cov-5.0 show WebKitBuild/Release/lib/libJavaScriptCore.so.1.0.0 -Xdemangler=c++filt  -instr-profile=def.profdata -format=html -output-dir=output_dir
```

![Покрытие кода для libJavaScriptCore.so.1.0.0 1](/static/code-coverage/cc_1.png)

![Покрытие кода для libJavaScriptCore.so.1.0.0 2](/static/code-coverage/cc_2.png)

## ChakraCore
Проект можно слить с https://github.com/Microsoft/ChakraCore. Для того, чтобы добавить в проект опции покрытия кода необходимо отредактировать `CMakeFile.txt` в корне проекта и добавить следующие строки:
```cmake
set(CMAKE_C_FLAGS "-O0 -g -fprofile-instr-generate -fcoverage-mapping ${CMAKE_C_FLAGS}")
set(CMAKE_CXX_FLAGS "-O0 -g -fprofile-instr-generate -fcoverage-mapping  ${CMAKE_CXX_FLAGS}")
set(CMAKE_EXE_LINKER_FLAGS "-fprofile-instr-generate -fcoverage-mapping ${CMAKE_EXE_LINKER_FLAGS}")
```
Запуск сборки осуществляется следующим образом:
```bash
test@ubnt16043x64:~/Desktop/ChakraCore$ ./build.sh --cxx=/usr/bin/clang++-4.0 --cc=/usr/bin/clang-4.0 -n
```
Из-за того, что проект не собирался с помощью `clang` версии `5.0` пришлось использовать `4.0`.
Для генерирования статистики можно использовать `llvm-cov` и `llvm-profdata` версии `5.0`:
```bash
test@ubnt16043x64:~/Desktop/ChakraCore$ out/Release/ch test.js
test@ubnt16043x64:~/Desktop/ChakraCore$ llvm-profdata-5.0 merge -sparse default.profraw -o def.profdata
test@ubnt16043x64:~/Desktop/ChakraCore$ llvm-cov-5.0 show out/Release/ch -Xdemangler=c++filt  -instr-profile=def.profdata -format=html -output-dir=output_dir
warning: 37 functions have mismatched data
test@ubnt16043x64:~/Desktop/ChakraCore$ llvm-cov-5.0 show out/Release/libChakraCore.so -Xdemangler=c++filt  -instr-profile=def.profdata -format=html -output-dir=output_dir2
warning: 4 functions have mismatched data
```
Статистику для `ch` будет в директории `output_dir`. Для основного движка, или `libChakraCore.so`, в `output_dir2`.

