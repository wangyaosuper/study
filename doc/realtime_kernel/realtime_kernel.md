# Kernel实时化
## 准备工作
### 获得准确的Jiffies64
通过Kernel 2 User共享内存，获取Jiffies64，不产生用户态和内核态的上下文切换
内核获取Jiffies64的函数   get_jiffies_64()

### qemu的时钟模式
--icount shift=2
这一点很重要, 它指定了每2的N次方纳秒(virtual time)，执行一个ARM指令，这样guest机器的虚拟时间就可以和虚拟CPU的指令执行周期一一对应，如果icount shift=auto(默认)，guest机器的虚拟时间是和host机器的真实时间一一对应，就起不到性能测试效果

-icount [shift=N|auto][,align=on|off][,sleep=on|off][,rr=record|replay,rrfile=filename[,rrsnapshot=snapshot]]

    Enable virtual instruction counter. The virtual cpu will execute one instruction every 2^N ns of virtual time. If auto is specified then the virtual cpu speed will be automatically adjusted to keep virtual time within a few seconds of real time.
    
    Note that while this option can give deterministic behavior, it does not provide cycle accurate emulation. Modern CPUs contain superscalar out of order cores with complex cache hierarchies. The number of instructions executed often has little or no correlation with actual performance.
    
    When the virtual cpu is sleeping, the virtual time will advance at default speed unless sleep=on is specified. With sleep=on, the virtual time will jump to the next timer deadline instantly whenever the virtual cpu goes to sleep mode and will not advance if no timer is enabled. This behavior gives deterministic execution times from the guest point of view. The default if icount is enabled is sleep=off. sleep=on cannot be used together with either shift=auto or align=on.
    
    align=on will activate the delay algorithm which will try to synchronise the host clock and the virtual clock. The goal is to have a guest running at the real frequency imposed by the shift option. Whenever the guest clock is behind the host clock and if align=on is specified then we print a message to the user to inform about the delay. Currently this option does not work when shift is auto. Note: The sync algorithm will work for those shift values for which the guest clock runs ahead of the host clock. Typically this happens when the shift value is high (how high depends on the host machine). The default if icount is enabled is align=off.

### lmbench

#### lmbench 运行方式
编译：
make OS=arm-gnu-linux CC=arm-linux-gnueabi-gcc lmbench
运行：
env OS=arm-gnu-linux ./config-run
env OS=arm-gnu-linux ./results 
输出：
../results/arm-gnu-linux/

#### lmbench 测试用例
*带宽测评工具
          —读取缓存文件
          —拷贝内存
          —读内存
          —写内存
          —管道
          —TCP
* 反应时间测评工具
          —上下文切换
          —网络： 连接的建立，管道，TCP，UDP和RPC hot potato
          —文件系统的建立和删除
          —进程创建
          —信号处理
          —上层的系统调用
          —内存读入反应时间
* 其他
          —处理器时钟比率计算
          
#### context_switch 测试用例
fork出海量子进程，父进程和子进程之间频繁进行pipe发送消息，实现进程间频繁切换
测量过程时间，并扣除pipe操作时间

## 如何使用Linux的 RT Patch
1. cp patch-4.19.94-rt38.patch /home/zsj	# 拷贝 rt 补丁包到 /home/zsj 目录
2. cd /home/zsj/linux-4.19.94   			# 进入内核源码目录
3. patch -p1 <../patch-4.19.94-rt38.patch   # 给内核打入实时补丁包
4. make menuconfig                          # 进入内核的图像化配置界面
5. /PREEMPT_RTB								# 搜素 rt 选项的位置，并进入该位置，勾选 Fully Preemptible Kernel(RT) 后，保存退出
6. make uImage	-j4							# 编译 rt 内核
7. # 替换到原有的 uImage 后，重启即可
8. uname -a									# 检查 rt 移植是否成功
————————————————
版权声明：本文为CSDN博主「end_宿命」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_27149449/article/details/118857704

### 如何编译RT Kernel
menuconfig中激活 Configure standard kernel features (expert users)  为 Y
然后选择 Preemption Model 为 Fully Preemptible Kernel (Real-Time)
最后在外层的Kernel Hacking中关闭 check for stack overflows




## Linux的 RT Patch内容


### CONFIG_PREEMPT_RT_FULL
#### switch_kmaps
/arch/arm/include/asm/switch_to.h
```c
#define switch_to(prev,next,last)					\
 do {									\
 	__complete_pending_tlbi();					\
+	switch_kmaps(prev, next);					\
 	last = __switch_to(prev,task_thread_info(prev), task_thread_info(next));	\
 } while (0)
```
```c
--- a/arch/arm/mm/highmem.c
+++ b/arch/arm/mm/highmem.c
@@ -31,6 +31,11 @@ static inline pte_t get_fixmap_pte(unsigned long vaddr)
 	return *ptep;
 }
 
+static unsigned int fixmap_idx(int type)
+{
+	return FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+}
+
 void *kmap(struct page *page)
 {
 	might_sleep();
@@ -51,12 +56,13 @@ EXPORT_SYMBOL(kunmap);
 
 void *kmap_atomic(struct page *page)
 {
+	pte_t pte = mk_pte(page, kmap_prot);
 	unsigned int idx;
 	unsigned long vaddr;
 	void *kmap;
 	int type;
 
-	preempt_disable();
+	preempt_disable_nort();
 	pagefault_disable();
 	if (!PageHighMem(page))
 		return page_address(page);
@@ -76,7 +82,7 @@ void *kmap_atomic(struct page *page)
 
 	type = kmap_atomic_idx_push();
 
-	idx = FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+	idx = fixmap_idx(type);
 	vaddr = __fix_to_virt(idx);
 #ifdef CONFIG_DEBUG_HIGHMEM
 	/*
@@ -90,7 +96,10 @@ void *kmap_atomic(struct page *page)
 	 * in place, so the contained TLB flush ensures the TLB is updated
 	 * with the new mapping.
 	 */
-	set_fixmap_pte(idx, mk_pte(page, kmap_prot));
+#ifdef CONFIG_PREEMPT_RT_FULL
+	current->kmap_pte[type] = pte;
+#endif
+	set_fixmap_pte(idx, pte);
 
 	return (void *)vaddr;
 }
@@ -103,44 +112,75 @@ void __kunmap_atomic(void *kvaddr)
 
 	if (kvaddr >= (void *)FIXADDR_START) {
 		type = kmap_atomic_idx();
-		idx = FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+		idx = fixmap_idx(type);
 
 		if (cache_is_vivt())
 			__cpuc_flush_dcache_area((void *)vaddr, PAGE_SIZE);
+#ifdef CONFIG_PREEMPT_RT_FULL
+		current->kmap_pte[type] = __pte(0);
+#endif
 #ifdef CONFIG_DEBUG_HIGHMEM
 		BUG_ON(vaddr != __fix_to_virt(idx));
-		set_fixmap_pte(idx, __pte(0));
 #else
 		(void) idx;  /* to kill a warning */
 #endif
+		set_fixmap_pte(idx, __pte(0));
 		kmap_atomic_idx_pop();
 	} else if (vaddr >= PKMAP_ADDR(0) && vaddr < PKMAP_ADDR(LAST_PKMAP)) {
 		/* this address was obtained through kmap_high_get() */
 		kunmap_high(pte_page(pkmap_page_table[PKMAP_NR(vaddr)]));
 	}
 	pagefault_enable();
-	preempt_enable();
+	preempt_enable_nort();
 }
 EXPORT_SYMBOL(__kunmap_atomic);
 
 void *kmap_atomic_pfn(unsigned long pfn)
 {
+	pte_t pte = pfn_pte(pfn, kmap_prot);
 	unsigned long vaddr;
 	int idx, type;
 	struct page *page = pfn_to_page(pfn);
 
-	preempt_disable();
+	preempt_disable_nort();
 	pagefault_disable();
 	if (!PageHighMem(page))
 		return page_address(page);
 
 	type = kmap_atomic_idx_push();
-	idx = FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+	idx = fixmap_idx(type);
 	vaddr = __fix_to_virt(idx);
 #ifdef CONFIG_DEBUG_HIGHMEM
 	BUG_ON(!pte_none(get_fixmap_pte(vaddr)));
 #endif
-	set_fixmap_pte(idx, pfn_pte(pfn, kmap_prot));
+#ifdef CONFIG_PREEMPT_RT_FULL
+	current->kmap_pte[type] = pte;
+#endif
+	set_fixmap_pte(idx, pte);
 
 	return (void *)vaddr;
 }
+#if defined CONFIG_PREEMPT_RT_FULL
+void switch_kmaps(struct task_struct *prev_p, struct task_struct *next_p)
+{
+	int i;
+
+	/*
+	 * Clear @prev's kmap_atomic mappings
+	 */
+	for (i = 0; i < prev_p->kmap_idx; i++) {
+		int idx = fixmap_idx(i);
+
+		set_fixmap_pte(idx, __pte(0));
+	}
+	/*
+	 * Restore @next_p's kmap_atomic mappings
+	 */
+	for (i = 0; i < next_p->kmap_idx; i++) {
+		int idx = fixmap_idx(i);
+
+		if (!pte_none(next_p->kmap_pte[i]))
+			set_fixmap_pte(idx, next_p->kmap_pte[i]);
+	}
+}
+#endif

```



### HAVE_PREEMPT_LAZY
#### preempt_lazy_count
/arch/arm/include/asm/thread_info.h
```c
struct thread_info {
 	unsigned long		flags;		/* low level flags */
 	int			preempt_count;	/* 0 => preemptable, <0 => bug */`

+	int			preempt_lazy_count; /* 0 => preemptable, <0 => bug */`

mm_segment_t		addr_limit;	/* address limit */
struct task_struct	*task;		/* main task structure */
__u32			cpu;		/* cpu */
```
```c
 #define TIF_SYSCALL_TRACE	4	/* syscall trace active */
 #define TIF_SYSCALL_AUDIT	5	/* syscall auditing active */
 #define TIF_SYSCALL_TRACEPOINT	6	/* syscall tracepoint instrumentation */
-#define TIF_SECCOMP		7	/* seccomp syscall filtering active */
+#define TIF_SECCOMP		8	/* seccomp syscall filtering active */
+#define TIF_NEED_RESCHED_LAZY	7
```

```c
 #define _TIF_SIGPENDING		(1 << TIF_SIGPENDING)
 #define _TIF_NEED_RESCHED	(1 << TIF_NEED_RESCHED)
 #define _TIF_NOTIFY_RESUME	(1 << TIF_NOTIFY_RESUME)
+#define _TIF_NEED_RESCHED_LAZY	(1 << TIF_NEED_RESCHED_LAZY)
 #define _TIF_UPROBE		(1 << TIF_UPROBE)
 #define _TIF_SYSCALL_TRACE	(1 << TIF_SYSCALL_TRACE)
 #define _TIF_SYSCALL_AUDIT	(1 << TIF_SYSCALL_AUDIT)
```

```c
 static inline bool should_resched(int preempt_offset)
 {
+#ifdef CONFIG_PREEMPT_LAZY
+	u64 pc = READ_ONCE(current_thread_info()->preempt_count);
+	if (pc == preempt_offset)
+		return true;
+
+	if ((pc & ~PREEMPT_NEED_RESCHED) != preempt_offset)
+		return false;
+
+	if (current_thread_info()->preempt_lazy_count)
+		return false;
+	return test_thread_flag(TIF_NEED_RESCHED_LAZY);
+#else
 	u64 pc = READ_ONCE(current_thread_info()->preempt_count);
 	return pc == preempt_offset;
+#endif
 }
```


/arch/arm/kernel/entry-armv.S
```assembly
 #ifdef CONFIG_PREEMPT
 	ldr	r8, [tsk, #TI_PREEMPT]		@ get preempt count
-	ldr	r0, [tsk, #TI_FLAGS]		@ get flags
 	teq	r8, #0				@ if preempt count != 0
+	bne	1f				@ return from exeption
+	ldr	r0, [tsk, #TI_FLAGS]		@ get flags
+	tst	r0, #_TIF_NEED_RESCHED		@ if NEED_RESCHED is set
+	blne	svc_preempt			@ preempt!
+
+	ldr	r8, [tsk, #TI_PREEMPT_LAZY]	@ get preempt lazy count
+	teq	r8, #0				@ if preempt lazy count != 0
 	movne	r0, #0				@ force flags to 0
-	tst	r0, #_TIF_NEED_RESCHED
+	tst	r0, #_TIF_NEED_RESCHED_LAZY
 	blne	svc_preempt
+1:
 #endif
```

```assembly
#ifdef CONFIG_PREEMPT
	ldr	r8, [tsk, #TI_PREEMPT]		@ get preempt count
-	ldr	r0, [tsk, #TI_FLAGS]		@ get flags
 	teq	r8, #0				@ if preempt count != 0
+	bne	1f				@ return from exeption
+	ldr	r0, [tsk, #TI_FLAGS]		@ get flags
+	tst	r0, #_TIF_NEED_RESCHED		@ if NEED_RESCHED is set
+	blne	svc_preempt			@ preempt!
+
+	ldr	r8, [tsk, #TI_PREEMPT_LAZY]	@ get preempt lazy count
+	teq	r8, #0				@ if preempt lazy count != 0
 	movne	r0, #0				@ force flags to 0
-	tst	r0, #_TIF_NEED_RESCHED
+	tst	r0, #_TIF_NEED_RESCHED_LAZY
 	blne	svc_preempt
+1:
#endif
```
/arch/arm/kernel/entry-common.S
```assembly
@@ -53,7 +53,9 @@ saved_pc	.req	lr
 	cmp	r2, #TASK_SIZE
 	blne	addr_limit_check_failed
 	ldr	r1, [tsk, #TI_FLAGS]		@ re-check for syscall tracing
-	tst	r1, #_TIF_SYSCALL_WORK | _TIF_WORK_MASK
+	tst	r1, #((_TIF_SYSCALL_WORK | _TIF_WORK_MASK) & ~_TIF_SECCOMP)
+	bne	fast_work_pending
+	tst	r1, #_TIF_SECCOMP
 	bne	fast_work_pending
 
 
@@ -90,8 +92,11 @@ ENDPROC(ret_fast_syscall)
 	cmp	r2, #TASK_SIZE
 	blne	addr_limit_check_failed
 	ldr	r1, [tsk, #TI_FLAGS]		@ re-check for syscall tracing
-	tst	r1, #_TIF_SYSCALL_WORK | _TIF_WORK_MASK
+	tst	r1, #((_TIF_SYSCALL_WORK | _TIF_WORK_MASK) & ~_TIF_SECCOMP)
+	bne	do_slower_path
+	tst	r1, #_TIF_SECCOMP
 	beq	no_work_pending
+do_slower_path:
  UNWIND(.fnend		)
 ENDPROC(ret_fast_syscall)
```

### fault
/arch/arm/mm/fault.c
```c
@@ -434,6 +434,9 @@ do_translation_fault(unsigned long addr, unsigned int fsr,
 	if (addr < TASK_SIZE)
 		return do_page_fault(addr, fsr, regs);
 
+	if (interrupts_enabled(regs))
+		local_irq_enable();
+
 	if (user_mode(regs))
 		goto bad_area;
 
@@ -501,6 +504,9 @@ do_translation_fault(unsigned long addr, unsigned int fsr,
 static int
 do_sect_fault(unsigned long addr, unsigned int fsr, struct pt_regs *regs)
 {
+	if (interrupts_enabled(regs))
+		local_irq_enable();
+
 	do_bad_area(addr, fsr, regs);
 	return 0;
 }
diff --git a/arch/arm/mm/highmem.c b/arch/arm/mm/highmem.c
index a76f8ace9ce66..66806fdeb787c 100644
--- a/arch/arm/mm/highmem.c
+++ b/arch/arm/mm/highmem.c
@@ -31,6 +31,11 @@ static inline pte_t get_fixmap_pte(unsigned long vaddr)
 	return *ptep;
 }
```

### highmem
/arch/arm/mm/highmem.c
```c
@@ -31,6 +31,11 @@ static inline pte_t get_fixmap_pte(unsigned long vaddr)
 	return *ptep;
 }
 
+static unsigned int fixmap_idx(int type)
+{
+	return FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+}
+
 void *kmap(struct page *page)
 {
 	might_sleep();
@@ -51,12 +56,13 @@ EXPORT_SYMBOL(kunmap);
 
 void *kmap_atomic(struct page *page)
 {
+	pte_t pte = mk_pte(page, kmap_prot);
 	unsigned int idx;
 	unsigned long vaddr;
 	void *kmap;
 	int type;
 
-	preempt_disable();
+	preempt_disable_nort();
 	pagefault_disable();
 	if (!PageHighMem(page))
 		return page_address(page);
@@ -76,7 +82,7 @@ void *kmap_atomic(struct page *page)
 
 	type = kmap_atomic_idx_push();
 
-	idx = FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+	idx = fixmap_idx(type);
 	vaddr = __fix_to_virt(idx);
 #ifdef CONFIG_DEBUG_HIGHMEM
 	/*
@@ -90,7 +96,10 @@ void *kmap_atomic(struct page *page)
 	 * in place, so the contained TLB flush ensures the TLB is updated
 	 * with the new mapping.
 	 */
-	set_fixmap_pte(idx, mk_pte(page, kmap_prot));
+#ifdef CONFIG_PREEMPT_RT_FULL
+	current->kmap_pte[type] = pte;
+#endif
+	set_fixmap_pte(idx, pte);
 
 	return (void *)vaddr;
 }
@@ -103,44 +112,75 @@ void __kunmap_atomic(void *kvaddr)
 
 	if (kvaddr >= (void *)FIXADDR_START) {
 		type = kmap_atomic_idx();
-		idx = FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+		idx = fixmap_idx(type);
 
 		if (cache_is_vivt())
 			__cpuc_flush_dcache_area((void *)vaddr, PAGE_SIZE);
+#ifdef CONFIG_PREEMPT_RT_FULL
+		current->kmap_pte[type] = __pte(0);
+#endif
 #ifdef CONFIG_DEBUG_HIGHMEM
 		BUG_ON(vaddr != __fix_to_virt(idx));
-		set_fixmap_pte(idx, __pte(0));
 #else
 		(void) idx;  /* to kill a warning */
 #endif
+		set_fixmap_pte(idx, __pte(0));
 		kmap_atomic_idx_pop();
 	} else if (vaddr >= PKMAP_ADDR(0) && vaddr < PKMAP_ADDR(LAST_PKMAP)) {
 		/* this address was obtained through kmap_high_get() */
 		kunmap_high(pte_page(pkmap_page_table[PKMAP_NR(vaddr)]));
 	}
 	pagefault_enable();
-	preempt_enable();
+	preempt_enable_nort();
 }
 EXPORT_SYMBOL(__kunmap_atomic);
 
 void *kmap_atomic_pfn(unsigned long pfn)
 {
+	pte_t pte = pfn_pte(pfn, kmap_prot);
 	unsigned long vaddr;
 	int idx, type;
 	struct page *page = pfn_to_page(pfn);
 
-	preempt_disable();
+	preempt_disable_nort();
 	pagefault_disable();
 	if (!PageHighMem(page))
 		return page_address(page);
 
 	type = kmap_atomic_idx_push();
-	idx = FIX_KMAP_BEGIN + type + KM_TYPE_NR * smp_processor_id();
+	idx = fixmap_idx(type);
 	vaddr = __fix_to_virt(idx);
 #ifdef CONFIG_DEBUG_HIGHMEM
 	BUG_ON(!pte_none(get_fixmap_pte(vaddr)));
 #endif
-	set_fixmap_pte(idx, pfn_pte(pfn, kmap_prot));
+#ifdef CONFIG_PREEMPT_RT_FULL
+	current->kmap_pte[type] = pte;
+#endif
+	set_fixmap_pte(idx, pte);
 
 	return (void *)vaddr;
 }
+#if defined CONFIG_PREEMPT_RT_FULL
+void switch_kmaps(struct task_struct *prev_p, struct task_struct *next_p)
+{
+	int i;
+
+	/*
+	 * Clear @prev's kmap_atomic mappings
+	 */
+	for (i = 0; i < prev_p->kmap_idx; i++) {
+		int idx = fixmap_idx(i);
+
+		set_fixmap_pte(idx, __pte(0));
+	}
+	/*
+	 * Restore @next_p's kmap_atomic mappings
+	 */
+	for (i = 0; i < next_p->kmap_idx; i++) {
+		int idx = fixmap_idx(i);
+
+		if (!pte_none(next_p->kmap_pte[i]))
+			set_fixmap_pte(idx, next_p->kmap_pte[i]);
+	}
+}
+#endif
```
### clock source

/drivers/clocksource/Kconfig b/drivers/clocksource/Kconfig
```c
--- a/drivers/clocksource/Kconfig
+++ b/drivers/clocksource/Kconfig
@@ -429,6 +429,13 @@ config ATMEL_TCB_CLKSRC
 	help
 	  Support for Timer Counter Blocks on Atmel SoCs.
 
+config ATMEL_TCB_CLKSRC_USE_SLOW_CLOCK
+	bool "TC Block use 32 KiHz clock"
+	depends on ATMEL_TCB_CLKSRC
+	default y
+	help
+	  Select this to use 32 KiHz base clock rate as TC block clock.
+
 config CLKSRC_EXYNOS_MCT
 	bool "Exynos multi core timer driver" if COMPILE_TEST
 	depends on ARM || ARM64
diff --git a/drivers/clocksource/timer-atmel-tcb.c b/drivers/clocksource/timer-atmel-tcb.c
```
/drivers/clocksource/timer-atmel-tcb.c
```c
@@ -27,8 +27,7 @@
  *     this 32 bit free-running counter. the second channel is not used.
  *
  *   - The third channel may be used to provide a 16-bit clockevent
- *     source, used in either periodic or oneshot mode.  This runs
- *     at 32 KiHZ, and can handle delays of up to two seconds.
+ *     source, used in either periodic or oneshot mode.
  *
  * REVISIT behavior during system suspend states... we should disable
  * all clocks and save the power.  Easily done for clockevent devices,
@@ -130,6 +129,8 @@ static u64 notrace tc_sched_clock_read32(void)
 struct tc_clkevt_device {
 	struct clock_event_device	clkevt;
 	struct clk			*clk;
+	bool				clk_enabled;
+	u32				freq;
 	void __iomem			*regs;
 };
 
@@ -138,15 +139,26 @@ static struct tc_clkevt_device *to_tc_clkevt(struct clock_event_device *clkevt)
 	return container_of(clkevt, struct tc_clkevt_device, clkevt);
 }
 
-/* For now, we always use the 32K clock ... this optimizes for NO_HZ,
- * because using one of the divided clocks would usually mean the
- * tick rate can never be less than several dozen Hz (vs 0.5 Hz).
- *
- * A divided clock could be good for high resolution timers, since
- * 30.5 usec resolution can seem "low".
- */
 static u32 timer_clock;
 
+static void tc_clk_disable(struct clock_event_device *d)
+{
+	struct tc_clkevt_device *tcd = to_tc_clkevt(d);
+
+	clk_disable(tcd->clk);
+	tcd->clk_enabled = false;
+}
+
+static void tc_clk_enable(struct clock_event_device *d)
+{
+	struct tc_clkevt_device *tcd = to_tc_clkevt(d);
+
+	if (tcd->clk_enabled)
+		return;
+	clk_enable(tcd->clk);
+	tcd->clk_enabled = true;
+}
+
 static int tc_shutdown(struct clock_event_device *d)
 {
 	struct tc_clkevt_device *tcd = to_tc_clkevt(d);
@@ -154,8 +166,14 @@ static int tc_shutdown(struct clock_event_device *d)
 
 	writel(0xff, regs + ATMEL_TC_REG(2, IDR));
 	writel(ATMEL_TC_CLKDIS, regs + ATMEL_TC_REG(2, CCR));
+	return 0;
+}
+
+static int tc_shutdown_clk_off(struct clock_event_device *d)
+{
+	tc_shutdown(d);
 	if (!clockevent_state_detached(d))
-		clk_disable(tcd->clk);
+		tc_clk_disable(d);
 
 	return 0;
 }
@@ -168,9 +186,9 @@ static int tc_set_oneshot(struct clock_event_device *d)
 	if (clockevent_state_oneshot(d) || clockevent_state_periodic(d))
 		tc_shutdown(d);
 
-	clk_enable(tcd->clk);
+	tc_clk_enable(d);
 
-	/* slow clock, count up to RC, then irq and stop */
+	/* count up to RC, then irq and stop */
 	writel(timer_clock | ATMEL_TC_CPCSTOP | ATMEL_TC_WAVE |
 		     ATMEL_TC_WAVESEL_UP_AUTO, regs + ATMEL_TC_REG(2, CMR));
 	writel(ATMEL_TC_CPCS, regs + ATMEL_TC_REG(2, IER));
@@ -190,12 +208,12 @@ static int tc_set_periodic(struct clock_event_device *d)
 	/* By not making the gentime core emulate periodic mode on top
 	 * of oneshot, we get lower overhead and improved accuracy.
 	 */
-	clk_enable(tcd->clk);
+	tc_clk_enable(d);
 
-	/* slow clock, count up to RC, then irq and restart */
+	/* count up to RC, then irq and restart */
 	writel(timer_clock | ATMEL_TC_WAVE | ATMEL_TC_WAVESEL_UP_AUTO,
 		     regs + ATMEL_TC_REG(2, CMR));
-	writel((32768 + HZ / 2) / HZ, tcaddr + ATMEL_TC_REG(2, RC));
+	writel((tcd->freq + HZ / 2) / HZ, tcaddr + ATMEL_TC_REG(2, RC));
 
 	/* Enable clock and interrupts on RC compare */
 	writel(ATMEL_TC_CPCS, regs + ATMEL_TC_REG(2, IER));
@@ -221,9 +239,13 @@ static struct tc_clkevt_device clkevt = {
 		.features		= CLOCK_EVT_FEAT_PERIODIC |
 					  CLOCK_EVT_FEAT_ONESHOT,
 		/* Should be lower than at91rm9200's system timer */
+#ifdef CONFIG_ATMEL_TCB_CLKSRC_USE_SLOW_CLOCK
 		.rating			= 125,
+#else
+		.rating			= 200,
+#endif
 		.set_next_event		= tc_next_event,
-		.set_state_shutdown	= tc_shutdown,
+		.set_state_shutdown	= tc_shutdown_clk_off,
 		.set_state_periodic	= tc_set_periodic,
 		.set_state_oneshot	= tc_set_oneshot,
 	},
@@ -243,8 +265,11 @@ static irqreturn_t ch2_irq(int irq, void *handle)
 	return IRQ_NONE;
 }
 
-static int __init setup_clkevents(struct atmel_tc *tc, int clk32k_divisor_idx)
+static const u8 atmel_tcb_divisors[5] = { 2, 8, 32, 128, 0, };
+
+static int __init setup_clkevents(struct atmel_tc *tc, int divisor_idx)
 {
+	unsigned divisor = atmel_tcb_divisors[divisor_idx];
 	int ret;
 	struct clk *t2_clk = tc->clk[2];
 	int irq = tc->irq[2];
@@ -265,7 +290,11 @@ static int __init setup_clkevents(struct atmel_tc *tc, int clk32k_divisor_idx)
 	clkevt.regs = tc->regs;
 	clkevt.clk = t2_clk;
 
-	timer_clock = clk32k_divisor_idx;
+	timer_clock = divisor_idx;
+	if (!divisor)
+		clkevt.freq = 32768;
+	else
+		clkevt.freq = clk_get_rate(t2_clk) / divisor;
 
 	clkevt.clkevt.cpumask = cpumask_of(0);
 
@@ -276,7 +305,7 @@ static int __init setup_clkevents(struct atmel_tc *tc, int clk32k_divisor_idx)
 		return ret;
 	}
 
-	clockevents_config_and_register(&clkevt.clkevt, 32768, 1, 0xffff);
+	clockevents_config_and_register(&clkevt.clkevt, clkevt.freq, 1, 0xffff);
 
 	return ret;
 }
@@ -333,8 +362,6 @@ static void __init tcb_setup_single_chan(struct atmel_tc *tc, int mck_divisor_id
 	writel(ATMEL_TC_SYNC, tcaddr + ATMEL_TC_BCR);
 }
 
-static const u8 atmel_tcb_divisors[5] = { 2, 8, 32, 128, 0, };
-
 static const struct of_device_id atmel_tcb_of_match[] = {
 	{ .compatible = "atmel,at91rm9200-tcb", .data = (void *)16, },
 	{ .compatible = "atmel,at91sam9x5-tcb", .data = (void *)32, },
@@ -452,7 +479,11 @@ static int __init tcb_clksrc_init(struct device_node *node)
 		goto err_disable_t1;
 
 	/* channel 2:  periodic and oneshot timer support */
+#ifdef CONFIG_ATMEL_TCB_CLKSRC_USE_SLOW_CLOCK
 	ret = setup_clkevents(&tc, clk32k_divisor_idx);
+#else
+	ret = setup_clkevents(&tc, best_divisor_idx);
+#endif
 	if (ret)
 		goto err_unregister_clksrc;

```

### DEFINE_LOCAL_IRQ_LOCK
```c
--- a/drivers/connector/cn_proc.c
+++ b/drivers/connector/cn_proc.c
@@ -18,6 +18,7 @@
 #include <linux/pid_namespace.h>
 
 #include <linux/cn_proc.h>
+#include <linux/locallock.h>
 
 /*
  * Size of a cn_msg followed by a proc_event structure.  Since the
@@ -40,10 +41,11 @@ static struct cb_id cn_proc_event_id = { CN_IDX_PROC, CN_VAL_PROC };
 
 /* proc_event_counts is used as the sequence number of the netlink message */
 static DEFINE_PER_CPU(__u32, proc_event_counts) = { 0 };
+static DEFINE_LOCAL_IRQ_LOCK(send_msg_lock);
 
 static inline void send_msg(struct cn_msg *msg)
 {
-	preempt_disable();
+	local_lock(send_msg_lock);
 
 	msg->seq = __this_cpu_inc_return(proc_event_counts) - 1;
 	((struct proc_event *)msg->data)->cpu = smp_processor_id();
@@ -56,7 +58,7 @@ static inline void send_msg(struct cn_msg *msg)
 	 */
 	cn_netlink_send(msg, 0, CN_IDX_PROC, GFP_NOWAIT);
 
-	preempt_enable();
+	local_unlock(send_msg_lock);
 }
```


### seqlock
/drivers/dma-buf/reservation.c
```c
@@ -110,15 +110,13 @@ int reservation_object_reserve_shared(struct reservation_object *obj,
 	new->shared_count = j;
 	new->shared_max = max;
 
-	preempt_disable();
-	write_seqcount_begin(&obj->seq);
+	write_seqlock(&obj->seq);
 	/*
 	 * RCU_INIT_POINTER can be used here,
 	 * seqcount provides the necessary barriers
 	 */
 	RCU_INIT_POINTER(obj->fence, new);
-	write_seqcount_end(&obj->seq);
-	preempt_enable();
+	write_sequnlock(&obj->seq);
 
 	if (!old)
 		return 0;
@@ -158,8 +156,7 @@ void reservation_object_add_shared_fence(struct reservation_object *obj,
 	fobj = reservation_object_get_list(obj);
 	count = fobj->shared_count;
 
-	preempt_disable();
-	write_seqcount_begin(&obj->seq);
+	write_seqlock(&obj->seq);
 
 	for (i = 0; i < count; ++i) {
 		struct dma_fence *old_fence;
@@ -181,8 +178,7 @@ void reservation_object_add_shared_fence(struct reservation_object *obj,
 	/* pointer update must be visible before we extend the shared_count */
 	smp_store_mb(fobj->shared_count, count);
 
-	write_seqcount_end(&obj->seq);
-	preempt_enable();
+	write_sequnlock(&obj->seq);
 }
 EXPORT_SYMBOL(reservation_object_add_shared_fence);
```



### raw_spin_lock

 在2.6.33之后的版本，内核加入了raw_spin_lock系列，使用方法和spin_lock系列一模一样，只是参数有spinlock_t变为了raw_spinlock_t 。而且在内核的主线版本中，spin_lock系列只是简单地调用了raw_spin_lock系列的函数，但内核的代码却是有的地方使用spin_lock，有的地方使用raw_spin_lock。是不是很奇怪？要解答这个问题，我们要回到2004年，MontaVista Software, Inc的开发人员在邮件列表中提出来一个Real-Time Linux Kernel的模型，旨在提升Linux的实时性，之后Ingo Molnar很快在他的一个项目中实现了这个模型，并最终产生了一个Real-Time preemption的patch。
该模型允许在临界区中被抢占，而且申请临界区的操作可以导致进程休眠等待，这将导致自旋锁的机制被修改，由原来的整数原子操作变更为信号量操作。当时内核中已经有大约10000处使用了自旋锁的代码，直接修改spin_lock将会导致这个patch过于庞大，于是，他们决定只修改哪些真正不允许抢占和休眠的地方，而这些地方只有100多处，这些地方改为使用raw_spin_lock，但是，因为原来的内核中已经有raw_spin_lock这一名字空间，用于代表体系相关的原子操作的实现，于是linus本人建议：

    把原来的raw_spin_lock改为arch_spin_lock；
    把原来的spin_lock改为raw_spin_lock；
    实现一个新的spin_lock；

写到这里不知大家明白了没？对于2.6.33和之后的版本，我的理解是：

    尽可能使用spin_lock；
    绝对不允许被抢占和休眠的地方，使用raw_spin_lock，否则使用spin_lock；
    如果你的临界区足够小，使用raw_spin_lock；

对于没有打上Linux-RT（实时Linux）的patch的系统，spin_lock只是简单地调用raw_spin_lock，实际上他们是完全一样的，如果打上这个patch之后，spin_lock会使用信号量完成临界区的保护工作，带来的好处是同一个CPU可以有多个临界区同时工作，而原有的体系因为禁止抢占的原因，一旦进入临界区，其他临界区就无法运行，新的体系在允许使用同一个临界区的其他进程进行休眠等待，而不是强占着CPU进行自旋操作。 写这篇文章的时候，内核的版本已经是3.3了，主线版本还没有合并Linux-RT的内容，说不定哪天就会合并进来，也为了你的代码可以兼容Linux-RT，最好坚持上面三个原则。
转载自：http://blog.csdn.net/droidphone/article/details/7395983 