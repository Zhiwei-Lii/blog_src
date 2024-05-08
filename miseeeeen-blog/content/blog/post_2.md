---
title: "Android Art里的JIT&AOT"
date: 2020-02-10T13:50:36+08:00
draft: false
---
简单介绍一下Art里的jit和aot.
本文分成三个部分:
1. JIT Introduction
2. AOT Introduction
3. Relation between JIT&AOT 

# JIT Introduction
JIT这里没有看太多的内容，先占个位:D

####Profile文件
Art会在app执行过程中对它做profiling, 记录hot method等优化信息, profiling的结果会存在storage当中, 以便下次执行的时候供art参考.

profile的文件格式如下:

```
324 /**
325 * Serialization format:
326 * [profile_header, zipped[[profile_line_header1, profile_line_header2...],[profile_line_data1,
327 *    profile_line_data2...]]]
328 * profile_header:
329 *   magic,version,number_of_dex_files,uncompressed_size_of_zipped_data,compressed_data_size
330 * profile_line_header:
331 *   dex_location,number_of_classes,methods_region_size,dex_location_checksum,num_method_ids
332 * profile_line_data:
333 *   method_encoding_1,method_encoding_2...,class_id1,class_id2...,startup/post startup bitmap
334 * The method_encoding is:
335 *    method_id,number_of_inline_caches,inline_cache1,inline_cache2...
336 * The inline_cache is:
337 *    dex_pc,[M|dex_map_size], dex_profile_index,class_id1,class_id2...,dex_profile_index2,...
338 *    dex_map_size is the number of dex_indeces that follows.
339 *       Classes are grouped per their dex files and the line
340 *       `dex_profile_index,class_id1,class_id2...,dex_profile_index2,...` encodes the
341 *       mapping from `dex_profile_index` to the set of classes `class_id1,class_id2...`
342 *    M stands for megamorphic or missing types and it's encoded as either
343 *    the byte kIsMegamorphicEncoding or kIsMissingTypesEncoding.
344 *    When present, there will be no class ids following.
345 **/
```

在art目录下面, google还提供了profman来作为profile相关操作的接口.
使用profman可以轻松地将profile打印成人类友好模式.

Example:
```
1. adb shell profman --generate-test-profile=/data/testprofile --generate-test-profile-seed=0 --apk=/system/framework/am.jar --dex-location=/system/framework/am.jar 
2. adb shell profman --dump-only --profile-file=/data/testprofile 
```

以下内容来自am.jar的profile, just an example:

```
=== Dex files  ===
=== profile ===
ProfileInfo:
am.jar [index=0] [checksum=ccdf4a91]
        hot methods: 40[], 50[], 64[], 89[], 91[], 99[], 135[], 144[],
        startup methods: 40, 50, 64, 144,
        post startup methods: 89, 91, 99, 135,
        classes:
```

根据profile文件的优化信息, art会在下面这个函数里面去判断是否要将method编译成native code.
```
986 bool CompilerDriver::ShouldCompileBasedOnProfile(const MethodReference& method_ref) const {
987  // Profile compilation info may be null if no profile is passed.
988  if (!CompilerFilter::DependsOnProfile(compiler_options_->GetCompilerFilter())) {
989    // Use the compiler filter instead of the presence of profile_compilation_info_ since
990    // we may want to have full speed compilation along with profile based layout optimizations.
991    return true;
992  }
993  // If we are using a profile filter but do not have a profile compilation info, compile nothing.
994  if (profile_compilation_info_ == nullptr) {
995    return false;
996  }
997  // Compile only hot methods, it is the profile saver's job to decide what startup methods to mark
998  // as hot.
999  bool result = profile_compilation_info_->GetMethodHotness(method_ref).IsHot();
1000
1001  if (kDebugProfileGuidedCompilation) {
1002    LOG(INFO) << "[ProfileGuidedCompilation] "
1003        << (result ? "Compiled" : "Skipped") << " method:" << method_ref.PrettyMethod(true);
1004  }
1005  return result;
1006}
```
# AOT Introduction

####AOT的编译策略
dex2oat有个选项叫--compiler-filter, 它的参数如下.
```
32  enum Filter {
33    kAssumeVerified,      // Skip verification but mark all classes as verified anyway.
34    kExtract,             // Delay verication to runtime, do not compile anything.
35    kVerify,              // Only verify classes.
36    kQuicken,             // Verify, quicken, and compile JNI stubs.
37    kSpaceProfile,        // Maximize space savings based on profile.
38    kSpace,               // Maximize space savings.
39    kSpeedProfile,        // Maximize runtime performance based on profile.
40    kSpeed,               // Maximize runtime performance.
41    kEverythingProfile,   // Compile everything capable of being compiled based on profile.
42    kEverything,          // Compile everything capable of being compiled.
                            // 注意kEverything这个参数
43  };
```

相关逻辑在这里
``` 
986 bool CompilerDriver::ShouldCompileBasedOnProfile(const MethodReference& method_ref) const {
987  // Profile compilation info may be null if no profile is passed.
    // 当选择了everthing之后, DependsOnProfile返回false,所以ShouldCompileBasedOnProfile返回true.
988  if (!CompilerFilter::DependsOnProfile(compiler_options_->GetCompilerFilter())) { 
989    // Use the compiler filter instead of the presence of profile_compilation_info_ since
990    // we may want to have full speed compilation along with profile based layout optimizations.
991    return true;
992  }
993  // If we are using a profile filter but do not have a profile compilation info, compile nothing.
994  if (profile_compilation_info_ == nullptr) {
995    return false;
996  }
997  // Compile only hot methods, it is the profile saver's job to decide what startup methods to mark
998  // as hot.
     // 假设选了kSpeedProfile, 会走到这里, 然后会去查询profile, 如果是hot的话, 才会返回true
999  bool result = profile_compilation_info_->GetMethodHotness(method_ref).IsHot(); 
1000
1001  if (kDebugProfileGuidedCompilation) {
1002    LOG(INFO) << "[ProfileGuidedCompilation] "
1003        << (result ? "Compiled" : "Skipped") << " method:" << method_ref.PrettyMethod(true);
1004  }
1005  return result;
1006 }
```

如果想要在安装apk的时候，将app完全编译成native, 可以这么做：
```
//在install apk的时候就会默认全部aot
setprop pm.dexopt.install everything
```
或者重编译指定的apk:
```
// everything是编译所有能编译的code
pm compile -m everything -f com.kiloo.subwaysurf
// speed-profile会根据profile文件来编译
pm compile -m speed-profile -f com.kiloo.subwaysurf
```

我尝试编译了一个apk. 下面是编译后的文件大小. 
|     | speed-profile (default mode)  | everything  |
|  ----  | ----  | ----  |
| base.art  | 32K | 0 |
| base.odex | 224K | 21M |
| base.vdex | 8M | 7.4M |

base.odex是一个elf, aot之后的native code就存在里面.
因为是首次安装, profile信息几乎没有, 所以speed-profile模式下的base.odex很小.

PS: profile的信息都保存在/data/misc/profiles, 查看profile的内容可用以下命令
profman --profile-file=./primary.prof  --dump-only

everything模式的缺点就是(1) install的时候慢一些 (2) 占用空间大一些



#####dex2oat的参数解析
**--instruction-set-variant= (default | atom | sandybridge | silvermont)**
**--instruction-set-features = (default | sse3 | sse4.1 | avx…)**
这两个参数共同决定了编译时候的cpu feature.

instruction-set-variant和instruction-set-features的值是由system property来的:
- dalvik.vm.isa.x86.variant/ dalvik.vm.isa.x86_64.variant
- dalvik.vm.isa.x86.features/ dalvik.vm.isa.x86_64.features

**--huge-method-max/--large-method-max/--small-method-max/--tiny-method-max**
定义包含多少条指令的method是huge/large/small/tiny method. 编译器对于它们有不同的处理方法.

1) compiler filter!= everything && huge, 会跳过编译步骤.
2) compiler filter!= everything && large && no branch, 会跳过编译步骤.
```
76 bool HGraphBuilder::SkipCompilation(size_t number_of_branches) {
```

3) small/tiny method在P的code里没有任何特殊处理.

**--inline-max-code-units**

定义一个函数最多包含多少code unit, 可以被优化成inline函数.
这个值等于0的话, 就不进行inline优化.

**--compiler-backend=(Quick|Optimizing)**

这个选项其实没什么用, 就算选quick也会跳转到optimizing.
```
31    case kQuick:
32      // TODO: Remove Quick in options.
33    case kOptimizing:
34      return CreateOptimizingCompiler(driver);
```

#####.art/.odex/.vdex的具体含义
.vdex: verified dex，其中包含 APK 的 DEX 代码，另外还有一些旨在加快验证速度的元数据. 
验证主要指的是, 当系统OTA等原因, 会需要重新验证dex的合法性. 
而vdex存放的是已经验证过的dex. 如果vdex存在, 就可以跳过验证这个步骤.

.odex: elf, native code存放的位置

.art: .art就是art虚拟机内部数据的一个snapshot.
因为有一些数据是通用的, 是每次vm起来都会需要的, 所以可以将它们提前存储起来.
而vm起来的时候通过mmap .art文件来达到加速目的.

# Relation between JIT&AOT 
在jit运行的时候, 它会收集hot method/class的数据, 然后把profile持久化到disk上面. 在android空闲的时候, BackgroundDexOptService会根据这个profile信息, 把apk重新aot.
![jit-arch.png](/images/post_2/16134950-7dfd6a495e70bffd.png)

![jit-profile-comp.png](/images/post_2/16134950-15dd2c836f4f603a.png)


BackgroundDexOptService的调用大致是:
runIdleOptimization(BackgroundDexOptService.java) 
-> performDexOptWithStatus(PackageManagerService.java) 
    -> dexopt(InstalldNativeService.cpp) 
        -> run_dex2oat(dexopt.cpp) //这里调用了dex2oat这个binary去做aot

BackgroundDexOptService会在特定条件下对apk进行aot.
它被调用的条件需要同时满足以下三条:
- setRequiresDeviceIdle(true)                          // 设备处于idle状态, idle的定义是: 息屏71mins or 睡眠状态71mins  or无线充电71mins
- setRequiresCharging(true)                             // 设备处于充电状态
- setPeriodic(IDLE_OPTIMIZATION_PERIOD) // 一天仅触发一次

BackgroundDexOptService的触发条件:
```
123        // Schedule a daily job which scans installed packages and compiles
124        // those with fresh profiling data.
125        js.schedule(new JobInfo.Builder(JOB_IDLE_OPTIMIZE, sDexoptServiceName)
126                    .setRequiresDeviceIdle(true)
127                    .setRequiresCharging(true)
128                    .setPeriodic(IDLE_OPTIMIZATION_PERIOD)
129                    .build());
```
关于Idle的定义: 
```
44    // Policy: we decide that we're "idle" if the device has been unused /
45    // screen off or dreaming or wireless charging dock idle for at least this long
46    private long mInactivityIdleThreshold; // 这个值等于71mins
```

