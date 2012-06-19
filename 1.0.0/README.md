ROSE Milestones - 1.0.0
=======================

## Contacts

@doubleotoo, @tristanvdb, @chunhualiao


## Design a new `SageInterface` and `SageBuilder` API


## Problem

1. **`SageInterface` API is not extremely usable or complete**

    The current `SageInterface` [namespace](https://github.com/rose--compiler/rose/tree/master/src/frontend/SageIII/sageInterface) and [utility functions]  (http://rosecompiler.org/ROSE_HTML_Reference/group__frontendSageUtilityFunctions.html) are not widely known or used.   Also, existing APIs are complex and, therefore, not user-friendly.

2. **`SageBuilder` API is not well-documented or demonstrated**

    The current `SageBuilder` interface simply lacks good examples (tutorials, recipes, FAQs, etc.)

## Solution

1. **Design a new `SageInterface` API**
    * providing **additional convenience functions** (wrappers over the low-level ROSE API) to aid users in using ROSE  effectively to accomplish common "tasks" (AST manipulations, analysis, building, ...)
    * adding **unit tests** to ensure the quality and stability of the API
    * **documenting** the API in depth through **Doxygen** and **user-level tutorials**
    * initial **refactoring** of core code to be `DESTDIR` conformant (e.g. `$ROSE_INSTALL/include/rose/midend`)

2. **Document and extend the existing `SageBuilder` API**

---

*A typical developer's workflow:*

![SageBuilder/Interface workflow](http://www.dabbleboard.com/image?b=doubleotoo&i=0&c=d7533050b2c7575c9b1bffdaf9ac90&t=.png)

1. Analyze the AST
2. According to the analysis in (1), perform AST transformations
3. Transformations usually involve the `SageBuilder`


## API Design

* [How to Design a Good API](https://github.com/rose-compiler/rose-docs/blob/master/good_api_design.md)

---

**Modularity**: logical grouping of utilities

``` C++
  namespace SageInterface {
        namespace Array {
            // SgArray convenience API
        }

        namespace Expression {
            // SgExpression convenience API
        }

        ...
  }
```

## Features

#### Assertions / Exceptions

`ROSE_ASSERT`:

```bash
$ grep -rn "#define ROSE_ASSERT" $ROSE/src
../../ROSE/src/frontend/SageIII/preproc-c.ll:537:#define ROSE_ASSERT assert
../../ROSE/src/frontend/CxxFrontend/EDG/EDG_3.3/edgFrontEndWithoutSage/edgFrontEnd.c:27:#define ROSE_ASSERT assert
../../ROSE/src/frontend/CxxFrontend/EDG/EDG_SAGE_Connection/debugging.C:47:#define ROSE_ASSERT(x) assert(x)
../../ROSE/src/util/processSupport.h:15:#define ROSE_ASSERT assert
../../ROSE/src/util/processSupport.h:22:  #define ROSE_ASSERT assert
../../ROSE/src/util/processSupport.h:24:  #define ROSE_ASSERT(exp) do {if (__builtin_constant_p(exp)) {if (exp) {} else (std::abort)();}} while (0)
../../ROSE/src/ROSETTA/src/ROSETTA_macros.h:137:#define ROSE_ASSERT assert
../../ROSE/src/ROSETTA/src/ROSETTA_macros.h:139:#define ROSE_ASSERT assert
```

1. `assert(false)`: should not exist; hard fail instead; assert=*should be here, but do a check*; shouldn't even exist because an assertion should have been performed beforehand to detect the reason why we need to fail now.
* Logging: we want to print messages (log)

2. `assert(<condition>, <message>)`: make assertions more meaningful

3. **Stack-trace**: instead of having to use GDB; Logging can accomplish a similar effect; LOG_MACROS to allow disabling of logging (i.e. redefine macro to be empty)


`assert(a && b)` should be broken into two in order to figure out which failed, `a` or `b`

Not important:
* assertions: help to debug in development; condition could be expensive;

**Logging**

`Log::warn(<msg>)`

**Exceptions / Assertions / Failures **
* production vs development
* `FrontEndException::assert(<condition>, <message>)`
* `FrontEndException::fail(<message>)`

  * Modularity: disable / enable certain exception assertions, e.g. FrontendException

* NO ASSERTIONS
* ROSE_LOG: stream; `rose::log << indent(4) << "something" << endl;`
* `ROSE_FAIL(<printf style format string>)`: hard fail instead of nasty assertions everywhere
   * using useful messages that indicate the reason for failure; production => exception

**Namespaces**
* Add namespaces to everything!

  * `SageIII::SgProject`; existing core code will have `using namespace SageIII` for backward compatibility

``` C++
  int main(int argc, char ** argv) {
    SgProject * project = SageInterface::frontend(argc, argv);
    return SageInterface::backend(project);
  }

  int main(int argc, char ** argv) {
    rose::core::Sage::SgProject * project = rose::core::SageBuilder::buildProject();
    rose::core::Sage::SgFile * file = rose::core::SageBuilder::buildEmptyFile("file_name");

    // Fill the file

    return rose::core::SageInterface::backend(project);
  }
```
Judgement call: `rose::core::SageBuilder::Support::buildEmptyFile("file_name");`

---

``` C++
  SageInterface::Array::get_dimension_info(...)
```


## Misc. Concepts

[Dynamic Programming (Wikipedia)](http://en.wikipedia.org/wiki/Dynamic_programming)

> The key idea behind dynamic programming is quite simple...we need to solve different parts of the problem (subproblems), then combine the solutions of the subproblems to reach an overall solution.  
Top-down dynamic programming simply means storing the results of certain calculations, which are later used again since the completed calculation is a sub-problem of a larger calculation. [e.g. memoization]

*Also see: [call-by-need](http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_need)*

---

[Functional (declarative) Programming (Wikipedia)](http://en.wikipedia.org/wiki/Functional_programming)

> [T]reats computation as the evaluation of mathematical functions and avoids state and mutable data.

*Also see: [Imperative programming](http://en.wikipedia.org/wiki/Imperative_programming)*

> [C]omputation in terms of statements that change a program state



## Misc. Notes

* SageInterface::Configuration: e.g. READ_ONLY

``` C++

  SageInterface {
      Array {
          struct TypeInfo {
              SgType*  base_type;
              vector<Array::Dimension>  dimensions;
          }

          struct Dimension {
              SgExp* original_expression;
              bool is_constant;
              SgValueExp* value;              
          }

          struct Subscript {
              int value;
          }

          Constant {
              int[] get_dimensions(SgArrayRef*) { ... }
              int[] get_dimensions(SgArrayType*) { ... }
          }

          // SageInterface::querySubTree
          std::vector<SgArray*> all(SgBlock*,

              int[] get_subscripts(SgArrayRef*) { ... }
              int[] get_subscripts(SgArrayType*) { ... }

          std::vector<...> get_dimensions(SgArrayType*) { ... }
          ... get_all_in_scope(SgArrayType*) { ... }
      }
   }

void foo (SgArrayType* node) {
    TypeInfo info = get_type_info(node);

    info.dimensions.each |d|

      // Method 1
      if d.is_constant
         ...
      end

      // Method 2
      SgValueExp * val;
      if SageInterface::Expression::isConstant(d.original_expression, &val)
         d.value
      end
    end
}
```
