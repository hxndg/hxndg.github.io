---
layout:     post   				    # 使用的布局（不需要改）
title:      2023-03-28-devsecops实践
subtitle:   I'm programmer #副标题
date:       2023-03-28 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# devsecops实践



## 1 SAST实践















#### 1.1.1 SAST PARTS



目前问题总结

1. 目前CI的Clang-Tidy Check开启的Checker不充足，需要扫描代码库检查存在哪些代码问题，从而判断开启哪些Checker。
2. Clang-Tidy Check只针对用户编辑的文件进行扫描，用户不编辑的文件无法发现问题。

使用场景：

每天定时触发一次对onboard & offboard下以gpu模式编译的cc_library，cc_binary对象的SAST扫描，检查结果上传至coverage.qcraftai.com/sast下，经由TPM同学进行分诊，并创建Issue到具体协同者。

从用户角度来看，最终只会看到一个codecheck_report文件，包含各个代码文件（可能）存在的问题，报告末尾汇总各种Checker的错误报告数量，和对应文件的错误数量，参考

```Go
Found no defects in parse_pos_to_traj.cc
[HIGH] /home/qcrafter/.cache/bazel/_bazel_qcrafter/4207c8486d34138987115c0107ecb5f8/execroot/com_qcraft/onboard/planner/assist/lcc_map_builder.cc:148:7: 2nd function call argument is an uninitialized value [core.CallAndMessage]
      mapping::LanePathData(lane_path.start_fraction(), end_fraction,
      ^

Found 1 defect(s) in lcc_map_builder.cc

----==== Severity Statistics ====----
----------------------------
Severity | Number of reports
----------------------------
HIGH     |               882
MEDIUM   |             13814
LOW      |              1973
STYLE    |                20
----------------------------
----=================----

----==== Checker Statistics ====----
---------------------------------------------------------------------------------------
Checker name                                             | Severity | Number of reports
---------------------------------------------------------------------------------------
core.CallAndMessage                                      | HIGH     |               136
security.FloatLoopCounter                                | MEDIUM   |               214
cppcoreguidelines-special-member-functions               | LOW      |              1264
...

----==== File Statistics ====----
-------------------------------------------------------------------------
File name                                             | Number of reports
-------------------------------------------------------------------------
lcc_map_builder.cc                                    |                 2
ground_line.cc                                        |                12
```



目前开启的Checker包括

```Shell
---------------------------------------------------------------------
Checker name                                             | Severity |
---------------------------------------------------------------------
core.CallAndMessage                                      | HIGH     |
security.FloatLoopCounter                                | MEDIUM   |
cppcoreguidelines-special-member-functions               | LOW      |
clang-diagnostic-sign-compare                            | MEDIUM   |
clang-diagnostic-unused-parameter                        | MEDIUM   |
bugprone-sizeof-container                                | HIGH     |
bugprone-undefined-memory-manipulation                   | MEDIUM   |
optin.cplusplus.UninitializedObject                      | MEDIUM   |
cert-dcl58-cpp                                           | HIGH     |
clang-diagnostic-deprecated-copy-with-user-provided-copy | MEDIUM   |
performance-noexcept-move-constructor                    | MEDIUM   |
clang-diagnostic-deprecated-declarations                 | MEDIUM   |
readability-suspicious-call-argument                     | LOW      |
performance-move-const-arg                               | MEDIUM   |
misc-definitions-in-headers                              | MEDIUM   |
optin.cplusplus.VirtualCall                              | MEDIUM   |
deadcode.DeadStores                                      | LOW      |
bugprone-forwarding-reference-overload                   | LOW      |
core.NullDereference                                     | HIGH     |
bugprone-use-after-move                                  | HIGH     |
misc-unconventional-assign-operator                      | MEDIUM   |
clang-diagnostic-missing-field-initializers              | MEDIUM   |
google-global-names-in-headers                           | STYLE    |
performance-no-automatic-move                            | LOW      |
cert-dcl59-cpp                                           | MEDIUM   |
google-build-namespaces                                  | MEDIUM   |
bugprone-integer-division                                | MEDIUM   |
core.UndefinedBinaryOperatorResult                       | HIGH     |
bugprone-unused-return-value                             | MEDIUM   |
misc-redundant-expression                                | MEDIUM   |
bugprone-unhandled-exception-at-new                      | MEDIUM   |
core.uninitialized.Assign                                | HIGH     |
bugprone-misplaced-widening-cast                         | HIGH     |
bugprone-virtual-near-miss                               | MEDIUM   |
core.NonNullParamChecker                                 | HIGH     |
bugprone-incorrect-roundings                             | HIGH     |
misc-misplaced-const                                     | LOW      |
unix.Malloc                                              | MEDIUM   |
cplusplus.NewDeleteLeaks                                 | HIGH     |
bugprone-suspicious-missing-comma                        | HIGH     |
bugprone-lambda-function-name                            | LOW      |
core.uninitialized.UndefReturn                           | HIGH     |
optin.portability.UnixAPI                                | MEDIUM   |
bugprone-signed-char-misuse                              | MEDIUM   |
core.DivideZero                                          | HIGH     |
core.StackAddressEscape                                  | HIGH     |
bugprone-swapped-arguments                               | HIGH     |
bugprone-forward-declaration-namespace                   | LOW      |
cert-err09-cpp                                           | HIGH     |
unix.API                                                 | MEDIUM   |
bugprone-not-null-terminated-result                      | MEDIUM   |
cplusplus.NewDelete                                      | HIGH     |
bugprone-fold-init-type                                  | HIGH     |
bugprone-sizeof-expression                               | HIGH     |
cplusplus.Move                                           | HIGH     |
bugprone-inaccurate-erase                                | HIGH     |
bugprone-string-constructor                              | HIGH     |
clang-diagnostic-unused-result                           | MEDIUM   |
clang-diagnostic-#pragma-messages                        | MEDIUM   |
bugprone-signal-handler                                  | MEDIUM   |
bugprone-too-small-loop-variable                         | MEDIUM   |
misc-uniqueptr-reset-release                             | MEDIUM   |
---------------------------------------------------------------------
```



发现问题：

内存泄露，空指针调用，move后调用



附加SAST软件分析

SAST的软件有很多，比方说sonarqube，codechecker.针对C++我使用的是codechecker

+ sonarqube
+ codechecker
+ 



codechecker流程，如何安装？直接使用pip3 install codechecker即可，最终工程治理组将codechecker装进了docker里面，就不需要指定目录了，直接可以调用CodeChecker

+ `CodeChecker log` runs the given build command and records the executed compilation steps. These steps are written to an output file (Compilation Database) in a JSON format。这个实践起来并不是一个好的方法，因为本身在本地做编译就比较荒谬。实际上使用https://github.com/grailbio/bazel-compilation-database/可以方便的生成compile database，不需要再用codechecker本身的注入式log

  ```
  #这种做法不推荐
  /home/qcraft/.local/bin/CodeChecker log -o ./sim_server_compile_commands.json -b   "bazel --batch \
     build \
       --spawn_strategy=local \
       --strategy=Genrule=local \
       --copt=-DLEVELDB_PLATFORM_POSIX \
       --action_env=LD_PRELOAD=\$LD_PRELOAD \
       --action_env=LD_LIBRARY_PATH=\$LD_LIBRARY_PATH \
       --action_env=CC_LOGGER_GCC_LIKE=\$CC_LOGGER_GCC_LIKE \
       --action_env=CC_LOGGER_FILE=\$CC_LOGGER_FILE \
     //offboard/dashboard:sim_server"
  
  #这种做法推荐
  (
    cd "${INSTALL_DIR}" \
    && curl -L "https://github.com/grailbio/bazel-compilation-database/archive/${VERSION}.tar.gz" | tar -xz \
    && ln -f -s "${INSTALL_DIR}/bazel-compilation-database-${VERSION}/generate.py" bazel-compdb
  )
  
  # This will generate compile_commands.json in your workspace root.
  # ./bazel-compdb
  
  # Only generate some folder
  ./bazel-compdb -q //offboard/simulation/simulator/...
  ```

  

+ `CodeChecker analyze` uses the previously created JSON Compilation Database to perform an analysis on the project, outputting analysis results in a machine-readable (plist) format。这里的skip file能帮忙过滤掉proto文件，只分析我们感兴趣的文件

  + ```
    #skip file
    -*/proto/*
    +*/offboard/*
    +*/onboard/*
    -*
    ```

  + ```
    /home/qcraft/.local/bin/CodeChecker analyze ./sim_server_compile_commands.json -o codechecker_report
    ```







#### 1.1.2 如何避免误报

以CodeChecker为例，说一下怎么避免误报

1. **终极方法**，忽略或者屏蔽，analyzor支持skip文件级别。即

   - 直接在文件层面skip掉这些报错的文件，这种粒度比较大，可能会有导致其它问题被隐蔽，参考https://codechecker.readthedocs.io/en/latest/analyzer/user_guide/#skip
   - 使用codechecker支持的注释或者comment内容来避免false positive误报，参考https://codechecker.readthedocs.io/en/latest/analyzer/user_guide/#source-code-comments
   - 在web interface能够配置，某些是false positive的。不过这个是网络层面，还没做

2. **代码层面避免**，语言层面的保障应该是高于逻辑层面的。也就是说如果用户说逻辑层面能保证这种行为，这个就不能算作是false positive

   - 使用const等手段，来避免编译器认为出现对应的问题

   - 使用好理解的代码比方说，下面的情况保证了在宏`__clang_analyzer__`生效的情况下，static analysis会正确地找到对应的代码，从而不会出现错误的理解。这个宏的作用实际上并不止这些还

     For example the following code:

     ```cpp
     unsigned f(unsigned x) {
       return (x >> 1) & 1;
     }
     ```

     Could be rewritten as:

     ```cpp
     #ifndef __clang_analyzer__
     unsigned f(unsigned x) {
       return (x >> 1) & 1;
     }
     #else
     unsigned f(unsigned x) {
       return (x / 2) % 2;
     }
     #endif
     ```

   + 防御式编程，这个和语言就强相关了，简单的例子

     Clang-Tidy has lots of useful syntax based checks. Some of these checks find bug-prone code snippets. When these snippets are intentional, usually there is a natural way to make the intent more explicit. Unfortunately, it is hard to give a general guideline, because the details are different for each check. The documentation of the check might contain hints how to express intention more clearly. Let us look at an example:

     ```cpp
     double f(int i) {
       return 32 / (2 + i); // Warning, integer division, loss of precision.
     }
     ```

     It can be rewritten to as the following to suppress the warning:

     ```cpp
     double f(int i) {
       return (int)(32 / (2 + i)); // No warning, the intention is explicit.
     }
     ```

     The second version makes it clear even though the return value is a floating point value the loss of precision during integer division is intentional. Adding a comment why this is intentional would make this even clearer. Such edits makes the code easier to understand for fellow developers.

3. **逻辑层面，确实不可执行到的路径**

   + 对函数/部分代码添加正确的annotations，比方说C++ 11 的noreturn，https://en.cppreference.com/w/cpp/language/attributes

   + analyzor支持assert检查，编译debug对象的时候这些assert能够有效的避免出现相应的flase positive报错，编译器会检查到assert必然满足了某些条件，从而不会出现对应的问题。参考https://clang-analyzer.llvm.org/annotations.html#custom_assertions

   + 也是用assert代码来保证一定不会出错，认为走入了错误的path会导致报错

     ```cpp
     int f(MyEnum Val) {
       int x = 0;
       switch (Val) {
         case MyEnumA: x = 1; break;
         case MyEnumB: x = 5; break;
       }
       return 5/x; // Division by zero when Val == MyEnumC.
     }
     ```

     It can be rewritten to as the following to suppress the warning:

     ```cpp
     int f(MyEnum Val) {
       int x = 0;
       switch (Val) {
         case MyEnumA: x = 1; break;
         case MyEnumB: x = 5; break;
         default: assert(false); break;
       }
       return 5/x; // No warning.
     }
     ```

     Other macros or builtins expressing unreachable code may be used. Note that the rewritten code is also safer, since debug builds now check for more precondition violations.

     In case of C++11 or later, another option is to use Immediately-Invoked Function Expression (IIFE) to avoid assigning a meaningless value.

     ```cpp
     int f(MyEnum Val) {
       const int x = [&] { // Note the lambda.
         switch (Val) {
           case MyEnumA: return 1;
           case MyEnumB: return 5;
           default: assert(false); return 0;
         }
       } ();
       return 5/x; // No warning.
     }
     ```

   













## 2 代码审计

### 2.1 SQL部分

针对SQL的代码审计一般发生在有对数据库提交的部分，这里针对最常用的GORM做审计：GORM对结构体、map结构的value在框架底层都进行了预编译，所以使用此类方式进行CRUD操作时是十分安全的，初次之外存在如下问题

+ GORM对map的value进行了预编译，但却没对key进行预编译，因此如果是key是可以用户指定的，那么就可能存在注入点。我们的代码没有这个问题
+ GORM对表结构的查询使用了预编译，最常见的例子是用？表示数据，比方说`db.Where("name = ?", name)`，但是如果没使用默认的预编译比方说如果是`db.Where("name = '" + name + "'")`这种，那么就会有注入风险。
+ raw sql执行，`db.Raw(sql)`，只要用户能控制sql的内容就存在注入。发现了这个问题
+ exec sql执行，和上面的raw sql执行一样，
+ db.order，采用预编译执行SQL语句传入的参数不能作为SQL语句的一部分，那么OrderBy所代表的的列名、或者是后面跟随的ASC/DESC也无法进行预编译处理。







































### 2.2 XSS审计


## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)