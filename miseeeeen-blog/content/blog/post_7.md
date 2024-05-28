---
title: "关于intel_pstate下面的powersave和performance governor"
date: 2024-05-28T10:17:36+08:00
draft: false
---

我们都知道, powersave和performance是cpufreq里面两个不同的governor, 用于控制cpu进入不同的频率(p_state).

但是我一直有这样一个固有认知:
1. 当cpu处于powersave mode时, 出于省电考虑, cpufreq会运行在最低频率. 
2. 当cpu处于performance mode时, 出于性能考虑, cpufreq会运行在turbo允许的最高频率.

但是最近才发现: 当我们的cpufreq driver = intel_pstate时, powersave其实更像是以前的ondemand/schedutil, 即根据实际情况, 动态调整power和performance之间的关系, 而不是死板地运行在最低频率.
```
zhiwei@zhiwei-pc:~$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
intel_pstate

zhiwei@zhiwei-pc:~$ zhiwei@zhiwei-pc:~$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
powersave

# 在intel_pstate下面, 只有performance和powersave两种governor
zhiwei@zhiwei-pc:~$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
performance powersave
```

然后我就去查看了一下相关的kernel code, 试图理解: 当我们把governor设成powersave之后, 发生了什么.
```
// 这是kernel 6.1, 调用链大致如下:
// store_scaling_governor -> cpufreq_set_policy -> intel_pstate_set_policy -> intel_pstate_hwp_set

// drivers/cpufreq/intel_pstate.c
static void intel_pstate_hwp_set(unsigned int cpu)
{
...
// 这里的max/min对应了scaling_max_freq/scaling_min_freq
        max = cpu_data->max_perf_ratio;
        min = cpu_data->min_perf_ratio;

// 如果是performance的话, 把min也设成max, 达到性能最大化
        if (cpu_data->policy == CPUFREQ_POLICY_PERFORMANCE)
                min = max;
// 读取MSR_HWP_REQUEST这个寄存器的值
        rdmsrl_on_cpu(cpu, MSR_HWP_REQUEST, &value);

        value &= ~HWP_MIN_PERF(~0L);
        value |= HWP_MIN_PERF(min);

        value &= ~HWP_MAX_PERF(~0L);
        value |= HWP_MAX_PERF(max);
...
        if (cpu_data->policy == CPUFREQ_POLICY_PERFORMANCE) {
                epp = intel_pstate_get_epp(cpu_data, value);
                cpu_data->epp_powersave = epp;
                /* If EPP read was failed, then don't try to write */
                if (epp < 0)
                        goto skip_epp;

// 如果是performance, epp置成0
                epp = 0;
        } else {
                /* skip setting EPP, when saved value is invalid */
                if (cpu_data->epp_powersave < 0)
                        goto skip_epp;

                /*
                 * No need to restore EPP when it is not zero. This
                 * means:
                 *  - Policy is not changed
                 *  - user has manually changed
                 *  - Error reading EPB
                 */
                epp = intel_pstate_get_epp(cpu_data, value);
                if (epp)
                        goto skip_epp;

// 如果是powersave, 则epp还原成旧值.
                epp = cpu_data->epp_powersave;
        }

        if (boot_cpu_has(X86_FEATURE_HWP_EPP)) {
                value &= ~GENMASK_ULL(31, 24);
                value |= (u64)epp << 24;
        } else {
// 设置epp, 即/sys/devices/system/cpu/cpu[X]/cpufreq/energy_performance_preference
                intel_pstate_set_epb(cpu, epp);
        }
skip_epp:
        WRITE_ONCE(cpu_data->hwp_req_cached, value);
// 写入MSR_HWP_REQUEST
        wrmsrl_on_cpu(cpu, MSR_HWP_REQUEST, value);
}
```

从以上code, 我们可以发现:
1. pstate driver是通过调整MSR_HWP_REQUEST这个MSR, 来影响频率.
2. 这段code一共修改了三个变量: min_freq/max_freq/epp, 除此以外没有别的改动.

min_freq/max_freq很容易理解, 那epp是什么东西呢? 
想理解这个问题, 我们需要翻一下intel的说明书.
下面这段出自Four-Volume Set of Intel® 64 and IA-32 Architectures Software Developer’s Manuals (Vol 3. Chapter 15)
![image.png](/images/post_7/16134950-9863c7f0cb0954a4.png)
> Minimum_Performance (bits 7:0, RW) — Conveys a hint to the HWP hardware. The OS programs the
minimum performance hint to achieve the required quality of service (QOS) or to meet a service level
agreement (SLA) as needed. Note that an excursion below the level specified is possible due to hardware
constraints. The default value of this field is IA32_HWP_CAPABILITIES.Lowest_Performance.

>Maximum_Performance (bits 15:8, RW) — Conveys a hint to the HWP hardware. The OS programs this
field to limit the maximum performance that is expected to be supplied by the HWP hardware. Excursions
above the limit requested by OS are possible due to hardware coordination between the processor cores and
other components in the package. The default value of this field is IA32_HWP_CAPABILITIES.Highest_Performance.

>Energy_Performance_Preference (bits 31:24, RW) — Conveys a hint to the HWP hardware. The OS may
write a range of values from 0 (performance preference) to 0FFH (energy efficiency preference) to influence
the rate of performance increase /decrease and the result of the hardware's energy efficiency and performance
optimizations. The default value of this field is 80H. Note: If CPUID.06H:EAX[bit 10] indicates that this field is
not supported, HWP uses the value of the IA32_ENERGY_PERF_BIAS MSR to determine the energy efficiency /
performance preference.

总结一下的话:
1. min_freq/max_freq/epp都是给hwp的hint, 但仅是hint, 实际频率要由hwp决定.
2. epp是一个0~255的值, epp的值越是接近0, 则性能越优, 耗电越大.

这里的epp其实对应了/sys/devices/system/cpu/cpu[X]/cpufreq/energy_performance_preference
可以看到performance_governor=powersave的时候, epp=balanced_performance. 说明intel_pstate的情况下, powersave的性能也没那么差. (至少是balanced_performance, 而不是balanced_power)
```
zhiwei@zhiwei-pc:~$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
powersave
zhiwei@zhiwei-pc:~$ cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
balance_performance

# epp设成任意一个[0, 256)的值, 都是可以的. 不仅限于下面几个档位.
zhiwei@zhiwei-pc:~$ cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_available_preferences
default performance balance_performance balance_power power
```
