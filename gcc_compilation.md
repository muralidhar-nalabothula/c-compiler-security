## GCC (11)

Understanding GCC flags is a *pain*. Which are enabled by `Wall` or `Wextra` is
not very easy to untangle.
The most reliable way is to parse and analyze the `commont.opt` and `c.opt`
files, which define (partially) the command line options which are supported by GCC.

The format is decribed in the GCC internals
[manual](https://gcc.gnu.org/onlinedocs/gccint/Option-file-format.html#Option-file-format),
so I've written a partial [parser][./gcc_copt_inclusions.py] which can help
identify what flags are needed.
You *should* also check the
[compiler-warnings](https://github.com/pkolbus/compiler-warnings) project, which has a real parser
for GCC, Clang and XCode.

### Warnings

Note that some warnings **depend** on some optimizations to be enabled to be
effective, so I recommend to always use `-O2`.

#### Generic

* `-Wall`: enable "most" of warnings by default
* `-Wextra`: enable *more* warnings by default
* `-Wpedantic`: and even more.

#### Security warnings

* `-Wformat=2`: check for format string problems
* `-Wformat-overflow=2`: check for *printf overflow
* `-Wformat-truncation=2`: check for *nprintf potential truncation
* `-Wformat-security`: check for dangerous format specifiers in *printf (enabled by `-Wformat=2`)
* `-Wnull-dereference`: Warn if dereferencing a NULL pointer may lead to erroneous or undefined behavior
* `-Wstack-protector`: Warn when not issuing stack smashing protection for some reason
* `-Wstrict-overflow=3`
* `-Wtrampolines`: Warn whenever a trampoline is generated (will probably create an executable stack)
* `-Walloca` or `-Walloca-larger-than=1048576`: don't use `alloca()`, or limit it to "small" sizes
* `-Wvla` or `-Wvla-larger-than=1048576`: don't use variable length arrays, or limit them to "small" sizes
* `-Warray-bounds=2`: Warn if an array is accessed out of bounds. Note that it is very limited and will not catch some cases which may seem obvious.
* `-Wimplicit-fallthrough=3`: already added by `-Wextra`, but mentioned for reference.
* `-Wtraditional-conversion`: Warn of prototypes causing type conversions different from what would happen in the absence of prototype.
* `-Wshift-overflow=2`: Warn if left shift of a signed value overflows.
* `-Wcast-qual`: Warn about casts which discard qualifiers.
* `-Wstringop-overflow=4`: Under the control of Object Size type, warn about buffer overflow in stringmanipulation functions like memcpy and strcpy. 
* `-Wconversion`: Warn for implicit type conversions that may change a value. *Note*: will probably introduce lots of warnings.
* `-Warith-conversion`: Warn if conversion of the result of arithmetic might change the value even though converting the operands cannot. *Note*: will probably introduce lots of warnings.

Those are not really security options per se, but will catch some logical errors:

* `-Wlogical-op`: Warn when a logical operator is suspiciously always evaluating to true or false.
* `-Wduplicated-cond`: Warn about duplicated conditions in an if-else-if chain.
* `-Wduplicated-branches`: Warn about duplicated branches in if-else statements.

#### Extra flags

* `-Wformat-signedness`: Warn about sign differences with format functions.
* `-Wshadow`: Warn when one variable shadows another.  Same as `-Wshadow=global`
* `-Wstrict-overflow=4` (or 5)
* `-Wundef`: Warn if an undefined macro is used in an `#if` directive
* `-Wstrict-prototypes`: Warn about unprototyped function declarations
* `-Wswitch-default`: Warn about enumerated switches missing a `default:` statement.
* `-Wswitch-enum`: Warn about all enumerated switches missing a specific case.
* `-Wstack-usage=<byte-size>`: Warn if stack usage might exceed `<byte-size>`.
* `-Wcast-align=strict`: Warn about pointer casts which increase alignment.

### Compilation flags

* `-fstack-protector-strong`: add stack cookie checks to functions with stack buffers or pointers
* `-fstack-clash-protection`: Insert code to probe each page of stack space as it is allocated to protect from stack-clash style attacks.
* `-fPIE`: generate position-independant code (needed for ASLR)

### Code analysis

GCC 10 [introduced](https://developers.redhat.com/blog/2020/03/26/static-analysis-in-gcc-10)
the `-fanalyzer` static code analysis tool, which was vastly [improved](https://developers.redhat.com/blog/2021/01/28/static-analysis-updates-in-gcc-11) in GCC 11.

It tries to detect memory management issues (double free, use after free,
etc.), pointers-related problems, etc.

It *is* costly and slows down compilation and also exhibits false positives, so
its use may not always be practical.

### Runtime sanitizers

GCC supports various *runtime* sanitizers, which are enabled by the `-fsanitize` flags, which are often not compatible and thus must be run separately.

* `address`: AddressSanitizer, with extra options available:
    * `pointer-compare `: Instrument comparison operation with pointer operands. Must be enabled at runtime by using `detect_invalid_pointer_pairs=2` in the `ASAN_OPTIONS` env var.
    * `pointer-substract`: Instrument subtraction with pointer operands. Must be enabled at runtime by using `detect_invalid_pointer_pairs=2` in the `ASAN_OPTIONS` env var.
    * `ASAN_OPTIONS=strict_string_checks=1:detect_stack_use_after_return=1:check_initialization_order=1:strict_init_order=1`
* `thread`: ThreadSanitizer, a data race detector.
* `leak`: memory leak detector for programs which override `malloc` and other allocators.
* `undefined`: UndefinedBehaviorSanitizer. Checks not enabled by default (GCC 11):
    * `-fsanitize=bounds-strict`
    * `-fsanitize=float-divide-by-zero`
    * `-fsanitize=float-cast-overflow`

`kernel-address` also exists and enables AddressSanitizer for the Linux kernel.

#### Use with fuzzing

Runtime sanitizers are particularly useful when:

* running test suites
* fuzzing code

as they may uncover runtime errors which would not necessarily trigger a crash.

### References
* <https://developers.redhat.com/blog/2020/03/26/static-analysis-in-gcc-10>
* <https://developers.redhat.com/blog/2021/01/28/static-analysis-updates-in-gcc-11>
* <https://developers.redhat.com/blog/2017/02/22/memory-error-detection-using-gcc>
* <https://codeforces.com/blog/entry/15547>
* <https://github.com/google/sanitizers/wiki/AddressSanitizerFlags>