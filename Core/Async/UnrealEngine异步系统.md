## Unreal Engine中的异步系统

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Unreal Engine中的异步系统](#unreal-engine中的异步系统)
  - [类图Overall](#类图overall)
  - [FRunnableThread 和 FRunnable实现](#frunnablethread-和-frunnable实现)
    - [FRunnableThread](#frunnablethread)
    - [FRunnable相关实现](#frunnable相关实现)
    - [FThreadManager](#fthreadmanager)
  - [异步任务的实现](#异步任务的实现)
    - [队列化任务的实现](#队列化任务的实现)
      - [FQueuedThread](#fqueuedthread)
      - [FQueuedThreadPoolBase](#fqueuedthreadpoolbase)
    - [TaskGraph的实现](#taskgraph的实现)
    - [FTaskGraphInterface](#ftaskgraphinterface)
      - [FTaskGraphImplementation](#ftaskgraphimplementation)
    - [FTaskThreadBase](#ftaskthreadbase)
      - [FTaskThreadAnyThread](#ftaskthreadanythread)
      - [FNamedTaskThread](#fnamedtaskthread)
    - [FBaseGraphTask](#fbasegraphtask)
      - [模板参数TTask](#模板参数ttask)
      - [FConstructor](#fconstructor)
    - [FGraphEvent](#fgraphevent)
    - [TaskGraph运行流程](#taskgraph运行流程)

<!-- /code_chunk_output -->


### 类图Overall
```plantuml
@startuml
class FRunnableThread
{
    {static} + FRunnableThread* Create(class FRunnable* InRunnable, const TCHAR* ThreadName ...)

    # virtual bool CreateInternal( FRunnable* InRunnable, const TCHAR* InThreadName, uint32 InStackSize = 0 ...)

    # FRunnable* Runnable
    # FEvent* ThreadInitSyncEvent
    # EThreadPriority ThreadPriority
    # uint64 ThreadAffinityMask
}

note left of FRunnableThread::Create
根据平台创建一个线程，并加入到FThreadManager中管理
end note

class FRunnableThreadWin
{
    - HANDLE Thread
}
FRunnableThread <|-- FRunnableThreadWin

note right of FRunnableThreadWin::Thread
windows平台上，线程对应句柄
end note

class FRunnableThreadPThread
{
    - pthread_t Thread
}
FRunnableThread <|-- FRunnableThreadPThread

note left of FRunnableThreadPThread::Thread
使用POSIX Thread实现的线程的句柄
end note

class FThreadManager
{
    - FCriticalSection ThreadsCritical
    - TMap<uint32, FRunnableThread*> Threads

    + void AddThread(uint32 ThreadId, FRunnableThread* Thread)
    + void RemoveThread(FRunnableThread* Thread)
    + void Tick()
}

FThreadManager *-- FRunnableThread

abstract class FRunnable
{
    - virtual uint32 Run()
}

FRunnable *-- FRunnableThread

abstract class IQueuedWork
{
    {abstract} - virtual void DoThreadedWork()
    {abstract} - virtual void Abandon()
}

note right of IQueuedWork::DoThreadedWork
实际需要完成的工作，需要重写该方法来添加自己需要的内容
end note

class FQueuedThread
{
    # FEvent* DoWorkEvent
    # TAtomic<bool> TimeToDie
    # IQueuedWork* volatile QueuedWork
    # class FQueuedThreadPoolBase* OwningThreadPool
    # FRunnableThread* Thread

    # virtual uint32 Run()

    + virtual bool Create(class FQueuedThreadPoolBase* InPool, uint32 InStackSize = 0, EThreadPriority ThreadPriority = TPri_Normal, const TCHAR* ThreadName = nullptr)

    + bool KillThread()
    + void DoWork(IQueuedWork* InQueuedWork)
}

FRunnable <|-- FQueuedThread
IQueuedWork *-- FQueuedThread
FRunnableThread *-- FQueuedThread

abstract class FQueuedThreadPool
{
    {abstract} + virtual bool Create()
    {abstract} + virtual void Destroy()
    {abstract} + virtual void AddQueuedWork()
    {abstract} + virtual void RetractQueuedWork()
    {abstract} + virtual int32 GetNumThreads()
}

class FQueuedThreadPoolBase
{
    # FThreadPoolPriorityQueue QueuedWork
    # TArray<FQueuedThread*> QueuedThreads
    # TArray<FQueuedThread*> AllThreads
    # FCriticalSection* SynchQueue
    # bool TimeToDie

    + virtual bool Create()
    + virtual void Destroy()
    + virtual void AddQueuedWork()
    + virtual void RetractQueuedWork()
    + virtual int32 GetNumThreads() 
}

FQueuedThreadPool <|-- FQueuedThreadPoolBase
FQueuedThread *-- FQueuedThreadPoolBase
IQueuedWork *-- FQueuedThreadPoolBase

note left of FQueuedThreadPoolBase::QueuedThreads
当前线程池中空闲的线程
end note

note left of FQueuedThreadPoolBase::QueuedWork
线程池中还没有被运行的任务，会根据Prriority进行排序
end note

@enduml
```

### FRunnableThread 和 FRunnable实现
```plantuml
@startuml
class FRunnableThread
{
    {static} + FRunnableThread* Create(class FRunnable* InRunnable, const TCHAR* ThreadName ...)

}

note left of FRunnableThread::Create
根据平台创建一个线程，并加入到FThreadManager中管理
end note

class FRunnableThreadWin
{
    - HANDLE Thread
}
FRunnableThread <|-- FRunnableThreadWin

note left of FRunnableThreadWin::Thread
windows平台上，线程对应句柄
end note

class FRunnableThreadPThread
{
    - pthread_t Thread
}
FRunnableThread <|-- FRunnableThreadPThread

note right of FRunnableThreadPThread::Thread
使用POSIX Thread实现的线程的句柄
end note

@enduml
```
#### FRunnableThread
FRunnableThread 是UE中所有线程的基类，根据不同的平台有不同的实现。例如在Windows平台的FRunnableThreadWin，以及通过pthread实现的FRunnableThreadPThread等等。
线程在创建成功之后，会先加入到FThreadManager之中，然后再调用FRunnableThread::Run函数，而后会调用FRunnable::Run
```cpp
// Windows 平台相关的实现
static ::DWORD STDCALL _ThreadProc(LPVOID pThis)
{
	check(pThis);
	auto* ThisThread = (FRunnableThreadWin*)pThis;
	FThreadManager::Get().AddThread(ThisThread->GetThreadID(), ThisThread);
	return ThisThread->GuardedRun();
}

uint32 FRunnableThreadWin::GuardedRun()
{
	uint32 ExitCode = 0;

	FPlatformProcess::SetThreadAffinityMask(ThreadAffinityMask);

	FPlatformProcess::SetThreadName(*ThreadName);
	const TCHAR* CmdLine = ::GetCommandLineW();
	bool bNoExceptionHandler = FParse::Param(::GetCommandLineW(), TEXT("noexceptionhandler"));
#if UE_BUILD_DEBUG
	if (true && !GAlwaysReportCrash)
#else
	if (bNoExceptionHandler || (FPlatformMisc::IsDebuggerPresent() && !GAlwaysReportCrash))
#endif // UE_BUILD_DEBUG
	{
		ExitCode = Run();
	}
	else
	{
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try
#endif // !PLATFORM_SEH_EXCEPTIONS_DISABLED
		{
			ExitCode = Run();
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except (FPlatformMisc::GetCrashHandlingType() == ECrashHandlingType::Default ? ReportCrash(GetExceptionInformation()) : EXCEPTION_CONTINUE_SEARCH)
		{
			__try
			{
				// Make sure the information which thread crashed makes it into the log.
				UE_LOG( LogThreadingWindows, Error, TEXT( "Runnable thread %s crashed." ), *ThreadName );
				GWarn->Flush();

				// Append the thread name at the end of the error report.
				FCString::Strncat( GErrorHist, TEXT(LINE_TERMINATOR_ANSI "Crash in runnable thread " ), UE_ARRAY_COUNT( GErrorHist ) );
				FCString::Strncat( GErrorHist, *ThreadName, UE_ARRAY_COUNT( GErrorHist ) );

				// Crashed.
				ExitCode = 1;
				GError->HandleError();
				FPlatformMisc::RequestExit(true, TEXT("FRunnableThreadWin::GuardedRun.ExceptionHandler"));
			}
			__except(EXCEPTION_EXECUTE_HANDLER)
			{
				// The crash handler crashed itself, exit with a code which the 
				// out-of-process monitor will be able to pick up and report into 
				// analytics.

				::exit(ECrashExitCodes::CrashHandlerCrashed);
			}
		}
#endif // !PLATFORM_SEH_EXCEPTIONS_DISABLED
	}

	return ExitCode;
}

uint32 FRunnableThreadWin::Run()
{
	uint32 ExitCode = 1;
	check(Runnable);

	if (Runnable->Init() == true)
	{
		ThreadInitSyncEvent->Trigger();

		// Setup TLS for this thread, used by FTlsAutoCleanup objects.
		SetTls();

		ExitCode = Runnable->Run();

		// Allow any allocated resources to be cleaned up
		Runnable->Exit();

#if STATS
		FThreadStats::Shutdown();
#endif
		FreeTls();
	}
	else
	{
		// Initialization has failed, release the sync event
		ThreadInitSyncEvent->Trigger();
	}

	return ExitCode;
}
```

**FRunnableThread::Create**中会调用FRunnableThread::CreateInternal函数，这个函数在不同的平台有不同的实现，从而创建平台相关的HAL
```cpp
FRunnableThread* FRunnableThread::Create(
	class FRunnable* InRunnable, 
	const TCHAR* ThreadName,
	uint32 InStackSize,
	EThreadPriority InThreadPri, 
	uint64 InThreadAffinityMask,
	EThreadCreateFlags InCreateFlags)
{
	bool bCreateRealThread = FPlatformProcess::SupportsMultithreading();

	FRunnableThread* NewThread = nullptr;

	if (bCreateRealThread)
	{
		check(InRunnable);
		// Create a new thread object
		NewThread = FPlatformProcess::CreateRunnableThread();
	}
	else if (InRunnable->GetSingleThreadInterface())
	{
		// Create a fake thread when multithreading is disabled.
		NewThread = new FFakeThread();
	}

	if (NewThread)
	{
		SetupCreatedThread(NewThread, InRunnable, ThreadName, InStackSize, InThreadPri, InThreadAffinityMask, InCreateFlags);
	}

	return NewThread;
}

void FRunnableThread::SetupCreatedThread(FRunnableThread*& NewThread, class FRunnable* InRunnable,  const TCHAR* ThreadName, uint32 InStackSize, EThreadPriority InThreadPri, uint64 InThreadAffinityMask, EThreadCreateFlags InCreateFlags)
{
	// Call the thread's create method
	bool bIsValid = NewThread->CreateInternal(InRunnable, ThreadName, InStackSize, InThreadPri, InThreadAffinityMask, InCreateFlags);

	if( bIsValid )
	{
		check(NewThread->Runnable);
		NewThread->PostCreate(InThreadPri);
	}
	else
	{
		// We failed to start the thread correctly so clean up
		delete NewThread;
		NewThread = nullptr;
	}
}
```

#### FRunnable相关实现
FRunnable是一个可以运行在线程上的任务的接口，可以通过继承他来实现一个可以在线程上运行的任务，他不能独立的存在，需要依赖真正的线程来执行任务。FRunableThread在运行时，最终会调用到FRunnable::Run。FRunnable的子类有很多，下面列举了一些：
```plantuml
@startuml
class FRunnbale{}
class FRenderingThread {}
class FRHIThread {}
class FTaskThreadBase {}
class FQueudedThread {}
class TAsyncRunnable {}

FRunnbale <|-- FRenderingThread
FRunnbale <|-- FRHIThread
FRunnbale <|-- FTaskThreadBase
FRunnbale <|-- FQueudedThread
FRunnbale <|-- TAsyncRunnable

@enduml
```

#### FThreadManager
FThreadManager是UE中用于管理线程的类，游戏中创建出来的线程都会加入该管理器中进行管理。该类的功能比较简单，使用一个TMap保存了创建出来的线程，使用ThreadID当Key。另外还会Tick线程中的FFakeThread，FFakeThread是一种假的线程，实际上还是单线程，并没有在一个独立的线程中执行。
```plantuml
@startuml
class FThreadManager
{
    - FCriticalSection ThreadsCritical
    - TMap<uint32, FRunnableThread*> Threads

    + void AddThread(uint32 ThreadId, FRunnableThread* Thread)
    + void RemoveThread(FRunnableThread* Thread)
    + void Tick()
}

note left of FThreadManager::Tick
用来更新Fake Thread
end note

@enduml
```


### 异步任务的实现

#### 队列化任务的实现
UE中的队列化任务主要有以下几个接口和类来实现：
- IQueuedWork是可以被运行在队列化的线程池中的线程执行的任务的基类。和FRunnable一样，它不能独立的执行自己的任务，需要和相应的线程来配合。
- FQueuedThread，包含了需要执行的任务IQueuedWork和执行任务的线程FRunnableThread，它继承于FRunnable，是真正执行IQueuedWork任务的类。
- FQueuedThreadPool是队列化线程池的基类，定义了线程池所需要的接口。FQueuedThreadPoolBase是UE实现的默认的线程池的版本。

```plantuml
@startuml
abstract class IQueuedWork
{
    {abstract} - virtual void DoThreadedWork()
    {abstract} - virtual void Abandon()
}

note right of IQueuedWork::DoThreadedWork
实际需要完成的工作，需要重写该方法来添加自己需要的内容
end note

class FQueuedThread
{
    # FEvent* DoWorkEvent
    # TAtomic<bool> TimeToDie
    # IQueuedWork* volatile QueuedWork
    # class FQueuedThreadPoolBase* OwningThreadPool
    # FRunnableThread* Thread

    # virtual uint32 Run()

    + virtual bool Create(class FQueuedThreadPoolBase* InPool, uint32 InStackSize = 0, EThreadPriority ThreadPriority = TPri_Normal, const TCHAR* ThreadName = nullptr)

    + bool KillThread()
    + void DoWork(IQueuedWork* InQueuedWork)
}

FRunnable <|-- FQueuedThread
IQueuedWork *-- FQueuedThread
FRunnableThread *-- FQueuedThread

abstract class FQueuedThreadPool
{
    {abstract} + virtual bool Create()
    {abstract} + virtual void Destroy()
    {abstract} + virtual void AddQueuedWork()
    {abstract} + virtual void RetractQueuedWork()
    {abstract} + virtual int32 GetNumThreads()
}

class FQueuedThreadPoolBase
{
    # FThreadPoolPriorityQueue QueuedWork
    # TArray<FQueuedThread*> QueuedThreads
    # TArray<FQueuedThread*> AllThreads
    # FCriticalSection* SynchQueue
    # bool TimeToDie

    + virtual bool Create()
    + virtual void Destroy()
    + virtual void AddQueuedWork()
    + virtual void RetractQueuedWork()
    + virtual int32 GetNumThreads() 
}

FQueuedThreadPool <|-- FQueuedThreadPoolBase
FQueuedThread *-- FQueuedThreadPoolBase
IQueuedWork *-- FQueuedThreadPoolBase

note left of FQueuedThreadPoolBase::QueuedThreads
当前线程池中空闲的线程
end note

note left of FQueuedThreadPoolBase::QueuedWork
线程池中还没有被运行的任务，会根据Prriority进行排序
end note

@enduml
```

##### FQueuedThread
FQueuedThread继承于FRunable，同时它也包含一个FRunnableThread的引用。一方面它代表了被线程池管理的那些线程，另外一方面它也包含了执行任务的逻辑。它的任务就是在运行的时候不断的从线程池中拿出队列中还没有执行的任务，然后执行它。如果线程池中的任务已经被执行完了，它就会把自己返回到线程池的空闲队列中。主要的逻辑在FQueuedThread::Run中：
```cpp
uint32 FQueuedThread::Run()
{
	while (!TimeToDie.Load(EMemoryOrder::Relaxed))
	{
		// This will force sending the stats packet from the previous frame.
		SET_DWORD_STAT(STAT_ThreadPoolDummyCounter, 0);
		// We need to wait for shorter amount of time
		bool bContinueWaiting = true;

		// Unless we're collecting stats there doesn't appear to be any reason to wake
		// up again until there's work to do (or it's time to die)

#if STATS
		if (FThreadStats::IsCollectingData())
		{
			while (bContinueWaiting)
			{
				DECLARE_CYCLE_STAT_WITH_FLAGS(TEXT("FQueuedThread::Run.WaitForWork"),
				STAT_FQueuedThread_Run_WaitForWork, STATGROUP_ThreadPoolAsyncTasks,
					EStatFlags::Verbose);

				SCOPE_CYCLE_COUNTER(STAT_FQueuedThread_Run_WaitForWork);

				// Wait for some work to do

				bContinueWaiting = !DoWorkEvent->Wait(GDoPooledThreadWaitTimeouts ? 10 : MAX_uint32);
			}
		}
#endif

		if (bContinueWaiting)
		{
			DoWorkEvent->Wait();
		}

		IQueuedWork* LocalQueuedWork = QueuedWork;
		QueuedWork = nullptr;
		FPlatformMisc::MemoryBarrier();
		check(LocalQueuedWork || TimeToDie.Load(EMemoryOrder::Relaxed)); // well you woke me up, where is the job or termination request?
		while (LocalQueuedWork)
		{
			// Tell the object to do the work
			LocalQueuedWork->DoThreadedWork();
			// Let the object cleanup before we remove our ref to it
			LocalQueuedWork = OwningThreadPool->ReturnToPoolOrGetNextJob(this);
		}
	}
	return 0;
}
```

##### FQueuedThreadPoolBase
FQueuedThreadPoolBase是UE中线程池的实现，主要就是管理线程池中的任务，先线程池中添加新任务，撤回任务这些等等。
- QueuedThreadPoolBase::AddQueuedWork 向线程池中添加任务，如果线程池中有空闲的线程，则开始执行该任务
- QueuedThreadPoolBase::RetractQueuedWork 如果该任务还没有开始执行，则撤销该任务，之后不会执行该任务
- QueuedThreadPoolBase::ReturnToPoolOrGetNextJob 这个接口会被FQueuedThread调用，当FQueuedThread执行完一个任务后，会调用该接口。如果已经没有可执行的任务了，会将FQueuedThread归还到空闲的线程池列表中。如果还有未执行的任务，则会从未执行的任务列表拿出来，然后执行。FQueuedThreadPoolBase和FQueuedThread是一个生产者和消费着模型。
```cpp
void AddQueuedWork(IQueuedWork* InQueuedWork, EQueuedWorkPriority InQueuedWorkPriority) override
{
	check(InQueuedWork != nullptr);

	if (TimeToDie)
	{
		InQueuedWork->Abandon();
		return;
	}

	// Check to see if a thread is available. Make sure no other threads
	// can manipulate the thread pool while we do this.
	//
	// We pick a thread from the back of the array since this will be the
	// most recently used thread and therefore the most likely to have
	// a 'hot' cache for the stack etc (similar to Windows IOCP scheduling
	// strategy). Picking from the back also happens to be cheaper since
	// no memory movement is necessary.

	check(SynchQueue);

	FQueuedThread* Thread = nullptr;

	{
		FScopeLock sl(SynchQueue);
		const int32 AvailableThreadCount = QueuedThreads.Num();
		if (AvailableThreadCount == 0)
		{
			// No thread available, queue the work to be done
			// as soon as one does become available
			QueuedWork.Enqueue(InQueuedWork, InQueuedWorkPriority);
			return;
		}

		const int32 ThreadIndex = AvailableThreadCount - 1;

		Thread = QueuedThreads[ThreadIndex];
		// Remove it from the list so no one else grabs it
		QueuedThreads.RemoveAt(ThreadIndex, 1, /* do not allow shrinking */ false);
	}

	// Tell our chosen thread to do the work
	Thread->DoWork(InQueuedWork);
}

virtual bool RetractQueuedWork(IQueuedWork* InQueuedWork) override
{
	if (TimeToDie)
	{
		return false; // no special consideration for this, refuse the retraction and let shutdown proceed
	}
	check(InQueuedWork != nullptr);
	check(SynchQueue);
	FScopeLock sl(SynchQueue);
	return QueuedWork.Retract(InQueuedWork);
}

IQueuedWork* ReturnToPoolOrGetNextJob(FQueuedThread* InQueuedThread)
{
	check(InQueuedThread != nullptr);
	IQueuedWork* Work = nullptr;
	// Check to see if there is any work to be done
	FScopeLock sl(SynchQueue);
	if (TimeToDie)
	{
		check(!QueuedWork.Num());  // we better not have anything if we are dying
	}
	
	Work = QueuedWork.Dequeue();

	if (!Work)
	{
		// There was no work to be done, so add the thread to the pool
		QueuedThreads.Add(InQueuedThread);
	}
	return Work;
}
```

#### TaskGraph的实现
```plantuml
@startuml
abstract class FBaseGraphTask
{
	- virtual void ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread, bool bDeleteOnCompletion)
	- virtual void DeleteTask()

}

class FConstructor<T>
{
	+ FGraphEventRef ConstructAndDispatchWhenReady(T&&... Args)
	+ TGraphTask* ConstructAndHold(T&&... Args)
}

class FGraphEvent
{
	{static} + FGraphEventRef CreateGraphEvent()
	+ bool AddSubsequent(class FBaseGraphTask* Subsequent)
	+ void DispatchSubsequents(ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
	+ void DispatchSubsequents(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread, bool bInternal = false)

	- TClosableLockFreePointerListUnorderedSingleConsumer<FBaseGraphTask, 0>	SubsequentList
	- FGraphEventArray EventsToWaitFor
	- FThreadSafeCounter ReferenceCount
}

class TGraphTask<TTask>
{
	{static} + FConstructor CreateTask(const FGraphEventArray* Prerequisites = NULL, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
	
	- void ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread, bool bDeleteOnCompletion)
	- void DeleteTask()
	- void SetupPrereqs(const FGraphEventArray* Prerequisites, ENamedThreads::Type CurrentThreadIfKnown, bool bUnlock)
	- FGraphEventRef Setup(const FGraphEventArray* Prerequisites = NULL, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
	- TGraphTask* Hold(const FGraphEventArray* Prerequisites = NULL, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)

	- TAlignedBytes<sizeof(TTask),alignof(TTask)> TaskStorage
	- bool TaskConstructed
	- FGraphEventRef Subsequents
}

FBaseGraphTask <|-- TGraphTask
FConstructor <.. TGraphTask
FGraphEvent *-- TGraphTask

abstract class FTaskThreadBase
{
	{abstract} + virtual void ProcessTaskUntilQuit(int32 QueueIndex)
	+ virtual uint64 ProcessTasksUntilIdle(int32 QueueIndex)
	+ virtual void EnqueueFromThisThread(int32 QueueIndex, FBaseGraphTask* Task)
	+ virtual void RequestQuit(int32 QueueIndex)
	+ virtual bool EnqueueFromOtherThread(int32 QueueIndex, FBaseGraphTask* Task)
	+ virtual void WakeUp(int32 QueueIndex = 0)
	+ virtual bool IsProcessingTasks(int32 QueueIndex)
	+ virtual uint32 Run()

	# ENamedThreads::Type ThreadId
	# uint32 PerThreadIDTLSSlot
	# FThreadSafeCounter IsStalled
	# TArray<FBaseGraphTask*> NewTasks
	# FWorkerThread* OwnerWorker
}

FBaseGraphTask *-- FTaskThreadBase

FRunnable <|-- FTaskThreadBase

struct FWorkerThread
{
	FTaskThreadBase* TaskGraphWorker
	FRunnableThread* RunnableThread
	bool bAttached
}

FWorkerThread *--* FTaskThreadBase
FRunnableThread *-- FWorkerThread

class FNamedTaskThread
{

}

FTaskThreadBase <|-- FNamedTaskThread

class FTaskThreadAnyThread
{

}
FTaskThreadBase <|-- FTaskThreadAnyThread


abstract class FTaskGraphInterface
{
	{abstract} + virtual void QueueTask(class FBaseGraphTask* Task, bool bWakeUpWorker, ENamedThreads::Type ThreadToExecuteOn, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)

	{abstract} + virtual uint64 ProcessThreadUntilIdle(ENamedThreads::Type CurrentThread)
	{abstract} + virtual void ProcessThreadUntilRequestReturn(ENamedThreads::Type CurrentThread)
	{abstract} + virtual void WaitUntilTasksComplete(const FGraphEventArray& Tasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
}

class FTaskGraphImplementation
{
	- FWorkerThread WorkerThreads[MAX_THREADS]
	
	- int32 NumThreads
	- int32 NumNamedThreads
	- int32 NumTaskThreadSets
	- int32 NumTaskThreadsPerSet
	- bool bCreatedHiPriorityThreads
	- bool bCreatedBackgroundPriorityThreads

	- ENamedThreads::Type LastExternalThread
	- FThreadSafeCounter	ReentrancyCheck

	- uint32 PerThreadIDTLSSlot

	{field} - TArray<TFunction<void()> > ShutdownCallbacks

	- FStallingTaskQueue<FBaseGraphTask, PLATFORM_CACHE_LINE_SIZE, 2>	IncomingAnyThreadTasks[MAX_THREAD_PRIORITIES]
}

FTaskGraphInterface <|-- FTaskGraphImplementation
FBaseGraphTask *-- FTaskGraphImplementation
FWorkerThread *-- FTaskGraphImplementation

class FTaskGraphCompatibilityImplementation
{
	
}

FTaskGraphInterface <|-- FTaskGraphCompatibilityImplementation


@enduml
```
#### FTaskGraphInterface
在TaskGraph中，FTaskGraphInterface是TaskGraph的接口类，用管理TaskGraph相关的工作，具体的实现在FTaskGraphImplementation和FTaskGraphCompatibilityImplementation来完成，是一个单例类。

在TaskGraph中，有两中类型的线程，一个是NamedThread，一个是AnyThread。分别对应FNamedTaskThread和FTaskThreadAnyThread。AnyTread会在TaskGraph初始化的时候被创建出来。NamedThread会在该类型的线程创建的时候Attach到相应的Workder中。目前支持的NamedThread有：
- RHIThread RHI线程
- GameThread 游戏线程
- RenderThread 渲染线程

```cpp
enum Type : int32
{
	UnusedAnchor = -1,
	/** The always-present, named threads are listed next **/
	RHIThread,
	GameThread,
	// The render thread is sometimes the game thread and is sometimes the actual rendering thread
	ActualRenderingThread = GameThread + 1,
	// CAUTION ThreadedRenderingThread must be the last named thread, insert new named threads before it

	/** not actually a thread index. Means "Unknown Thread" or "Any Unnamed Thread" **/
	AnyThread = 0xff, 
	....
}
```

对于TaskGraph中的任务和线程也分别有不同的优先级。有普通和高优先级的任务，也有各种不同优先级的线程。
- NormalTaskPriority  普通优先级任务
- HighTaskPriority  高优先级任务
- NormalThreadPriority  普通优先级线程
- HighThreadPriority  高优先级线程
- BackgroundThreadPriority  低优先级线程

然后还可以通过这些不同的flag进行组合，组合成不同类型。同时由于这些Flag占用了不同位，所以也可以通过位运算，将其中的不同的属性单独拿出来。UE中也定义了相应的函数。最后这些所有属性的组合决定了我们当前的任务在哪一个线程中来运行。例如AnyHiPriThreadNormalTask就是运行在一个高优先级AnyThread的普通优先级的任务。

```cpp
enum Type : int32
{
	 ...

	MainQueue =			0x000,
	LocalQueue =		0x100,

	NumQueues =			2,
	ThreadIndexMask =	0xff,
	QueueIndexMask =	0x100,
	QueueIndexShift =	8,

	/** High bits are used for a queue index task priority and thread priority**/

	NormalTaskPriority =	0x000,
	HighTaskPriority =		0x200,

	NumTaskPriorities =		2,
	TaskPriorityMask =		0x200,
	TaskPriorityShift =		9,

	NormalThreadPriority = 0x000,
	HighThreadPriority = 0x400,
	BackgroundThreadPriority = 0x800,

	NumThreadPriorities = 3,
	ThreadPriorityMask = 0xC00,
	ThreadPriorityShift = 10,

	/** Combinations **/
	GameThread_Local = GameThread | LocalQueue,
	ActualRenderingThread_Local = ActualRenderingThread | LocalQueue,

	AnyHiPriThreadNormalTask = AnyThread | HighThreadPriority | NormalTaskPriority,
	AnyHiPriThreadHiPriTask = AnyThread | HighThreadPriority | HighTaskPriority,

	AnyNormalThreadNormalTask = AnyThread | NormalThreadPriority | NormalTaskPriority,
	AnyNormalThreadHiPriTask = AnyThread | NormalThreadPriority | HighTaskPriority,

	AnyBackgroundThreadNormalTask = AnyThread | BackgroundThreadPriority | NormalTaskPriority,
	AnyBackgroundHiPriTask = AnyThread | BackgroundThreadPriority | HighTaskPriority,
};

//辅助函数，用于设置和获取不同的Flag
FORCEINLINE Type GetThreadIndex(Type ThreadAndIndex)
{
	return ((ThreadAndIndex & ThreadIndexMask) == AnyThread) ? AnyThread : Type(ThreadAndIndex & ThreadIndexMask);
}

FORCEINLINE int32 GetQueueIndex(Type ThreadAndIndex)
{
	return (ThreadAndIndex & QueueIndexMask) >> QueueIndexShift;
}

FORCEINLINE int32 GetTaskPriority(Type ThreadAndIndex)
{
	return (ThreadAndIndex & TaskPriorityMask) >> TaskPriorityShift;
}

FORCEINLINE int32 GetThreadPriorityIndex(Type ThreadAndIndex)
{
	int32 Result = (ThreadAndIndex & ThreadPriorityMask) >> ThreadPriorityShift;
	check(Result >= 0 && Result < NumThreadPriorities);
	return Result;
}

FORCEINLINE Type SetPriorities(Type ThreadAndIndex, Type ThreadPriority, Type TaskPriority)
{
	check(
		!(ThreadAndIndex & ~ThreadIndexMask) &&  // not a thread index
		!(ThreadPriority & ~ThreadPriorityMask) && // not a thread priority
		(ThreadPriority & ThreadPriorityMask) != ThreadPriorityMask && // not a valid thread priority
		!(TaskPriority & ~TaskPriorityMask) // not a task priority
		);
	return Type(ThreadAndIndex | ThreadPriority | TaskPriority);
}

FORCEINLINE Type SetPriorities(Type ThreadAndIndex, int32 PriorityIndex, bool bHiPri)
{
	check(
		!(ThreadAndIndex & ~ThreadIndexMask) && // not a thread index
		PriorityIndex >= 0 && PriorityIndex < NumThreadPriorities // not a valid thread priority
		);
	return Type(ThreadAndIndex | (PriorityIndex << ThreadPriorityShift) | (bHiPri ? HighTaskPriority : NormalTaskPriority));
}

FORCEINLINE Type SetThreadPriority(Type ThreadAndIndex, Type ThreadPriority)
{
	check(
		!(ThreadAndIndex & ~ThreadIndexMask) &&  // not a thread index
		!(ThreadPriority & ~ThreadPriorityMask) && // not a thread priority
		(ThreadPriority & ThreadPriorityMask) != ThreadPriorityMask // not a valid thread priority
		);
	return Type(ThreadAndIndex | ThreadPriority);
}

FORCEINLINE Type SetTaskPriority(Type ThreadAndIndex, Type TaskPriority)
{
	check(
		!(ThreadAndIndex & ~ThreadIndexMask) &&  // not a thread index
		!(TaskPriority & ~TaskPriorityMask) // not a task priority
		);
	return Type(ThreadAndIndex | TaskPriority);
}
```

对于AnyThread而言，会在初始化的时候，根据当前系统和CPU的核心数量，还有相应的config来决定创建多少AnyThread。对于AnyThread而言，又有线程集（Thread Set）和线程优先级（Thread Priority）的概念。
- 线程优先级
TaskGraph中有三个优先级的线程，分别是Normal, High, Background。High的优先级最高，Background的优先级最低。

- 线程集
一组由多个同一优先级的线程组成的集合叫线程集，UE中至少有一个线程集，最多有三个线程集。线程集的数量和当前引擎的设置有关。下面的代码是线程集数量的计算方式：
```cpp
NumTaskThreadSets = 1 + bCreatedHiPriorityThreads + bCreatedBackgroundPriorityThreads
```

NamedTread是通过外部设置到TaskGraph中来的，并未TaskGraph内部创建的。可以通过函数**FTaskGraphInterface::AttachToThread**来设置NamedThread。
```cpp
virtual void AttachToThread(ENamedThreads::Type CurrentThread) final override
{
	CurrentThread = ENamedThreads::GetThreadIndex(CurrentThread);
	check(NumTaskThreadsPerSet);
	check(CurrentThread >= 0 && CurrentThread < NumNamedThreads);
	check(!WorkerThreads[CurrentThread].bAttached);
	Thread(CurrentThread).InitializeForCurrentThread();
}
```

GameThread和RenderThread这些NamedThread都会在线程创建成功后，就把对应的线程Attach到TaskGraph中来。
```cpp
void RenderingThreadMain( FEvent* TaskGraphBoundSyncEvent )
{
	LLM_SCOPE(ELLMTag::RenderingThreadMemory);

	ENamedThreads::Type RenderThread = ENamedThreads::Type(ENamedThreads::ActualRenderingThread);

	ENamedThreads::SetRenderThread(RenderThread);
	ENamedThreads::SetRenderThread_Local(ENamedThreads::Type(ENamedThreads::ActualRenderingThread_Local));

	FTaskGraphInterface::Get().AttachToThread(RenderThread);
	...
}

// FRHIThread::Run
virtual uint32 Run() override
{
	LLM_SCOPE(ELLMTag::RHIMisc);

#if CSV_PROFILER
	FCsvProfiler::Get()->SetRHIThreadId(FPlatformTLS::GetCurrentThreadId());
#endif

	FMemory::SetupTLSCachesOnCurrentThread();
	{
		FTaskTagScope Scope(ETaskTag::ERhiThread);
		FPlatformProcess::SetupRHIThread();
		FTaskGraphInterface::Get().AttachToThread(ENamedThreads::RHIThread);
		FTaskGraphInterface::Get().ProcessThreadUntilRequestReturn(ENamedThreads::RHIThread);
	}
	FMemory::ClearAndDisableTLSCachesOnCurrentThread();
	return 0;
}

int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine)
{
	....

	if (bCreateTaskGraphAndThreadPools)
	{
		// initialize task graph sub-system with potential multiple threads
		SCOPED_BOOT_TIMING("FTaskGraphInterface::Startup");
		FTaskGraphInterface::Startup(FPlatformMisc::NumberOfWorkerThreadsToSpawn());
		FTaskGraphInterface::Get().AttachToThread(ENamedThreads::GameThread);
	}

	...
}

```

##### FTaskGraphImplementation
FTaskGraphImplementation在初始化时候，会根据当前系统的核心数量和配置初始化一定数量的Worker。对于AnyThread，还会创建其对应的FRunnableThread，计算对应线程的Affinity。

当前系统的核心数会决定线程集的数量，而每个线程集中的线程数量取决于**ENamedThreads::bHasHighPriorityThreads和ENamedThreads::bHasBackgroundThreads**。最终Worker的数量计算公式为：**NumTaskThreads * NumTaskThreadSets + NumNamedThreads**，然后根据实际的情况进行clamp。
```cpp
FTaskGraphImplementation(int32)
{
	TaskTrace::Init();
	bCreatedHiPriorityThreads = !!ENamedThreads::bHasHighPriorityThreads;
	bCreatedBackgroundPriorityThreads = !!ENamedThreads::bHasBackgroundThreads;
	int32 MaxTaskThreads = MAX_THREADS;
	int32 NumTaskThreads = FPlatformMisc::NumberOfWorkerThreadsToSpawn();
	// if we don't want any performance-based threads, then force the task graph to not create any worker threads, and run in game thread
	if (!FTaskGraphInterface::IsMultithread())
	{
		// this is the logic that used to be spread over a couple of places, that will make the rest of this function disable a workerthread
		// @todo: it could probably be made simpler/clearer
		// this - 1 tells the below code there is no rendering thread
		MaxTaskThreads = 1;
		NumTaskThreads = 1;
		LastExternalThread = (ENamedThreads::Type)(ENamedThreads::ActualRenderingThread - 1);
		bCreatedHiPriorityThreads = false;
		bCreatedBackgroundPriorityThreads = false;
		ENamedThreads::bHasBackgroundThreads = 0;
		ENamedThreads::bHasHighPriorityThreads = 0;
	}
	else
	{
		LastExternalThread = ENamedThreads::ActualRenderingThread;
		if (FForkProcessHelper::IsForkedMultithreadInstance())
		{
			NumTaskThreads = CVar_ForkedProcess_MaxWorkerThreads;
		}
	}
	
	NumNamedThreads = LastExternalThread + 1;
	NumTaskThreadSets = 1 + bCreatedHiPriorityThreads + bCreatedBackgroundPriorityThreads;
	// if we don't have enough threads to allow all of the sets asked for, then we can't create what was asked for.
	check(NumTaskThreadSets == 1 || FMath::Min<int32>(NumTaskThreads * NumTaskThreadSets + NumNamedThreads, MAX_THREADS) == NumTaskThreads* NumTaskThreadSets + NumNamedThreads);
	NumThreads = FMath::Max<int32>(FMath::Min<int32>(NumTaskThreads * NumTaskThreadSets + NumNamedThreads, MAX_THREADS), NumNamedThreads +1);
	// Cap number of extra threads to the platform worker thread count
	// if we don't have enough threads to allow all of the sets asked for, then we can't create what was asked for.
	check(NumTaskThreadSets == 1 || FMath::Min(NumThreads, NumNamedThreads + NumTaskThreads * NumTaskThreadSets) == NumThreads);
	NumThreads = FMath::Min(NumThreads, NumNamedThreads + NumTaskThreads * NumTaskThreadSets);
	NumTaskThreadsPerSet = (NumThreads - NumNamedThreads) / NumTaskThreadSets;
	check((NumThreads - NumNamedThreads) % NumTaskThreadSets == 0); // should be equal numbers of threads per priority set
	UE_LOG(LogTaskGraph, Log, TEXT("Started task graph with %d named threads and %d total threads with %d sets of task threads."),NumNamedThreads, NumThreads, NumTaskThreadSets);
	check(NumThreads - NumNamedThreads >= 1);  // need at least one pure worker thread
	check(NumThreads <= MAX_THREADS);
	check(!ReentrancyCheck.GetValue()); // reentrant?
	ReentrancyCheck.Increment(); // just checking for reentrancy
	PerThreadIDTLSSlot = FPlatformTLS::AllocTlsSlot();
	for (int32 ThreadIndex = 0; ThreadIndex < NumThreads; ThreadIndex++)
	{
		check(!WorkerThreads[ThreadIndex].bAttached); // reentrant?
		bool bAnyTaskThread = ThreadIndex >= NumNamedThreads;
		if (bAnyTaskThread)
		{
			WorkerThreads[ThreadIndex].TaskGraphWorker = new FTaskThreadAnyThread(ThreadIndexToPriorityIndex(ThreadIndex));
		}
		else
		{
			WorkerThreads[ThreadIndex].TaskGraphWorker = new FNamedTaskThread;
		}
		WorkerThreads[ThreadIndex].TaskGraphWorker->Setup(ENamedThreads::Type(ThreadIndex), PerThreadIDTLSSlot, &WorkerThread[ThreadIndex]);
	}
	TaskGraphImplementationSingleton = this; // now reentrancy is ok
	const TCHAR* PrevGroupName = nullptr;
	for (int32 ThreadIndex = LastExternalThread + 1; ThreadIndex < NumThreads; ThreadIndex++)
	{
		FString Name;
		const TCHAR* GroupName = TEXT("TaskGraphNormal");
		int32 Priority = ThreadIndexToPriorityIndex(ThreadIndex);
        // These are below normal threads so that they sleep when the named threads are active
		EThreadPriority ThreadPri;
		uint64 Affinity = FPlatformAffinity::GetTaskGraphThreadMask();
		if (Priority == 1)
		{
			Name = FString::Printf(TEXT("TaskGraphThreadHP %d"), ThreadIndex - (LastExternalThread + 1));
			GroupName = TEXT("TaskGraphHigh");
			ThreadPri = TPri_SlightlyBelowNormal; // we want even hi priority tasks below the normal threads
			// If the platform defines FPlatformAffinity::GetTaskGraphHighPriorityTaskMask then use it
			if (FPlatformAffinity::GetTaskGraphHighPriorityTaskMask() != 0xFFFFFFFFFFFFFFFF)
			{
				Affinity = FPlatformAffinity::GetTaskGraphHighPriorityTaskMask();
			}
		}
		else if (Priority == 2)
		{
			Name = FString::Printf(TEXT("TaskGraphThreadBP %d"), ThreadIndex - (LastExternalThread + 1));
			GroupName = TEXT("TaskGraphLow");
			ThreadPri = TPri_Lowest;
			// If the platform defines FPlatformAffinity::GetTaskGraphBackgroundTaskMask then use it
			if ( FPlatformAffinity::GetTaskGraphBackgroundTaskMask() != 0xFFFFFFFFFFFFFFFF )
			{
				Affinity = FPlatformAffinity::GetTaskGraphBackgroundTaskMask();
			}
		}
		else
		{
			Name = FString::Printf(TEXT("TaskGraphThreadNP %d"), ThreadIndex - (LastExternalThread + 1));
			ThreadPri = TPri_BelowNormal; // we want normal tasks below normal threads like the game thread
		}
		int32 StackSize;
		StackSize = 1024 * 1024;
		if (FForkProcessHelper::IsForkedMultithreadInstance() && GAllowTaskGraphForkMultithreading)
		{
			WorkerThreads[ThreadIndex].RunnableThread = FForkProcessHelper::CreateForkableThread(&Thread(ThreadIndex), *Name, StackSize,ThreadPri, Affinity);
		}
		else
		{
			WorkerThreads[ThreadIndex].RunnableThread = FRunnableThread::Create(&Thread(ThreadIndex), *Name, StackSize, ThreadPri,Affinity); 
		}
		
		WorkerThreads[ThreadIndex].bAttached = true;
	}
	UE::Trace::ThreadGroupEnd();
}
```

从代码中可以看出，最终创建的线程都会放在**FTaskGraphImplementation::WorkderThreads**这个数组中，而且会按照顺序排列，所以我们就可以直接通过Index来获取当前Thread的属性，是执行什么任务的线程。
```mermaid
graph LR
A[按顺序排列的NameThread] --> B[普通优先级线程] --> C[高优先级线程] --> D[低优先级线程]
```

#### FTaskThreadBase
FTaskThreadBase继承于FRunnable，是一个抽象类。FTaskThreadAnyThread和FNamedTaskThread都继承于它，是TaskGraph中的线程真正运行的对象。在TaskGraph运行的时候，FTaskThreadAnyThread和FNamedTaskThread会通过自己的运行逻辑，拿到真正需要的运行的FBaseGraphTask，从而运行真正的任务。

FRunnable对象最后会被对应的线程调用FRunnable::Run中，所以FTaskThreadBase的主要运行逻辑也在FTaskThreadBase::Run中。FTaskThreadBase::Run中调用了FTaskThreadBase::ProcessTasksUntilQuit，而该函数是一个纯虚函数，所以对应的逻辑在子类中实现。

##### FTaskThreadAnyThread
FTaskThreadAnyThread 代表了两种线程中的AnyThread，它的数量会在FTaskGraphImplementation初始化的时候根据当前系统的核心数量以及一些配置来决定。FTaskThreadAnyThread的主要函数为FTaskThreadAnyThread::ProcessTasks，这个函数会从FTaskGraphInterface中去寻找自己ThreadId所对应的任务。
```cpp
	uint64 ProcessTasks()
	{
		LLM_SCOPE_BYNAME(TEXT("Tasks/AnyThread/ProcessTasks"));

		TStatId StallStatId;
		bool bCountAsStall = true;
		uint64 ProcessedTasks = 0;
#if STATS
		TStatId StatName;
		FCycleCounter ProcessingTasks;
		StatName = GET_STATID(STAT_TaskGraph_OtherTasks);
		StallStatId = GET_STATID(STAT_TaskGraph_OtherStalls);
		bool bTasksOpen = false;
		if (FThreadStats::IsCollectingData(StatName))
		{
			bTasksOpen = true;
			ProcessingTasks.Start(StatName);
		}
#endif
		verify(++Queue.RecursionGuard == 1);
		bool bDidStall = false;
		while (1)
		{
			FBaseGraphTask* Task = FindWork();
			if (!Task)
			{
#if STATS
				if (bTasksOpen)
				{
					ProcessingTasks.Stop();
					bTasksOpen = false;
				}
#endif

				TestRandomizedThreads();
				const bool bIsMultithread = FTaskGraphInterface::IsMultithread();
				if (bIsMultithread)
				{
					FScopeCycleCounter Scope(StallStatId, EStatFlags::Verbose);
					Queue.StallRestartEvent->Wait(MAX_uint32, bCountAsStall);
					bDidStall = true;
				}
				if (Queue.QuitForShutdown || !bIsMultithread)
				{
					break;
				}
				TestRandomizedThreads();

#if STATS
				if (FThreadStats::IsCollectingData(StatName))
				{
					bTasksOpen = true;
					ProcessingTasks.Start(StatName);
				}
#endif
				continue;
			}
			TestRandomizedThreads();
#if YIELD_BETWEEN_TASKS
			// the Win scheduler is ill behaved and will sometimes let BG tasks run even when other tasks are ready....kick the scheduler between tasks
			if (!bDidStall && PriorityIndex == (ENamedThreads::BackgroundThreadPriority >> ENamedThreads::ThreadPriorityShift))
			{
				FPlatformProcess::Sleep(0);
			}
#endif
			bDidStall = false;
			Task->Execute(NewTasks, ENamedThreads::Type(ThreadId), true);
			ProcessedTasks++;
			TestRandomizedThreads();
			if (Queue.bStallForTuning)
			{
#if STATS
				if (bTasksOpen)
				{
					ProcessingTasks.Stop();
					bTasksOpen = false;
				}
#endif
				{
					FScopeLock Lock(&Queue.StallForTuning);
				}
#if STATS
				if (FThreadStats::IsCollectingData(StatName))
				{
					bTasksOpen = true;
					ProcessingTasks.Start(StatName);
				}
#endif
			}
		}
		verify(!--Queue.RecursionGuard);
		return ProcessedTasks;
	}

```

可以从上面看出来，最终调用到了FBaseGraphTask::Excute。

##### FNamedTaskThread
FNamedTaskThread 代表了两种线程中的NamedThread，它的数量和当前的设置和引擎的版本都有关。例如在UE4中，有Stats和Audio相关的NamedThread，在升级到UE5的时候被移除了。而且UE4中，也只有STATS宏打开的时候会有StatsThread。

FNamedTaskThread相对于FTaskThreadAnyThread还多了一个Queue的区分，分为MainQueue和LocalQueue。
- MainQueue
- LocalQueu


#### FBaseGraphTask
FBaseGraphTask是TaskGraph中所有Task的基类，线程在执行任务时会调用FBaseGraphTask::ExecuteTask。FBaseGraphTask本身是一个抽象类，UE实现了一个模板类TGraphTask用于设置前置任务和后续任务。TGraphTask中还有一个辅助类FConstructor用于任务的创建。

##### 模板参数TTask
TGraphTask中完善了设置前置任务和后置任务，以及执行任务的代码。由于是一个模板类，所以在实现中调用了一些模板的函数，所以实例化TGraphTask的模板参数需要满足一些条件。不然在生成模板代码的时候，会出现编译错误。UE在注释中给出了这个例子：
```cpp
class FGenericTask
{
	TSomeType	SomeArgument;
public:
	FGenericTask(TSomeType InSomeArgument) // CAUTION!: Must not use references in the constructor args; use pointers instead if you need by reference
		: SomeArgument(InSomeArgument)
	{
		// Usually the constructor doesn't do anything except save the arguments for use in DoWork or GetDesiredThread.
	}
	~FGenericTask()
	{
		// you will be destroyed immediately after you execute. Might as well do cleanup in DoWork, but you could also use a destructor.
	}
	FORCEINLINE TStatId GetStatId() const
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(FGenericTask, STATGROUP_TaskGraphTasks);
	}

	[static] ENamedThreads::Type GetDesiredThread()
	{
		return ENamedThreads::[named thread or AnyThread];
	}
	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		// The arguments are useful for setting up other tasks. 
		// Do work here, probably using SomeArgument.
		MyCompletionGraphEvent->DontCompleteUntil(TGraphTask<FSomeChildTask>::CreateTask(NULL,CurrentThread).ConstructAndDispatchWhenReady());
	}
};
```

##### FConstructor
FConstructor中有两个公共接口FGraphEventRef ConstructAndDispatchWhenReady(T&&... Args)和TGraphTask* ConstructAndHold(T&&... Args)：
- TGraphTask* ConstructAndHold(T&&... Args) 创建一个新的Task, 暂时不加入到任务队列，需要手动调用加入到任务队列中去执行，例如调用TGraphTask::Unlock
- FGraphEventRef ConstructAndDispatchWhenReady(T&&... Args) 创建一个新的Task，并加入到任务队列，在合适的时机开始执行

#### FGraphEvent
FGraphEvent是一个后续任务的集合，依赖的任务完成之后，才会将后续的任务放入TaskGraph的任务队列中进行执行。在FGraphEvent中通过一个Lock Free的表**SubsequentList**来保存后续的任务，同时也可以保证访问后续任务时的效率。它本身有一个引用计数来控制它的生命周期。FGraphEvent主要有以下几个接口：
- FGraphEvent::AddSubsequent 添加后续新的任务
- FGraphEvent::DispatchSubsequents 执行后续的任务，会在FBaseGraphTask被执行调用Excute的时候被调用
- FGraphEvent::Wait 等待直到当前任务被执行完
- FGraphEvent::AddRef && FGraphEvent::Release 增加减少FGraphEvent的引用计数

**FGraphEvent::Wait**会调用FTaskGraphImplementation::WaitUntilTasksComplete，然后会在任务的最后又添加了一个新的任务**FTriggerEventGraphTask**，在内部有一个FEvent会在这个任务被完成的时候被触发，调用FEvent::Trigger。这样从而达到在完成了所有任务之后等待的效果。
```cpp
virtual void WaitUntilTasksComplete(const FGraphEventArray& Tasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread) final override
{
	.....

	if (!FTaskGraphInterface::IsMultithread())
	{
		bool bAnyPending = false;
		for (int32 Index = 0; Index < Tasks.Num(); Index++)
		{
			FGraphEvent* Task = Tasks[Index].GetReference();
			if (Task && !Task->IsComplete())
			{
				bAnyPending = true;
				break;
			}
		}
		if (!bAnyPending)
		{
			return;
		}
		UE_LOG(LogTaskGraph, Fatal, TEXT("Recursive waits are not allowed in single threaded mode."));
	}

	// We will just stall this thread on an event while we wait
	FScopedEvent Event;
	TriggerEventWhenTasksComplete(Event.Get(), Tasks, CurrentThreadIfKnown);	
}

virtual void TriggerEventWhenTasksComplete(FEvent* InEvent, const FGraphEventArray& Tasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread, ENamedThreads::Type TriggerThread = ENamedThreads::AnyHiPriThreadHiPriTask) final override
{
	check(InEvent);
	bool bAnyPending = true;
	if (Tasks.Num() < 8) // don't bother to check for completion if there are lots of prereqs...too expensive to check
	{
		bAnyPending = false;
		for (int32 Index = 0; Index < Tasks.Num(); Index++)
		{
			FGraphEvent* Task = Tasks[Index].GetReference();
			if (Task && !Task->IsComplete())
			{
				bAnyPending = true;
				break;
			}
		}
	}
	if (!bAnyPending)
	{
		TestRandomizedThreads();
		InEvent->Trigger();
		return;
	}
	TGraphTask<FTriggerEventGraphTask>::CreateTask(&Tasks, CurrentThreadIfKnown).ConstructAndDispatchWhenReady(InEvent, TriggerThread);
}
```

#### TaskGraph运行流程
```mermaid
graph TB
DispathReady[ConstructAndDispatchWhenReady] --bUnlock=true--> SetupPrereqs[TGraphTask::SetupPrereqs] --> PreComplete[FBaseGraphTask::PrerequisitesComplete] -->Unlock{bUnlock}

TaskQueueTask[FBaseGraphTask::QueueTask] --> InterfaceQueueTask{FTaskGraphInterface::QueueTask}

Unlock -->|bUnlock=true| TaskQueueTask

Hold[ConstructAndHold] --bUnnlock=false--> SetupPrereqs

Unlock -->|bUnlock=false| WaitForUnlock[WaitForUnlock]
WaitForUnlock --> ConditionalQueueTask[FBaseGraphTask::ConditionalQueueTask] --> TaskQueueTask

InterfaceQueueTask -->|AnyThread| StartTaskThread[FTaskGraphImplementation::StartTaskThread]
StartTaskThread --> WakeUp[FTaskThreadAnyThread::WakeUp] --> ProcessTasks[FTaskThreadAnyThread::ProcessTasks] --> Excute[FBaseGraphTask::Execute] --> ExcuteTask[TGraphTask::ExcuteTask]

InterfaceQueueTask -->|NamedThread| NamedThreadChoice{ThreadToExecuteOn==CurrentThreadIfKnown}

NamedThreadChoice -->|true| EnqueueFromThisThread[FTaskThreadBase::EnqueueFromThisThread]
NamedThreadChoice -->|false| EnqueueFromOtherThread[FTaskThreadBase::EnqueueFromOtherThread]

```