# 主 Goroutine的生与死

## 主 Goroutine 的一生

- 代码位于 `src\runtime\proc.go`

``` go
func main() {
	g := getg()

	// Racectx of m0->g0 is used only as the parent of the main goroutine.
	// It must not be used for anything else.
	g.m.g0.racectx = 0

	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

	// An upper limit for max stack size. Used to avoid random crashes
	// after calling SetMaxStack and trying to allocate a stack that is too big,
	// since stackalloc works with 32-bit sizes.
	maxstackceiling = 2 * maxstacksize

	// Allow newproc to start new Ms.
	mainStarted = true

	if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
		// For runtime_syscall_doAllThreadsSyscall, we
		// register sysmon is not ready for the world to be
		// stopped.
    // 启动系统后台监控（定期垃圾回收、抢占调度等等）
		atomic.Store(&sched.sysmonStarting, 1)
		systemstack(func() {
			newm(sysmon, nil, -1)
		})
	}

	// Lock the main goroutine onto this, the main OS thread,
	// during initialization. Most programs won't care, but a few
	// do require certain calls to be made by the main thread.
	// Those can arrange for main.main to run in the main thread
	// by calling runtime.LockOSThread during initialization
	// to preserve the lock.
	lockOSThread()

	if g.m != &m0 {
		throw("runtime.main not on m0")
	}
	m0.doesPark = true

	// Record when the world started.
	// Must be before doInit for tracing init.
	runtimeInitTime = nanotime()
	if runtimeInitTime == 0 {
		throw("nanotime returning zero")
	}

	if debug.inittrace != 0 {
		inittrace.id = getg().goid
		inittrace.active = true
	}
	// 执行runtime_init的一些任务
	doInit(&runtime_inittask) // Must be before defer.

	// Defer unlock so that runtime.Goexit during init does the unlock too.
	needUnlock := true
	defer func() {
		if needUnlock {
			unlockOSThread()
		}
	}()
	// 启动垃圾回收器后台操作
	gcenable()

	main_init_done = make(chan bool)
	if iscgo {
		if _cgo_thread_start == nil {
			throw("_cgo_thread_start missing")
		}
		if GOOS != "windows" {
			if _cgo_setenv == nil {
				throw("_cgo_setenv missing")
			}
			if _cgo_unsetenv == nil {
				throw("_cgo_unsetenv missing")
			}
		}
		if _cgo_notify_runtime_init_done == nil {
			throw("_cgo_notify_runtime_init_done missing")
		}
		// Start the template thread in case we enter Go from
		// a C-created thread and need to create a new thread.
		startTemplateThread()
		cgocall(_cgo_notify_runtime_init_done, nil)
	}
	// 执行用户 main 包中的 init 函数，因为链接器设定运行时不知道 main 包的地址，处理为非间接调用
	doInit(&main_inittask)

	// Disable init tracing after main init done to avoid overhead
	// of collecting statistics in malloc and newproc
	inittrace.active = false

	close(main_init_done)

	needUnlock = false
	unlockOSThread()

	if isarchive || islibrary {
		// A program compiled with -buildmode=c-archive or c-shared
		// has a main, but it is not executed.
		return
	}
  // 执行用户 main 包中的main函数
	fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
	if raceenabled {
		racefini()
	}

	// Make racy client program work: if panicking on
	// another goroutine at the same time as main returns,
	// let the other goroutine finish printing the panic trace.
	// Once it does, it will exit. See issues 3934 and 20018.
	if atomic.Load(&runningPanicDefers) != 0 {
		// Running deferred functions should not take long.
		for c := 0; c < 1000; c++ {
			if atomic.Load(&runningPanicDefers) == 0 {
				break
			}
			Gosched()
		}
	}
	if atomic.Load(&panicking) != 0 {
		gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
	}

	exit(0)
	for {
		var x *int32
		*x = 0
	}
}
```

- 关键步骤
  1. `systemstack` 会运行 `newm(sysmon, nil)` 启动后台监控
  2. `runtime_init` 负责执行运行时的多个初始化函数 `runtime.init`
  3. `gcenable` 启用垃圾回收器
  4. `main_init` 开始执行用户态 `main.init` 函数，这意味着所有的 `main.init` 均在同一个主 Goroutine 中执行
  5. `main_main` 开始执行用户态 `main.main` 函数，这意味着 `main.main` 和 `main.init` 均在同一个 Goroutine 中执行。

- 重要 init 函数

  1. 垃圾回收器所需的参数检查并创建强制启动 GC 的监控 Goroutine

  ``` go
  // 代码位于 `src\runtime\mgcwork.go`
  const (
  	_WorkbufSize = 2048 // in bytes; larger values result in less contention 以字节为单位，较大的值减少竞争
  
  	// workbufAlloc is the number of bytes to allocate at a time
  	// for new workbufs. This must be a multiple of pageSize and
  	// should be a multiple of _WorkbufSize.
  	// workbufAlloc 是一次为新工作缓冲区分配的字节数。这必须是 pageSize 的倍数，并且应该是 _WorkbufSize 的倍数。
  	// Larger values reduce workbuf allocation overhead. Smaller
  	// values reduce heap fragmentation.
    // 较大的值会减少工作缓冲区分配开销。较小的值会减少堆碎片。
  	workbufAlloc = 32 << 10
  )
  func init() {
  	if workbufAlloc%pageSize != 0 || workbufAlloc%_WorkbufSize != 0 {
  		throw("bad workbufAlloc")
  	}
  }
  // 代码位于 `src\runtime\proc.go`
  func init() {
      go forcegchelper()
  }
  ```

  2. 确定 `defer` 的运行时类型：

  ``` go
  // 代码位于 `src\runtime\panic.go`
  var deferType *_type // type of _defer struct
  
  func init() {
  	var x interface{}
  	x = (*_defer)(nil)
  	deferType = (*(**ptrtype)(unsafe.Pointer(&x))).elem
  }
  ```

  

