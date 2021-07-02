# MPG 模型与并发调度单元

- 调度器的三个主要概念
  - G: **G**oroutine，即我们在 Go 程序中使用 `go` 关键字创建的执行体；
  - M: **M**achine，或 worker thread，即传统意义上进程的线程；
  - P: **P**rocessor，即一种人为抽象的、用于执行 Go 代码被要求局部资源。只有当 M 与一个 P 关联后才能执行 Go 代码。除非 M 发生阻塞或在进行系统调用时间过长时，没有与之关联的 P。

## 工作线程的暂止和复始

运行时调度器的任务是给不同的工作线程 (worker thread) 分发可供运行的（ready-to-run）Goroutine。 我们不妨设每个工作线程总是贪心的执行所有存在的 Goroutine，那么当运行进程中存在 n 个线程（M），且 每个 M 在某个时刻有且只能调度一个 G。根据抽屉原理，可以很容易的证明这两条性质：

- 性质 1：当用户态代码创建了 p(p>n)p(p>n) 个 G 时，则必定存在 p−np−n 个 G 尚未被 M 调度执行；
- 性质 2：当用户态代码创建的 q (q < n) 时，则必定存在 n-q 个 M 不存在正在调度的 G。

这两条性质分别决定了工作线程的 **暂止（park）** 和 **复始（unpark）** 。

我们不难发现，调度器的设计需要在性质 1 和性质 2 之间进行权衡： 即既要保持足够的运行工作线程来利用有效硬件并发资源，又要暂止过多的工作线程来节约 CPU 能耗。 如果我们把调度器想象成一个系统，则寻找这个权衡的最优解意味着我们必须求解调度器系统中 每个 M 的状态，即系统的全局状态。这是非常困难的，不妨考虑以下两个难点：

**难点 1: 在多个 M 之间不使用屏障的情况下，得出调度器中多个 M 的全局状态是不可能的。**

我们都知道计算的局部性原理，为了利用这一原理，调度器所需调度的 G 都会被放在每个 M 自身对应的本地队列中。 换句话说，每个 M 都无法直接观察到其他的 M 所具有的 G 的状态，存在多个 M 之间的共识问题。这本质上就是一个分布式系统。 显然，每个 M 都能够连续的获取自身的状态，但当它需要获取整个系统的全局状态时却不容易， 原因在于我们没有一个能够让所有线程都同步的时钟。换句话说， 我们需要依赖屏障来保证多个 M 之间的全局状态同步。更进一步，在不使用屏障的情况下， 能否利用每个 M 在不同时间中记录的本地状态中计算出调度器的全局状态，或者形式化的说： 能否在快速路径（fast path）下计算进程集的全局谓词（global predicates）呢？根据我们在共识技术中的知识，是不可能的。

**难点 2: 为了获得最佳的线程管理，我们必须获得未来的信息，即当一个新的 G 即将就绪（ready）时，则不再暂止一个工作线程。**

举例来说，目前我们的调度器存在 4 个 M，并其中有 3 个 M 正在调度 G，则其中有 1 个 M 处于空闲状态。 这时为了节约 CPU 能耗，我们希望对这个空闲的 M 进行暂止操作。但是，正当我们完成了对此 M 的暂止操作后， 用户态代码正好执行到了需要调度一个新的 G 时，我们又不得不将刚刚暂止的 M 重新启动，这无疑增加了开销。 我们当然有理由希望，如果我们能知晓一个程序生命周期中所有的调度信息， 提前知晓什么时候适合对 M 进行暂止自然再好不过了。 尽管我们能够对程序代码进行静态分析，但这显然是不可能的：考虑一个简单的 Web 服务端程序，每个用户请求 到达后会创建一个新的 G 交于调度器进行调度。但请求到达是一个随机过程，我们只能预测在给定置信区间下 可能到达的请求数，而不能完整知晓所有的调度需求。

那么我们又应该如何设计一个通用且可扩展的调度器呢？我们很容易想到三种平凡的做法：

**设计 1: 集中式管理所有状态**

显然这种做法自然是不可取的，在多个并发实体之间集中管理所有状态这一共享资源，需要锁的支持， 当并发实体的数量增大时，将限制调度器的可扩展性。

**设计 2**: 每当需要就绪一个 G1 时，都让出一个 P，直接切换出 G2，再复始一个 M 来执行 G2。

因为复始的 M 可能在下一个瞬间又没有调度任务，则会发生线程颠簸（thrashing），进而我们又需要暂止这个线程。 另一方面，我们希望在相同的线程内保存维护 G，这种方式还会破坏计算的局部性原理。

**设计 3**: 任何时候当就绪一个 G、也存在一个空闲的 P 时，都复始一个额外的线程，不进行切换。

因为这个额外线程会在没有检查任何工作的情况下立即进行暂止，最终导致大量 M 的暂止和复始行为，产生大量开销。

基于以上考虑，目前的 Go 的调度器实现中设计了工作线程的**自旋（spinning）状态**：

1. 如果一个工作线程的本地队列、全局运行队列或网络轮询器中均没有可调度的任务，则该线程成为自旋线程；
2. 满足该条件、被复始的线程也被称为自旋线程，对于这种线程，运行时不做任何事情。

自旋线程在进行暂止之前，会尝试从任务队列中寻找任务。当发现任务时，则会切换成非自旋状态， 开始执行 Goroutine。而找到不到任务时，则进行暂止。

当一个 Goroutine 准备就绪时，会首先检查自旋线程的数量，而不是去复始一个新的线程。

如果最后一个自旋线程发现工作并且停止自旋时，则复始一个新的自旋线程。 这个方法消除了不合理的线程复始峰值，且同时保证最终的最大 CPU 并行度利用率。

我们可以通过下图来直观理解工作线程的状态转换：

``` 
  如果存在空闲的 P，且存在暂止的 M，并就绪 G
          +------+
          v      |
执行 --> 自旋 --> 暂止
 ^        |
 +--------+
  如果发现工作
```

总的来说，调度器的方式可以概括为： **如果存在一个空闲的 P 并且没有自旋状态的工作线程 M，则当就绪一个 G 时，就复始一个额外的线程 M。** 这个方法消除了不合理的线程复始峰值，且同时保证最终的最大 CPU 并行度利用率。

这种设计的实现复杂性表现在进行自旋与非自旋线程状态转换时必须非常小心。 这种转换在提交一个新的 G 时发生竞争，最终导致任何一个工作线程都需要暂止对方。 如果双方均发生失败，则会以半静态 CPU 利用不足而结束调度。

因此，就绪一个 G 的通用流程为：

- 提交一个 G 到 per-P 的本地工作队列
- 执行 StoreLoad 风格的写屏障
- 检查 `sched.nmspinning` 数量

而从自旋到非自旋转换的一般流程为：

- 减少 `nmspinning` 的数量
- 执行 StoreLoad 风格的写屏障
- 在所有 per-P 本地任务队列检查新的工作

当然，此种复杂性在全局任务队列对全局队列并不适用的，因为当给一个全局队列提交工作时， 不进行线程的复始操作。

## 主要结构

### M 的结构

M 是 OS 线程的实体。我们介绍几个比较重要的字段，包括：

- 持有用于执行调度器的 g0
- 持有用于信号处理的 gsignal
- 持有线程本地存储 tls
- 持有当前正在运行的 curg
- 持有运行 Goroutine 时需要的本地资源 p
- 表示自身的自旋和非自旋状态 spining
- 管理在它身上执行的 cgo 调用
- 将自己与其他的 M 进行串联

等等其他五十多个字段，包括关于 M 的一些调度统计、调试信息等。

``` go
// src/runtime/runtime2.go
type m struct {
	g0          *g			// 用于执行调度指令的 Goroutine
	gsignal     *g			// 处理 signal 的 g
	tls         [tlsSlots]uintptr // thread-local storage (for x86 extern register) 线程本地存储
	curg        *g			// current running goroutine 当前运行的用户 Goroutine
	p           puintptr	// 执行 go 代码时持有的 p (如果没有执行则为 nil)
	spinning    bool		// m 当前没有运行 work 且正处于寻找 work 的活跃状态
	cgoCallers  *cgoCallers	// cgo 调用崩溃的 cgo 回溯
	alllink     *m			// 在 allm 上

	...
}
```

### P的结构

P存在的意义在于实现工作窃取（work stealing）算法。 简单来说，每个 P 持有一个 G 的本地队列。

在没有 P 的情况下，所有的 G 只能放在一个全局的队列中。 当 M 执行完 G 而没有 G 可执行时，必须将队列锁住从而取值。

当引入了 P 之后，P 持有 G 的本地队列，而持有 P 的 M 执行完 G 后在 P 本地队列中没有 发现其他 G 可以执行时，虽然仍然会先检查全局队列、网络，但这时增加了一个从其他 P 的 队列偷取（steal）一个 G 来执行的过程。优先级为本地 > 全局 > 网络 > 偷取。

一个不恰当的比喻：银行服务台排队中身手敏捷的顾客，当一个服务台队列空（没有人）时， 没有在排队的顾客（全局）会立刻跑到该窗口，当彻底没人时在其他队列排队的顾客才会迅速 跑到这个没人的服务台来，即所谓的偷取。

``` go
type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/... p 的状态
	link        puintptr
	schedtick   uint32     // incremented on every scheduler call
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // back-link to associated m (nil if idle) // 反向链接到关联的 m （nil 则表示 idle）
	mcache      *mcache
  pcache      pageCache
	raceprocctx uintptr

	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead 	uint32			// 可运行的 Goroutine 队列，可无锁访问
	runqtail 	uint32
	runq     	[256]guintptr  
}
```

整个结构除去 P 的本地 G 队列外，就是一些统计、调试、GC 辅助的字段了。

### G的结构

```go
type g struct {
	stack struct {
		lo uintptr
		hi uintptr
	} 							// 栈内存：[stack.lo, stack.hi)
	stackguard0	uintptr
	stackguard1 uintptr

	_panic       *_panic
	_defer       *_defer
	m            *m				// 当前的 m
	sched        gobuf
	stktopsp     uintptr		// 期望 sp 位于栈顶，用于回溯检查
	param        unsafe.Pointer // wakeup 唤醒时候传递的参数
	atomicstatus uint32
	goid         int64
	preempt      bool       	// 抢占信号，stackguard0 = stackpreempt 的副本
	timer        *timer         // 为 time.Sleep 缓存的计时器

	...
}
```

### 调度器 `sched` 结构

调度器，所有 Goroutine 被调度的核心，存放了调度器持有的全局资源，访问这些资源需要持有锁：

- 管理了能够将 G 和 M 进行绑定的 M 队列
- 管理了空闲的 P 链表（队列）
- 管理了 G 的全局队列
- 管理了可被复用的 G 的全局缓存
- 管理了 defer 池

``` go
type schedt struct {
	lock mutex

	pidle      puintptr	// 空闲 p 链表
	npidle     uint32	// 空闲 p 数量
	nmspinning uint32	// 自旋状态的 M 的数量
	runq       gQueue	// 全局 runnable G 队列
	runqsize   int32
	gFree struct {		// 有效 dead G 的全局缓存.
		lock    mutex
		stack   gList	// 包含栈的 Gs
		noStack gList	// 没有栈的 Gs
		n       int32
	}
	sudoglock  mutex	// sudog 结构的集中缓存
	sudogcache *sudog
	deferlock  mutex	// 不同大小的有效的 defer 结构的池
	deferpool  [5]*_defer
	
	...
}
```

- 调度器初始化

``` go
// runtime/proc.go
func schedinit() {
  (...)
	_g_ := getg()
	(...)

	// M 初始化
	mcommoninit(_g_.m)
	(...)

	// P 初始化
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
	(...)
}
```

``` go
// runtime/asm_amd64.s
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	(...)
	CALL	runtime·schedinit(SB) // M, P 初始化
	MOVQ	$runtime·mainPC(SB), AX
	PUSHQ	AX
	PUSHQ	$0
	CALL	runtime·newproc(SB) // G 初始化
	POPQ	AX
	POPQ	AX
	(...)
	RET

DATA	runtime·mainPC+0(SB)/8,$runtime·main(SB)
GLOBL	runtime·mainPC(SB),RODATA,$8
```

