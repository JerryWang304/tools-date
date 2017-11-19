SAFL Fuzzer
==================

## Install
SAFL is installed in /opt/tsmart-date/. Please configure the `PATH` environment variable in your shell first.

Execute in shell:

```sh
export PATH="/opt/tsmart-date:$PATH"
```

Or you can directly insert the content above in configuration files such as `~/.bashrc`.

## Symbolic execution

### Configure and Build
You can use SAFL after the installation. SAFL uses symbolic execution to build high-quality seeds as the input for fuzzer, thus the first step is to configure the size of the input. For example, execute the following command when you want 10 bytes:


```sh
~/test/c-ares root@HghDTG
❯ date-prepare sym 10
[*] Toolchain is at /tmp/toolchain
[+] Saving configuration (symbolic, size is 10)
```

Next, let's configure the project before build. Just like normal configuration procedure, you only need to insert `safl-configure` before the command.

```sh
~/test/c-ares/src heads/master* root@HghDTG
❯ date-configure ./configure
[*] Config: sym 10
[+] Setting up bitcode toolchain (unfiltered)
[*] LLVM is at /opt/llvm
[*] Toolchain is at /tmp/toolchain
[*] Additional argument(s) to pass:
[*]     CXXFLAGS=-stdlib=libc++ -nostdinc -I/opt/llvm/include/c++/v1 -Qunused-arguments
[*]     LDFLAGS=-stdlib=libc++ -nostdinc -L/opt/llvm/lib -Wl,-rpath,/opt/llvm/lib
[*]     CC=/tmp/toolchain/clang
[*]     CXX=/tmp/toolchain/clang++
[+] Running target program
checking whether to enable maintainer-specific portions of Makefiles... no
[...]
config.status: executing libtool commands
configure: amending ./Makefile
```

The next step is compilation. Just like normal configuration procedure, you need to insert `safl-build` before the command.


```sh
❯ date-build make -j
[*] Config: sym 10
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] Custom build comand, ignoring
[+] Running target program
make  all-recursive
[...]
make[1]: Leaving directory '/home/hugh/test/c-ares/src'
```

After building the library file, you may need to compiler additional test harness. Just like the normal `g++` or `clang` invocation, but you need to insert `safl-build` before the command.

```sh
~/test/c-ares root@HghDTG
❯ date-build g++ target.cc -I src -c
[*] Config: sym 10
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] C++ compiler intercepted
[*] Command to execute:
[*]     /tmp/toolchain/clang++
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -I/opt/llvm/include/c++/v1
[*]     -Qunused-arguments
[*]     target.cc
[*]     -I
[*]     src
[*]     -c
```

Finally, let's link the object files together! Caveat: SAFL uses a customized driver instead of the built-in version of clang, so you can't link with the standard toolchain. You should use `date-link` instead.

The following example links the test harness `target.o` and library `src/.libs/libcares.a` together into binary `app`, and output the LLVM bitcode file `app.bc`.

```sh
~/test/c-ares root@HghDTG 
❯ date-link target.o src/.libs/libcares.a
[*] Config: sym 10
[+] Setting up bitcode toolchain
[*] WLLVM is at /opt/wllvm
[*] Toolchain is at /tmp/toolchain
[+] Generating driver, symbolic argument size = 10
[*] Command to execute:
[*]     /tmp/toolchain/clang
[*]     -xc
[*]     -c
[*]     -o
[*]     /tmp/toolchain/driver.o
[*]     /tmp/toolchain/driver.c
[+] Linking objects
[*] Command to execute:
[*]     /tmp/toolchain/clang++
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -I/opt/llvm/include/c++/v1
[*]     -Qunused-arguments
[*]     -stdlib=libc++
[*]     -nostdinc++
[*]     -L/opt/llvm/lib
[*]     -Wl,-rpath,/opt/llvm/lib
[*]     /tmp/toolchain/driver.o
[*]     target.o
[*]     src/.libs/libcares.a
[*]     -o
[*]     app
[+] Extracting bitcode
Loading '/tmp/toolchain/.driver.o.bc'
Loading '/home/hugh/test/c-ares/.target.o.bc'
Linking in '/home/hugh/test/c-ares/.target.o.bc'
Loading '/home/hugh/test/c-ares/src/.libcares_la-ares_create_query.o.bc'
Linking in '/home/hugh/test/c-ares/src/.libcares_la-ares_create_query.o.bc'
Loading '/home/hugh/test/c-ares/src/.libcares_la-ares_library_init.o.bc'
Linking in '/home/hugh/test/c-ares/src/.libcares_la-ares_library_init.o.bc'
Writing bitcode...
```

### Start Symbolic execution
Create a new directory for intermediate files for symbolic execution, then place the linked bitcode here. For example, the following command uses `sym` as the directory.

```sh
~/test/c-ares root@HghDTG
❯ mkdir sym && cd sym

~/test/c-ares/sym root@HghDTG
❯ cp ../app.bc .
```

Use command `safl-fuzz` to start symbolic execution. To get better results, you may re-run it several times.

```sh
~/test/c-ares/sym root@HghDTG
❯ date-fuzz app
[*] Config: sym 10
[*] Symbolic argument size = 10
[*] Application is at app.bc
[+] Starting symbolic execution
[*] KLEE cmdline:
[*]     /opt/klee/bin/klee
[*]     -only-output-states-covering-new
[*]     -libc=uclibc
[*]     -posix-runtime
[*]     -simplify-sym-indices
[*]     app.bc
[*]     -sym-arg
[*]     10
KLEE: NOTE: Using klee-uclibc : /opt/klee/lib64/klee/runtime/klee-uclibc.bca
KLEE: NOTE: Using model: /opt/klee/lib64/klee/runtime/libkleeRuntimePOSIX.bca
KLEE: output directory is "/home/hugh/test/c-ares/sym/klee-out-0"
KLEE: Using Z3 solver backend
[...]
^CKLEE: ctrl-c detected, requesting interpreter to halt.
KLEE: halting execution, dumping remaining states

KLEE: done: total instructions = 224686
KLEE: done: completed paths = 1371
KLEE: done: generated tests = 6
```

### Generate seeds
Symbolic execution outputs testcases. You need to use `date-test-gen` to convert the testcases into seeds that can be used in fuzzing. The following example generates 6 seeds in `seed` directory.

```sh
~/test/c-ares/sym root@HghDTG
❯ ls
klee-out-0  app.bc  klee-last

~/test/c-ares/sym root@HghDTG
❯ mkdir seed

~/test/c-ares/sym root@HghDTG
❯ date-test-gen klee-out-*/*.ktest --output seed

~/test/c-ares/sym root@HghDTG
❯ ls seed
test000001.symseed  test000003.symseed  test000005.symseed
test000002.symseed  test000004.symseed  test000006.symseed
```

These seeds can be used as inputs for fuzzing.

## Fuzz

### Configure and build
Before fuzzing, please cleanup the build directory and delete the compiled object files and binaries.

```sh
~/test/c-ares/src heads/master* root@HghDTG
❯ make clean
make[1]: Entering directory '/home/hugh/test/c-ares/src'
[...]
rm -f *.o
rm -f *.lo
make[1]: Leaving directory '/home/hugh/test/c-ares/src'
```

Next, reconfigure SAFL to switch to dynamic mode.

```sh
~/test/c-ares/src heads/master* root@HghDTG
❯ date-prepare dyn
[*] Toolchain is at /tmp/toolchain
[+] Saving configuration (dynamic)
```

Execute the same commands as the aforementioned build steps:

```sh
# Build library
date-configure ./configure
date-build make -j

# Build test program
cd ..
date-build g++ target.cc -I src -c
date-link target.o src/.libs/libcares.a 
```

### Start fuzzing

```sh
❯ mkdir dyn && cd dyn
date-fuzz -i ../sym/seed -o out -- ./app
```

#### Direct invocation

You need to have `tmux` installed (`sudo apt-get install tmux`), then execute:

```sh
date-fuzz ./app 
```

#### AFL customization
To customize parameters of AFL, add `-afl-mode` at the end of the command:


```sh
date-fuzz -afl-mode -my-afl-parameters ./app
```

See [documentation of AFL](http://lcamtuf.coredump.cx/afl/README.txt) for advanced tuning.
