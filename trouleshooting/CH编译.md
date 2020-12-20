记一次 MacOSX 编译 CH 失败问题：

编译：https://clickhouse.tech/docs/zh/development/build-osx/

1. 失败日志：
```
$ cmake .. -DCMAKE_CXX_COMPILER=`which clang++` -DCMAKE_C_COMPILER=`which clang`
-- The C compiler identification is Clang 10.0.0
-- The CXX compiler identification is Clang 10.0.0
-- Check for working C compiler: /usr/local/opt/llvm/bin/clang
-- Check for working C compiler: /usr/local/opt/llvm/bin/clang - broken
CMake Error at /usr/local/Cellar/cmake/3.17.3/share/cmake/Modules/CMakeTestCCompiler.cmake:60 (message):
  The C compiler

    "/usr/local/opt/llvm/bin/clang"

  is not able to compile a simple test program.

  It fails with the following output:

    Change Dir: /Users/wangqian/ClickHouse/build/CMakeFiles/CMakeTmp

    Run Build Command(s):/usr/local/bin/ninja cmTC_be84d && [1/2] Building C object CMakeFiles/cmTC_be84d.dir/testCCompiler.c.o
    [2/2] Linking C executable cmTC_be84d
    FAILED: cmTC_be84d
    : && /usr/local/opt/llvm/bin/clang -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk -Wl,-search_paths_first -Wl,-headerpad_max_install_names -L/usr/local/opt/llvm/lib CMakeFiles/cmTC_be84d.dir/testCCompiler.c.o  -o cmTC_be84d   && :
    ld: unknown option: -platform_version
    clang-10: error: linker command failed with exit code 1 (use -v to see invocation)
    ninja: build stopped: subcommand failed.

  CMake will not be able to correctly generate this project.
Call Stack (most recent call first):
  CMakeLists.txt:29 (project)

-- Configuring incomplete, errors occurred!
See also "ClickHouse/build/CMakeFiles/CMakeOutput.log".
See also "ClickHouse/build/CMakeFiles/CMakeError.log".
```
解决：删除环境变量，卸载 cmake 等一系列安装包，删除 Clickhouse 重装。

2. mac 自带的clang 无法编译，需要安装llvm
CMake Error at cmake/tools.cmake:20 (message):
  AppleClang is not supported, you should install clang from brew.
  
解决方法：brew install llvm