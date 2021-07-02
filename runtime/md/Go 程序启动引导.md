# Go 程序启动引导

## 入口

- 程序入口，以 AMD64 架构上的 Linux 和 macOS 为例，分别位于：`src/runtime/rt0_linux_amd64.s` 和 `src/runtime/rt0_darwin_amd64.s`

``` c
// macOS多了一些注释
#include "textflag.h"

TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)

TEXT _rt0_amd64_linux_lib(SB),NOSPLIT,$0
	JMP	_rt0_amd64_lib(SB)
```

- 均跳转到了 `_rt0_amd64` 函数

## 入口详情(参数)

- 位于：`src/runtime/asm_arm.s` 
- 栈指针 SP 的前两个值分别对应 `argc` 和 `argv`

``` c
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)
```

``` c
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
	// copy arguments forward on an even stack 将参数向前复制到一个偶数栈上
	MOVQ	DI, AX		// argc
	MOVQ	SI, BX		// argv
	SUBQ	$(4*8+7), SP		// 2args 2auto
	ANDQ	$~15, SP
	MOVQ	AX, 16(SP)
	MOVQ	BX, 24(SP)

	// create istack out of the given (operating system) stack.
	// _cgo_init may update stackguard.
  // 初始化 g0 执行栈
	MOVQ	$runtime·g0(SB), DI
	LEAQ	(-64*1024+104)(SP), BX
	MOVQ	BX, g_stackguard0(DI)
	MOVQ	BX, g_stackguard1(DI)
	MOVQ	BX, (g_stack+stack_lo)(DI)
	MOVQ	SP, (g_stack+stack_hi)(DI)

	// find out information about the processor we're on
  // 确定 CPU 处理器的信息
	MOVL	$0, AX
	CPUID
	MOVL	AX, SI
	CMPL	AX, $0
	JE	nocpuinfo
  (...) // 针对各种操作系统做处理
```

## 线程本地存储TLS

- 代码位置同上

``` c
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
  (...) // 入口参数以及 CPU 处理器信息
#ifdef GOOS_darwin
	// skip TLS setup on Darwin 在 Darwin 系统上跳过 TLS 设置
	JMP ok
#endif
#ifdef GOOS_openbsd
	// skip TLS setup on OpenBSD 在 OpenBSD 系统上跳过 TLS 设置
	JMP ok
#endif
  
  LEAQ	runtime·m0+m_tls(SB), DI	// DI = m0.tls
	CALL	runtime·settls(SB)				// 将 TLS 地址设置到 DI

	// store through it, to make sure it works 使用它进行存储，确保正常工作
	get_tls(BX)
	MOVQ	$0x123, g(BX)
	MOVQ	runtime·m0+m_tls(SB), AX
	CMPQ	AX, $0x123								// 判断 TLS 是否设置成功
	JEQ 2(PC)												// 如果相等则向后跳转两条指令
	CALL	runtime·abort(SB)					// 使用 INT 指令执行中断
ok:
	// set the per-goroutine and per-mach "registers"
  // 程序刚刚启动，此时位于主线程
	// 当前栈与资源保存在 g0
	// 该线程保存在 m0
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)
```

- `g0` 和 `m0` 是一组全局变量，在程序运行之初就已经存在。除了程序参数外，会首先将 m0 与 g0 通过指针互相关联。

## 早期校验与系统级初始化

- 在正式初始化运行时组件之前，还需要做一些校验和系统级的初始化工作，这包括：运行时类型检查， 系统参数的获取以及影响内存管理和程序调度的相关常量的初始化。

``` c
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
  (...)
  CALL	runtime·check(SB)

	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)
  (...)
```

1. 运行时类型检查，位于`src/runtime/runtime1.go` 

``` go
func check() {
	var (
		a     int8
		b     uint8
    (...) // 一系列类型
	)
  (...)

	if unsafe.Sizeof(a) != 1 {
		throw("bad a")
	}
	if unsafe.Sizeof(b) != 1 {
		throw("bad b")
	}
  (...) // 一系列判断
}
```

2. 系统参数、处理器与内存常量，位于`src/runtime/runtime1.go` 

``` go
func args(c int32, v **byte) {
	argc = c
	argv = v
	sysargs(c, v)
}
```

3. Darwin等操作系统不关注

## 运行时组件核心

``` c
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	(...)
	// 调度器初始化
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program 创建一个新的 goroutine 来启动程序
	MOVQ	$runtime·mainPC(SB), AX
	PUSHQ	AX
	PUSHQ	$0			// 参数大小
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// 启动这个 M，mstart 应该永不返回
	CALL	runtime·mstart(SB)
	(...)
	RET
```

1. `schedinit`：进行各种运行时组件初始化工作，这包括我们的调度器与内存分配器、回收器的初始化
2. `newproc`：负责根据主 goroutine （即 `main`）入口地址创建可被运行时调度的执行单元
3. `mstart`：开始启动调度器的调度循环

``` go
// 代码位于`src/runtime/proc.go` 
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
func schedinit() {
	lockInit(&sched.lock, lockRankSched)
	lockInit(&sched.sysmonlock, lockRankSysmon)
	lockInit(&sched.deferlock, lockRankDefer)
	lockInit(&sched.sudoglock, lockRankSudog)
	lockInit(&deadlock, lockRankDeadlock)
	lockInit(&paniclk, lockRankPanic)
	lockInit(&allglock, lockRankAllg)
	lockInit(&allpLock, lockRankAllp)
	lockInit(&reflectOffs.lock, lockRankReflectOffs)
	lockInit(&finlock, lockRankFin)
	lockInit(&trace.bufLock, lockRankTraceBuf)
	lockInit(&trace.stringsLock, lockRankTraceStrings)
	lockInit(&trace.lock, lockRankTrace)
	lockInit(&cpuprof.lock, lockRankCpuprof)
	lockInit(&trace.stackTab.lock, lockRankTraceStackTab)
	// Enforce that this lock is always a leaf lock. 强制此锁始终为叶锁
	// All of this lock's critical sections should be 
	// extremely short. 这个锁的所有临界区都应该很短
	lockInit(&memstats.heapStats.noPLock, lockRankLeafRank)

	// raceinit must be the first call to race detector.
	// In particular, it must be done before mallocinit below calls racemapshadow.
	_g_ := getg()
	if raceenabled {
		_g_.racectx, raceprocctx0 = raceinit()
	}
	// 栈、内存分配器、调度器相关初始化
	sched.maxmcount = 10000		// 限制最大系统线程数量

	// The world starts stopped.
	worldStopped()

	moduledataverify()
	stackinit()								// 初始化执行栈
	mallocinit()							// 初始化内存分配器
	fastrandinit() // must run before mcommoninit
	mcommoninit(_g_.m, -1)		// 初始化当前系统线程
	cpuinit()       // must run before alginit
	alginit()       // maps must not be used before this call
	modulesinit()   // provides activeModules
	typelinksinit() // uses maps, activeModules
	itabsinit()     // uses activeModules

	sigsave(&_g_.m.sigmask)
	initSigmask = _g_.m.sigmask

	if offset := unsafe.Offsetof(sched.timeToRun); offset%8 != 0 {
		println(offset)
		throw("sched.timeToRun not aligned to 8 bytes")
	}

	goargs()
	goenvs()
	parsedebugvars()
	gcinit()				// 垃圾回收器初始化

	lock(&sched.lock)
	sched.lastpoll = uint64(nanotime())
  // 创建 P
	// 通过 CPU 核心数和 GOMAXPROCS 环境变量确定 P 的数量
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
	unlock(&sched.lock)

	// World is effectively started now, as P's can run.
	worldStarted()

	// For cgocheck > 1, we turn on the write barrier at all times
	// and check all pointer writes. We can't do this until after
	// procresize because the write barrier needs a P.
	if debug.cgocheck > 1 {
		writeBarrier.cgo = true
		writeBarrier.enabled = true
		for _, p := range allp {
			p.wbBuf.reset()
		}
	}

	if buildVersion == "" {
		// Condition should never trigger. This code just serves
		// to ensure runtime·buildVersion is kept in the resulting binary.
		buildVersion = "unknown"
	}
	if len(modinfo) == 1 {
		// Condition should never trigger. This code just serves
		// to ensure runtime·modinfo is kept in the resulting binary.
		modinfo = ""
	}
}
```

- `stackinit()` goroutine 执行栈初始化
- `mallocinit()` 内存分配器初始化
- `mcommoninit()` 系统线程的部分初始化工作
- `gcinit()` 垃圾回收器初始化
- `procresize()` 根据 CPU 核心数，初始化系统线程的本地缓存
