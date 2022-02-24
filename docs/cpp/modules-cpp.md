---
title: "Overview of modules in C++"
ms.date: 02/14/2022
helpviewer_keywords: ["modules [C++]", "modules [C++], overview"]
description: Modules in C++20 provide a modern alternative to header files.
---
# Overview of modules in C++

C++20 introduces *modules*, a modern solution for componentization of C++ libraries and programs. A module is a set of source code files that are compiled independently of the [translation units](https://wikipedia.org/wiki/Translation_unit_(programming)) that import them. Modules eliminate or reduce many of the problems associated with the use of header files. They often reduce compilation times. Macros, preprocessor directives, and non-exported names declared in a module aren't visible outside the module. They have no effect on the compilation of the translation unit that imports the module. You can import modules in any order without concern for macro redefinitions. Declarations in the importing translation unit don't participate in overload resolution or name lookup in the imported module. After a module is compiled once, the results are stored in a binary file that describes all the exported types, functions, and templates. That file can be processed much faster than a header file. And, the compiler can reuse it every place where the module is imported in a project.

Modules can be used side by side with header files. A C++ source file can import modules and also #include header files. In some cases, a header file can be imported as a module rather than textually #included by the preprocessor. We recommend you use modules in new projects rather than header files as much as possible. For larger existing projects under active development, experiment with converting legacy headers to modules. Base adoption on whether you get a meaningful reduction in compilation times.

## Enable modules in the Microsoft C++ compiler

Modules have had experimental support in the Microsoft C++ compiler for a long time. As of Visual Studio 2022 version 17.1, C++20 standard modules are fully implemented in the Microsoft C++ compiler. You can use the modules feature to create single-partition modules and to import the Standard Library modules provided by Microsoft. To enable support for Standard Library modules, compile with [`/experimental:module`](../build/reference/experimental-module.md) and [`/std:c++latest`](../build/reference/std-specify-language-standard-version.md). In a Visual Studio project, right-click the project node in **Solution Explorer** and choose **Properties**. Set the **Configuration** drop-down to **All Configurations**, then choose **Configuration Properties** > **C/C++** > **Language** > **Enable C++ Modules (experimental)**.

A module and the code that consumes it must be compiled with the same compiler options.

## Consume the C++ Standard Library as modules

Although not specified by the C++20 standard, Microsoft enables its implementation of the C++ Standard Library to be imported as modules. By importing the C++ Standard Library as modules rather than #including it through header files, you can potentially speed up compilation times depending on the size of your project. The library is componentized into the following modules:

- `std.regex` provides the content of header `<regex>`
- `std.filesystem` provides the content of header `<filesystem>`
- `std.memory` provides the content of header `<memory>`
- `std.threading` provides the contents of headers `<atomic>`, `<condition_variable>`, `<future>`, `<mutex>`, `<shared_mutex>`, and `<thread>`
- `std.core` provides everything else in the C++ Standard Library

To consume these modules, add an import declaration to the top of the source code file. For example:

```cpp
import std.core;
import std.regex;
```

To consume the Microsoft Standard Library module, compile your program with [`/EHsc`](../build/reference/eh-exception-handling-model.md) and [`/MD`](../build/reference/md-mt-ld-use-run-time-library.md) options.

## Basic example

The following example shows a simple module definition in a source file called *`Example.ixx`*. The *`.ixx`* extension is required for module interface files in Visual Studio. In this example, the interface file contains both the function definition and the declaration. However, the definitions can be also placed in one or more separate files (as shown in a later example). The `export module Example;` statement indicates that this file is the primary interface for a module called `Example`. The **`export`** modifier on `f()` indicates that this function is visible when `Example` is imported by another program or module. The module references a namespace `Example_NS`.

```cpp
export module Example;

#define ANSWER 42

namespace Example_NS
{
   int f_internal() {
        return ANSWER;
      }

   export int f() {
      return f_internal();
   }
}
```

The file *`MyProgram.cpp`* uses the **`import`** declaration to access the name that is exported by `Example`. The name `Example_NS` is visible here, but not all of its members. Also, the macro `ANSWER` isn't visible.

```cpp

import Example;
import std.core;

using namespace std;

int main()
{
   cout << "The result of f() is " << Example_NS::f() << endl; // 42
   // int i = Example_NS::f_internal(); // C2039
   // int j = ANSWER; //C2065
}

```

The `import` declaration can appear only at global scope.

## Implementing modules

You can create a module with a single interface file (*`.ixx`*) that exports names and includes implementations of all functions and types. You can also put the implementations in one or more separate implementation files, similar to how .h and .cpp files are used. The **`export`** keyword is used in the interface file only. An implementation file can **`import`** another module, but can't **`export`** any names. Implementation files may be named with any extension. An interface file and the backing set of implementation files are treated as a special variation of a translation unit called a *module unit*. A name that's declared in any implementation file is automatically visible in all other files within the same module unit.

For larger modules, you can split the module into multiple module units called *partitions*. Each partition consists of an interface file backed by one or more implementation files.

## Modules, namespaces, and argument-dependent lookup

The rules for namespaces in modules are the same as in any other code. If a declaration within a namespace is exported, the enclosing namespace (excluding non-exported members) is also implicitly exported. If a namespace is explicitly exported, all declarations within that namespace definition are exported.

When it does argument-dependent lookup for overload resolutions in the importing translation unit, the compiler considers functions declared in the same translation unit (including module interfaces) as where the type of the function's arguments are defined.

### Module partitions

A module can be componentized into *partitions*, each consisting of an interface file and zero or more implementation files. A module partition is similar to a module, except it shares ownership of all declarations in the entire module. All names exported by partition interface files are imported and re-exported by the primary interface file. A partition's name must begin with the module name followed by a colon. Declarations in any of the partitions are visible within the entire module. No special precautions are needed to avoid one-definition-rule (ODR) errors. You can declare a name (function, class, and so on) in one partition and define it in another. A partition implementation file begins like this:

```cpp
module Example:part1;
```

The partition interface file begins like this:

```cpp
export module Example:part1;
```

To access declarations in another partition, a partition must import it, but it can only use the partition name, not the module name:

```cpp
module Example:part2;
import :part1;
```

The primary interface unit must import and re-export all of the module's interface partition files like this:

```cpp
export import :part1;
export import :part2;
...
```

The primary interface unit can import partition implementation files, but can't export them. Those files aren't allowed to export any names. This restriction enables a module to keep implementation details internal to the module.

## Modules and header files

You can include header files in a module source file by putting the `#include` directive before the module declaration. These files are considered to be in the *global module fragment*. A module can only see the names in the global module fragment that are in headers it explicitly includes. The global module fragment only contains symbols that are used.

```cpp
// MyModuleA.cpp

#include "customlib.h"
#include "anotherlib.h"

import std.core;
import MyModuleB;

//... rest of file
```

You can use a traditional header file to control which modules are imported:

```cpp
// MyProgram.h
import std.core;
#ifdef DEBUG_LOGGING
import std.filesystem;
#endif
```

### Imported header files

Some headers are sufficiently self-contained that they can be brought in using the **`import`** keyword. The main difference between an imported header and an imported module is that any preprocessor definitions in the header are visible in the importing program immediately after the `import` statement. However, preprocessor definitions in any files included by that header aren't visible.

```cpp
import <vector>;
import "myheader.h";
```

## See also

[`module`, `import`, `export`](import-export-module.md)
