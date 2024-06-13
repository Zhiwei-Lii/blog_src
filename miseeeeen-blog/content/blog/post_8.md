---
title: "记录一次性能调优的经历"
date: 2024-06-13T10:17:36+08:00
draft: false
---
请原谅我不知道标题该起什么名字合适:D
最近在负责某benchmark在x86平台上的cpu性能优化, 遇到了这样一个问题: 该benchmark有app版本和native版本, app版本其实就是简单地通过jni调用到了native层, 明明native层的代码都是一样的, 但是它们的性能差了10%以上.

由于该benchmark比较复杂, 所以我第一个实验就是写了一个简单的app, 模拟它的行为, 同样调到了它的native层, 然后发现能够复现问题. 
下一步重写了它的native层, 依旧发现可以复现问题.

至此, 这个问题已经和benchmark无关了, 变成了这样一个通用的问题: 为什么在x86平台上面,  同样的code在app里面被调用到的时候, 相较native层, 会出现10%以上的性能损失?

于是, 我就开始了一通调查, 一个比较重要的发现是: 这个问题和selinux有关, 只有selinux=enforcing的时候, 会有这个现象. selinux=permissive的时候, 就不能复现问题.

顺着这个线索, 我开始跟踪zygote的启动过程, 发现了下面这一段代码.
Zygote在init的时候, 会去check selinux的状态.
如果是permissive的话, 什么都不做.
如果是enforcing的话, 会给所有后续从zygote fork的进程, 加上seccomp这个security feature.

```
632  static void SetUpSeccompFilter(uid_t uid, bool is_child_zygote) {
633    if (!gIsSecurityEnforced) {
634      ALOGI("seccomp disabled by setenforce 0");
635      return;
636    }
637  
638    // Apply system or app filter based on uid.
639    if (uid >= AID_APP_START) {
640      if (is_child_zygote) {
641        set_app_zygote_seccomp_filter();
642      } else {
643        set_app_seccomp_filter();
644      }
645    } else {
646      set_system_seccomp_filter();
647    }
648  }
```

然后seccomp加上的时候, 默认会disable掉这两个优化: Speculation_Store_Bypass和SpeculationIndirectBranch.
状态如下: (这是benchmark app的dump)
```
# /proc/[pid]/status
Seccomp:	2
Seccomp_filters:	1
Speculation_Store_Bypass:	thread force mitigated
SpeculationIndirectBranch:	conditional force disabled
```
不加seccomp, 它是这样的: (这是native shell的dump, native shell起来的进程由于与zygote无关, 所以不会被添加seccomp的配置)
```
# /proc/[pid]/status
Seccomp:	0
Seccomp_filters:	0
Speculation_Store_Bypass:	thread vulnerable
SpeculationIndirectBranch:	conditional enabled
```

至此, 问题就比较清楚了, 总结一下就是: zygote配置seccomp的时候, 出于安全考虑, 关掉了两个处理器level的优化, 导致app层的code执行得相对较慢. 如果想强制改变这个行为, 只需要加上这两个kernel参数就好了. “ spec_store_bypass_disable=off  spectre_v2_user=off ”
