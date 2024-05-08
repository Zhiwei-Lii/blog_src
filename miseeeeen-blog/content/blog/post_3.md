---
title: "Android PGO Guide"
date: 2020-04-10T13:50:36+08:00
draft: false
---

##### Part 1 采集数据
1. 在fio的Android.bp加入下面的code.
```
pgo: {
    instrumentation: true,
    benchmarks: [
        "fio",                      // benchmarks可以理解为当前优化的workload的名字.
    ],
    profile_file: "fio.profdata",   // fio.profdata表示会去toolchain/pgo-profiles下面寻找该文件.
}
```
2. make fio ANDROID_PGO_INSTRUMENT=fio
3. 运行binary/library, 它会在/data/local/tmp下面生成profraw的文件
```
# build/soong/cc/pgo.go
const profileInstrumentFlag = "-fprofile-generate=/data/local/tmp"
```
4. llvm-profdata merge -output=fio.profdata default_xxxxxx.profraw
注意llvm-profdata需要用Android prebuild的版本, 位于./prebuilts/clang/host/
 llvm-profdata show -all-functions fio.profdata      // 可以dump profile信息, 可以通过function id将default_xxx.profraw和具体的library/binary对应起来.

##### Part 2 使用profile来优化
1. 把profdata放到toolchain/pgo-profiles
2. make fio      // 不设置ANDROID_PGO_INSTRUMENT的话, 它就不会插入profile的代码.
3. run 

##### PS:
1. 进程会在exit的时候dump profile, 收到signal 9的时候并不会dump. 
所以如果是要优化长期存在的service, 需要做些hack.
```
# external/compiler-rt/lib/profile/InstrProfilingFile.c
int __llvm_profile_register_write_file_atexit(void) {
```
2. 假设已经带上ANDROID_PGO_INSTRUMENT编译过libc了. 但是你在make别的库的时候, 没有加这个变量, 同时这个库引用过libc, 那也会引起libc的重编译
3. 可以通过nm或者objdump来验证opted library是否符合预期. 
(hot的method排列在相邻的VMA上)
```
nm -n [lib_name]
```

参考: [https://source.android.com/devices/tech/perf/pgo](https://source.android.com/devices/tech/perf/pgo)

