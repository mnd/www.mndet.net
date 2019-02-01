---
layout: default
title: Custom Warnings for C Code
---
As a part of my work for [Inango Systems](https://www.inango.com/) I investigated possibility to create custom warnings for C code and prepared this article.

# Motivation

We had a task to investigate a big amount of a C code in order to estimate how many issues we can face in future work with this code. This task had the following properties:

1. It was a huge amount of code. And it was not possible to investigate it manually or semi-automatically. All the statistics should be gathered automatically.
2. Big part of this code was obsolete and was not compiled at all. This included two cases: there were files that were not compiled and there was a code in the compiled files that was disabled with preprocessor directives. Our estimations should have been only for the active code, i.e. the one, that is currently compiled and used. 
3. It was not possible to create a run-time test with a full (or at least a big enough) code coverage. All our estimations should have been made at the compile time.

In order to solve this task we decided to add custom compile time warnings through system headers. Only when the work was finished, we've discovered that the GNU Libc widely uses the same technique for a `-D_FORTIFY_SOURCE` support.

# Implementation Technique

There is a broadly known technique to stop a compilation on any of the compile-time conditions:

    // Starting from C11 there is _Static_assert construction
    #define non_zero_constant_argument(x) \
      _Static_assert(x, "Argument should be not zero")
    // Before C11 you can declare an impossible array with a constant integer expression
    #defime non_zero_constant_argument_pre_C11(x) \
      char static_assert_argument_should_be_not_zero[(x) != 0 ? 1 : -1]

But there is no standard technique to add a compilation warning instead of a compilation error. Luckily, there are compiler-specific ways to generate warnings instead of errors. 

## GCC Compilation Warnings 

Our environment was designed to work with the GCC compiler and we worked only with the GCC to implement custom warnings. The GCC among other things [function attributes](https://gcc.gnu.org/onlinedocs/gcc/Function-Attributes.html) supports the next attributes that are suitable to emit warnings during the compilation:
    
 1. `void deprected_function() __attribute__((deprecated("custom message")));` will emit a compilation warning on each `deprecated_function` usage in a code, except function declarations.

        void __attribute__((deprecated("custom message"))) warning() { }
        int main () {
          // warning: ‘warning’ is deprecated: custom message [-Wdeprecated-declarations]
          warning();
        }

   Unfortunately, this attribute does not allow to add any conditions for warnings.
 2. `void warning_if_false(void *arg) __attribute__((nonnull(1)));` will emit a warning if an integer constant expression in the first argument is calculated as zero:

        void __attribute__((nonnull(1))) warning_if_false(void * arg) { }
        int main () {
          // warning: null argument where non-null required (argument 1) [-Wnonnull]
          warning_if_false((void *)(sizeof(long) == 7));
        }

   Such conditions allow us to define conditional warnings, but it does not provide the possibility to emit any message related to this warning.
 3. `void warning_if_true(int unused, ...) __attribute__((sentinel));` will emit a warning, if an integer constant expression in the last argument is calculated as non-zero value:

        void __attribute__((sentinel)) warning_if_true(int unused, ...) { }
        int main () {
          // warning: missing sentinel in function call [-Wformat=]
          warning_if_true(0, (void *)(sizeof(long) != 7));
        }

   This solution in the same manner as the previous one does not allow to write a custom warning message.
 4. `int warning() __attribute__((warn_unused_result));` will emit a warning if the function was compiled and the function result was not used in any way:

        int __attribute__((warn_unused_result)) warning() { return 0; }
        int main () {
          // warning: ignoring return value of ‘warning’, declared with attribute warn_unused_result [-Wunused-result]
          (void)((sizeof(long) != 7) ? warning() : 0);
        }

    This attribute allows to emit warnings based on a condition, but if we want to provide a custom warning message, we should embed this message into the function name:

        int __attribute__((warn_unused_result)) warning_64bit_support_is_unstable() { return 0; }
        // warning: ignoring return value of ‘warning_64bit_support_is_unstable’, declared with attribute warn_unused_result [-Wunused-result]
        (void)((sizeof(void *) == 8) ? warning_64bit_support_is_unstable() : 0);
        
 5. `int warning() __attribute__((warning("custom message")));` will emit a warning with a custom message if a call to such a function is not eliminated through the dead code elimination or other optimizations:
 
        void __attribute__((warning("custom message"))) warning() { }
        int main () {
          // warning: call to ‘warning’ declared with attribute warning: custom message
          (void)((sizeof(void *) != 7) ? warning() : 0);
        }

    This attribute allows both custom messages and compile-time conditions. And even better: this attribute emits a warning after the dead code elimination and we can use this warning-attributed function both in a conditional block and as integer constant expressions. Consider the following code to understand what benefits it give to us:

        void __attribute__((warning("..."))) warning1() { }
        int __attribute__((warn_unused_result)) warning2() { return 0; }
        if (sizeof(void *) == 7) {
          // Will not emit a warning: dead code elimination will remove if block
          warning1();
          // Will emit a warning: return value ignored
          warning2();
        }
        // Will not emit a warning: warning2 will be removed on GCC constant folding
        (void) ((sizeof(void *) == 7) ? warning2() : 0);

## Obtaining Usefull Compile-Time Information

First of all, in order to use this warnings technique for your code, you should have wrappers for all your APIs:

    library.c:
    void __null_warning() { }
    int __internal_do_stuff(int *arg) {
      // do usefull stuff
      return 0;
    }

    library.h:
    extern void __null_warning() __attribute__((warning("function called with NULL argument")))
    extern int __internal_do_stuff(int *arg);

    // A function can be wrapped with an inlined function to make sure that the GCC will propagate the argument values.
    // Inline qualification does not guarantee that the function will be inlined; use an attribute additionally.
    static inline int __attribute__((always_inline)) do_stuff(int *arg) {
      if (arg == NULL) {
        __null_warning();
      }
      return __internal_do_stuff(arg);
    }

    // OR a function can be wrapped with a macro with the GCC Compound Expression.
    // https://gcc.gnu.org/onlinedocs/gcc/Statement-Exprs.html
    #define do_stuff(arg) ({    \
      if (arg == NULL) {        \
        __null_warning();       \
      }                         \
      __internal_do_stuff(arg); \
    })

Suggested code will emit a warning on every `do_stuff(0)` call. Unfortunately, this code will also emit warnings for the calls like `do_stuff(non_constant_argument)`. In order to create more useful warnings you should consider to use additional GCC built-in functions from the [https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html](https://gcc.gnu.org/onlinedocs/gcc/Object-Size-Checking.html) and the (https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)[https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html].

This builtins allowed us to emit the required information from the compiler:

    #define do_stuff(arg) ({                                                  \
      if (__builtin_constant_p(arg) && arg == NULL) {                         \
        __called_with_a_null_constant_warning();                              \
      }                                                                       \
      if (!__builtin_types_compatible_p(typeof(arg), int *)) {                \
        __called_with_an_int_pointer();                                       \
      }                                                                       \
      if (__builtin_object_size(arg, 0) == -1) {                              \
        __called_with_a_pointer_to_an_unknown_object_warning();               \
      }                                                                       \
      if (__builtin_object_size(arg, 0) != -1                                 \
          && __builtin_object_size(arg, 0) < 20 ) {                           \
        __called_with_a_pointer_to_an_object_smaller_then_20_bytes_warning(); \
      }                                                                       \
      if (!__builtin_types_compatible_p(typeof(&((arg)[0])), typeof(arg))) {  \
        __called_with_an_array_argument_warning();                            \
      }                                                                       \
      __internal_do_stuff(arg);                                               \
    })
    
This technique can be adjusted for any purposes: the GNU Libc used this technique to find the memory bounds violations in the string functions when compiled with the `-D_FORTIFY_SOURCE=1`, this technique can be used to warn about a non-recommended API usage (e.g. "in next versions calls with NULL argument will be forbidden").

## Issues and Workarounds

 1. To use the `__attribute__((warning("message")))` you should ensure that the attributed function will not be optimized from the code. This can be achieved in two ways:

      - You can place a definition of `warning` function to a library and call it from a library header:

            library.c:
            void library_warning() { }
            int __internal_do_stuff(int *arg) {
              // do usefull stuff
              return 0;
            }

            library.h:
            extern void library_warning() __attribute__((warning("Warning message")));
            extern int __internal_do_stuff(int *arg);
            #define do_stuff(arg) ({ \
              library_warning();     \
              __internal_do_stuff(); \
            })

        This solution is used by the GNU Libc for the `__warn_memset_zero_len` diagnostic function.

      - Or you can disable the optimization of a warning function directly in the header file with another GCC attribute:

            library.c:
            int __internal_do_stuff(int *arg) {
              // do usefull stuff
              return 0;
            }

            library.h:
            static inline void __attribute__((warning("Warning message"), optimize("O0"))) library_warning() {  }
            extern int __internal_do_stuff(int *arg);
            #define do_stuff(arg) ({ \
              library_warning();     \
              __internal_do_stuff(); \
            })

        For Inango code we have selected this solution.
 2. With the code described above all your warnings will suddenly disappear once you install your `library.h` header to the `/usr/include` directory. This happens because the GCC suppresses all warnings that were found in the code of system headers. You can bring back your warnings in two ways:

      - If you use a function wrapper you can use the "artificial" GCC attribute:

            library.c:
            void library_warning() { }
            int __internal_do_stuff(int *arg) {
              // do usefull stuff
              return 0;
            }

            library.h:
            extern void library_warning() __attribute__((warning("Warning message")));
            extern int __internal_do_stuff(int *arg);
            static inline void __attribute__ ((always_inline, artificial)) do_stuff() {
              library_warning();
              __internal_do_stuff();
            }

        This solution used by the GNU Libc.
      - If you use a macro wrapper you should enable "-Wsystem-headers" warnings for your code:

            library.c:
            int __internal_do_stuff(int *arg) {
              // do usefull stuff
              return 0;
            }

            library.h:
            static inline void __attribute__((warning("Warning message"), optimize("O0"))) library_warning() {  }
            extern int __internal_do_stuff(int *arg);
            #pragma GCC diagnostic push
            #pragma GCC diagnostic warning "-Wsystem-headers"
            #define do_stuff(arg) ({ \
              library_warning();     \
              __internal_do_stuff(); \
            })
            #pragma GCC diagnostic pop

 3. The solution with warnings will conflict with the usage of the "-Werror" compilation flag. As on December 2018 all the GCC versions will fail the compilation with an error if the `__attribute__((warning("...")))` code will be compiled with the "-Werror" flag. There is no way to change this behavior. The possible solutions to this issue:

      - You can change a code to use the `__attribute__((warn_unused_result))` instead: 

            library.c:
            int __internal_do_stuff(int *arg) {
              // do usefull stuff
              return 0;
            }

            library.h:
            // this pragma should be global to forcely move this diagnostics from errors to warnings
            #pragma GCC diagnostic warning "-Wunused-result"
            #pragma GCC diagnostic push
            #pragma GCC diagnostic warning "-Wsystem-headers"
            static inline void __attribute__((warn_unused_result, optimize("O0"))) library_warning() {  }
            extern int __internal_do_stuff(int *arg);
            #define do_stuff(arg) ({ \
              (void)((integer_constant_expression_condition) ? library_warning() : 0);     \
              __internal_do_stuff(); \
            })
            #pragma GCC diagnostic pop

      - Or you can wait for the next GCC upstream release where we added the solution for the `__attribute__((warning("")))` itself and use a new "-Wno-error=attribute-warning" that we submitted to the GCC as a result of the described work.
      
## Final Code Template

Finally, let's combine all the described work together and show how the custom warnings can be created for a GCC-based code:

            library.c:
            int __internal_do_stuff(int *arg) {
              // do usefull stuff
              return 0;
            }

            library.h:
            #ifndef LIBRARY_H
            #define LIBRARY_H
            
            /* Interpret all the "warning" attributes in a code compiled 
               with this library as warnings, regardless of -Werror or
               -Wno-attribute-warning flags.
               As on Dec 2018, this flag is supported only in GCC trunk */
            #pragma GCC diagnostic warning "-Wattribute-warning"

            /* Declare the internal functions that we will wrap */
            extern int __internal_do_stuff(int *arg);

            /* Create the warning functions with a specific messages.
               Make sure that these functions would not be optimized out from the code.
               Define function as a static to not export the name from the resulting binaries. */
            static inline void __attribute__((warning("Warning message"), optimize("O0"))) library_warning() {  }

            /* Print this warnings even if the library.h is installed to the system headers directory.
               This behavior will be applied only to library.h code.
               Other system headers will not print the warnings */
            #pragma GCC diagnostic push
            #pragma GCC diagnostic warning "-Wsystem-headers"

            /* Wrap all the functions in the macro definitions with the GCC Compound expressions inside.
               Make sure that all the `if' conditions are integer constant expressions. Otherwise,
               there is no guarantees that the compiler will be able to otimize out the failed checks. */
            #define do_stuff(arg) ({                       \
              if (integer_constant_expression_condition) { \
                library_warning();                         \
              }                                            \
              __internal_do_stuff();                       \
            })
            
            #pragma GCC diagnostic pop
            
            #endif /* LIBRARY_H */

# Clang Based Solution

There is an alternate solution for the described issue supported in the Clang compiler. In order to emit compile-time warnings in Clang, you can just specify the `diagnose-if` [attribute](https://clang.llvm.org/docs/AttributeReference.html#diagnose-if) on the relevant function:

    library.h:
    int do_stuff(int *arg) {
      // do usefull stuff
      return 0;
    }

    library.h:
    int do_stuff(int *arg) __attribute__((diagnose_if(arg == NULL, "arg is NULL", "warning")));
    int do_stuff(int *arg) 
      __attribute__((diagnose_if(__builtin_object_size(arg, 0) == -1, "arg of unknown size", "warning")));

With the Clang `__attribute__((diagnose_if(...)))` there is no need in additional wrappers (you still should create a wrapper if you want to extract the information about the original type of the argument) and new warnings can be simply added with additional declaration lines. 

But the GNU GCC is still the main compiler in GNU/Linux systems and we think that the proposed article will be usable for all the GCC users. 
