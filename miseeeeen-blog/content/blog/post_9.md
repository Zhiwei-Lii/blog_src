---
title: "Intel CPU上面的Linux大小核调度问题"
date: 2024-07-30T12:43:52+08:00
draft: false
---
自从12代Alder lake开始, Intel引入了大小核架构. 
我最近也遇到了这样一个问题: 在Linux环境下面, 对于单线程的workload来说, 有时候它会运行在p-core, 有时候它会运行在e-core(即使p-core是idle的).
碰到这个问题的时候, 我是一脸懵逼的Orz. 因为印象中, 这类问题应该在adl刚出来的时候, 就已经受到广泛讨论了, 应该已经解决了才对, 毕竟我用的是kernel 6.1.

目前, 第一个线索是: 这个问题只发生在virtualization的环境下面(我们的方案是基于acrn的https://github.com/projectacrn), 在baremetal下面没有这个问题.

浅浅地读了一下sched这边的kernel代码. 我第一个怀疑的是select_task_rq_fair这个函数.
但是我仔细读了一下, 发现它就是遵循cfs的描述, 单纯地选择一个idlest的cpu, 塞进rq里面.
```
# kernel/sched/fair.c
7500   * select_task_rq_fair: Select target runqueue for the waking task in domains
7501   * that have the relevant SD flag set. In practice, this is SD_BALANCE_WAKE,
7502   * SD_BALANCE_FORK, or SD_BALANCE_EXEC.
7503   *
7504   * Balances load by selecting the idlest CPU in the idlest group, or under
7505   * certain conditions an idle sibling CPU if the domain has SD_WAKE_AFFINE set.
7506   *
7507   * Returns the target CPU number.
7508   */
7509  static int
7510  select_task_rq_fair(struct task_struct *p, int prev_cpu, int wake_flags)
```
接着我试着把VM/BM的load_balance都关掉. 发现即使是Baremetal的情况下, 也会选择一个e-core去运行. 这就表明了:
1. select_task_rq_fair也是会选择e-core的.
2. 在Baremetal下面, 是cpu load balance使得任务最终运行在e-core上面.
```
echo 0 > /sys/fs/cgroup/cpuset/[cgroup_name]/sched_load_balance
```
现在我已经知道了这个问题和load balance有关, 接着我就在寻找kernel是如何分辨p-core/e-core的.
最终发现是通过ITMT这个组件. ITMT(Intel Turbo Max Tech)这个名字听起来和scheduler没多大关系, 但它确实是负责cpu调度优先级的汗.
```
175  /**
176   * sched_set_itmt_core_prio() - Set CPU priority based on ITMT
177   * @prio:	Priority of cpu core
178   * @core_cpu:	The cpu number associated with the core
179   *
180   * The pstate driver will find out the max boost frequency
181   * and call this function to set a priority proportional
182   * to the max boost frequency. CPU with higher boost
183   * frequency will receive higher priority.
184   *
185   * No need to rebuild sched domain after updating
186   * the CPU priorities. The sched domains have no
187   * dependency on CPU priorities.
188   */
189  void sched_set_itmt_core_prio(int prio, int core_cpu)
190  {
191  	int cpu, i = 1;
192  
193  	for_each_cpu(cpu, topology_sibling_cpumask(core_cpu)) {
194  		int smt_prio;
195  
196  		/*
197  		 * Ensure that the siblings are moved to the end
198  		 * of the priority chain and only used when
199  		 * all other high priority cpus are out of capacity.
200  		 */
201  		smt_prio = prio * smp_num_siblings / (i * i);
202  		per_cpu(sched_core_priority, cpu) = smt_prio;
203  		i++;
204  	}
205  }
```
下面的命令可以查看ITMT的状态
```
cat /proc/sys/kernel/sched_itmt_enabled
```

在我有问题的环境下面, ITMT的这个节点根本就不存在, 看来就是和它有关了.
再继续往下看, ITMT是在intel_pstate的driver里面初始化的(所以要先确保cpufreq driver = intel_pstate).
它先去cppc里面拿一个core的highest_perf, 然后再根据这个值去分配core priority.
我的环境里,  挂在了line 357上面.
```
351  static void intel_pstate_set_itmt_prio(int cpu)
352  {
353  	struct cppc_perf_caps cppc_perf;
354  	static u32 max_highest_perf = 0, min_highest_perf = U32_MAX;
355  	int ret;
356  
357  	ret = cppc_get_perf_caps(cpu, &cppc_perf);
358  	if (ret)
359  		return;
360  
361  	/*
362  	 * On some systems with overclocking enabled, CPPC.highest_perf is hardcoded to 0xff.
363  	 * In this case we can't use CPPC.highest_perf to enable ITMT.
364  	 * In this case we can look at MSR_HWP_CAPABILITIES bits [8:0] to decide.
365  	 */
366  	if (cppc_perf.highest_perf == CPPC_MAX_PERF)
367  		cppc_perf.highest_perf = HWP_HIGHEST_PERF(READ_ONCE(all_cpu_data[cpu]->hwp_cap_cached));
368  
...
374  	sched_set_itmt_core_prio(cppc_perf.highest_perf, cpu);
375  
376  	if (max_highest_perf <= min_highest_perf) {
377  		if (cppc_perf.highest_perf > max_highest_perf)
378  			max_highest_perf = cppc_perf.highest_perf;
379  
380  		if (cppc_perf.highest_perf < min_highest_perf)
381  			min_highest_perf = cppc_perf.highest_perf;
382  
383  		if (max_highest_perf > min_highest_perf) {
...
390  			schedule_work(&sched_itmt_work);
391  		}
392  	}
393  }
```
再往下跟下去, 发现cppc的初始化失败了.
```
# drivers/acpi/cppc_acpi.c
663  int acpi_cppc_processor_probe(struct acpi_processor *pr)
664  {
665  	struct acpi_buffer output = {ACPI_ALLOCATE_BUFFER, NULL};
666  	union acpi_object *out_obj, *cpc_obj;
667  	struct cpc_desc *cpc_ptr;
668  	struct cpc_reg *gas_t;
669  	struct device *cpu_dev;
670  	acpi_handle handle = pr->handle;
671  	unsigned int num_ent, i, cpc_rev;
672  	int pcc_subspace_id = -1;
673  	acpi_status status;
674  	int ret = -ENODATA;
675  
676  	if (!osc_sb_cppc2_support_acked) {
677  		pr_debug("CPPC v2 _OSC not acked\n");
678  		if (!cpc_supported_by_cpu())
679  			return -ENODEV;
680  	}
```
至此, 问题已经比较清楚了. 
ITMT是帮助scheduler了解cpu调度优先级的. 因为ITMT工作有问题, 所以scheduler不知道哪些是p-core, 哪些是e-core, 所以调度出了问题. 而ITMT初始化失败是由于它依赖于CPPC. 在我的环境里, cppc初始化失败, 是因为它依赖于hypervisor提供acpi _osc的查询功能, 而目前这点还没有实现. 作为workaround, 暂时只能强制cpc_supported_by_cpu() return true. 或者干脆不要从cppc拿highest_perf了, 从MSR_HWP_CAPABILITIES 里面拿应该也可以?
