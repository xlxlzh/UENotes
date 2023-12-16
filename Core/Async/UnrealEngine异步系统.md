## Unreal Engine中的异步系统

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Unreal Engine中的异步系统](#unreal-engine中的异步系统)
  - [类图Overall](#类图overall)
  - [FRunnableThread 和 FRunnable实现](#frunnablethread-和-frunnable实现)
    - [FThreadManager](#fthreadmanager)
    - [FRunnable相关实现](#frunnable相关实现)
  - [异步任务的实现](#异步任务的实现)
    - [线程池的实现](#线程池的实现)
    - [TaskGraph的实现](#taskgraph的实现)

<!-- /code_chunk_output -->


### 类图Overall
```plantuml
@startuml
class FRunnableThread
{
    {static} + FRunnableThread* Create(class FRunnable* InRunnable, const TCHAR* ThreadName ...)

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
FRunnableThread 是UE中所有线程的基类，根据不同的平台有不同的实现。例如在Windows平台的FRunnableThreadWin，以及通过pthread实现的FRunnableThreadPThread等等


**FRunnableThread::Create**会根据传入的参数返回创建好的线程，并把创建好的线程添加到线程池中
```

```

#### FThreadManager
FThreadManager是UE中用于管理线程的类，游戏中创建出来的线程都会加入该管理器中进行管理
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


#### FRunnable相关实现
FRunnable是一个可以运行在线程上的任务的基类，可以通过继承他来实现一个可以在线程上运行的任务,UE中的一些异步任务也是继承于它
   
```plantuml
@startuml
class FRunnable
{

}

abstract class IQueuedWork
{
    {abstract} - virtual void DoThreadedWork()
    {abstract} - virtual void Abandon()
}

note left of IQueuedWork::DoThreadedWork
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

note left of FQueuedThreadPoolBase::QueuedThreads
当前线程池中空闲的线程
end note

note left of FQueuedThreadPoolBase::QueuedWork
线程池中还没有被运行的任务，会根据Prriority进行排序
end note

@enduml
```
FQueuedThreadPool为线程池的一个抽象类，定义了线程池需要使用到的一些函数。FQueuedThreadPoolBase则是对线程池的一个具体实现，包括：
- 创建和销毁一个线程池
- 向线程池中添加，撤回任务
- 获取线程池中线程数量等等

### 异步任务的实现

#### 线程池的实现

```plantuml
@startuml
class TAsyncRunnable
{
    + virtual uint32 Run()

    {field} - TUniqueFunction<ResultType()> Function
    - TPromise<ResultType> Promise
    - TFuture<FRunnableThread*> ThreadFuture
}

FRunnable <|-- TAsyncRunnable
note on link: TAsyncRunnable一个运行在单独线程的异步任务的模板类

abstract class IQueuedWork
{
    {abstract} + virtual void DoThreadWork()
    {abstract} + virtual void Abandon()
}

class TAsyncQueuedWork
{
    + virtual void DoThreadWork()
    + virtual void Abandon()

    {field} - TUniqueFunction<ResultType()> Function
    - TPromise<ResultType> Promise
}

IQueuedWork <|-- TAsyncQueuedWork
note on link: TAsyncQueuedWork一个运行在线程池的异步任务的模板

@enduml
```

#### TaskGraph的实现