# Chromium setTimeout/clearTimeout 源码分析

Chromium 源码版本 91.0.4437.3。

setTimeout 函数相关的源码量巨大，涉及线程、消息循环、任务队列及操作系统的定时器。笔者曾看看过 V8 microtask 队列的源码，并写过一篇文章，目测 setTimeout 函数的源码量大约是 microtask 队列源码量的 100 倍，所以本文只分析核心代码。

## setTimeout

本文绝大部分篇幅只分析以下 JavaScript 代码。

```JavaScript
setTimeout(_ => {
  console.log('test')
}, 100)
```

### 生成任务

setTimeout 调用的是 Blink 的 [WindowOrWorkerGlobalScope::setTimeout](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/window_or_worker_global_scope.cc#136)，源码如下：

```C++
int WindowOrWorkerGlobalScope::setTimeout(
    ScriptState* script_state,
    EventTarget& event_target,
    V8Function* handler,
    int timeout,
    const HeapVector<ScriptValue>& arguments) {
  ExecutionContext* execution_context = event_target.GetExecutionContext();
  auto* action = MakeGarbageCollected<ScheduledAction>(
      script_state, execution_context, handler, arguments);
  return DOMTimer::Install(execution_context, action,
                           base::TimeDelta::FromMilliseconds(timeout), true);
}
```

handler 参数是 setTimeout 定时器到期的回调函数，timeout 表示延迟时间，通过这两个参数，生成 action，最后调用 [DOMTimer::Install](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer.cc#53)。

DOMTimer::Install 会调用 [DOMTimerCoordinator::InstallNewTimeout](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer_coordinator.cc#14)，向定时器哈希表插入一个定时器。


```C++
using TimeoutMap = HeapHashMap<int, Member<DOMTimer>>;
TimeoutMap timers_;

int DOMTimerCoordinator::InstallNewTimeout(ExecutionContext* context,
                                           ScheduledAction* action,
                                           base::TimeDelta timeout,
                                           bool single_shot) {
  // 生成 setTimeout 的返回值
  int timeout_id = NextID();
  // timers 是一个哈希表，key 是 timeout_id，值是定时器对象
  timers_.insert(timeout_id,
                 MakeGarbageCollected<DOMTimer>(context, action, timeout,
                                                single_shot, timeout_id));
  // 这里是 setTimeout 的返回值
  return timeout_id;
}
```

timeout_id 是 setTimeout 的返回值，它是由 [NextID](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer_coordinator.cc#42) 生成的。

```C++
int circular_sequential_id_ = 0;

int DOMTimerCoordinator::NextID() {
  while (true) {
    ++circular_sequential_id_;
    if (circular_sequential_id_ <= 0)
      circular_sequential_id_ = 1;
    // timers_ 是定时器哈希表，如果哈希表已经有 key 为当前
    // circular_sequential_id_ 的定时器，继续死循环
    // 如果没有，返回当前 circular_sequential_id_
    if (!timers_.Contains(circular_sequential_id_))
      return circular_sequential_id_;
  }
}
```

NextID 函数的核心代码就一行：++circular_sequential_id_，circular_sequential_id_ 是 DOMTimerCoordinator 的私有属性。可以看到，多次调用 setTimeout，它的返回值是越来越大的，如下图。

![setTimeout](https://raw.githubusercontent.com/xudale/blog/master/assets/setTimeout.png)

timers_.insert() 中的 timers_ 是一个哈希表，存放所有的定时器对象。key 是 定时器的返回值 timeout_id， value 是定时器对象 DOMTimer。还要注意 timers_ 存放的是定时器对象，与任务无关。

> 所有的定时器对象 DOMTimer，都存在哈希表 timers_ 中

定时器构造函数，[DOMTimer::DOMTimer](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer.cc#71)，源码如下：

```C++
DOMTimer::DOMTimer(ExecutionContext* context,
                   ScheduledAction* action,
                   base::TimeDelta timeout,
                   bool single_shot,
                   int timeout_id)
    : ExecutionContextLifecycleObserver(context),
      TimerBase(nullptr),
      timeout_id_(timeout_id),
      // Step 9:
      nesting_level_(context->Timers()->TimerNestingLevel()),
      action_(action) {

  // 获取任务类型
  TaskType task_type;
  if (timeout.is_zero()) {
    task_type = TaskType::kJavascriptTimerImmediate;
  } else if (nesting_level_ >= kMaxTimerNestingLevel) {
    task_type = TaskType::kJavascriptTimerDelayedHighNesting;
  } else {
    task_type = TaskType::kJavascriptTimerDelayedLowNesting;
  }
  MoveToNewTaskRunner(context->GetTaskRunner(task_type));
  // Clamping up to 1ms for historical reasons crbug.com/402694.
  timeout = std::max(timeout, base::TimeDelta::FromMilliseconds(1));
  if (single_shot)
    StartOneShot(timeout, FROM_HERE);
  else
    StartRepeating(timeout, FROM_HERE);
}
```

参数 action 是一个包含定时器延时处理函数和延时时间的对象，timeout 表示延时时间，因为 setTimeout 是单次触发，所以 single_shot 为 true，timeout_id 是定时器的返回值。

DOMTimer::DOMTimer 的逻辑分为两部分。首先通过定时器的延时时间，获取任务类型 task_type。本文示例代码的 task_type 是 TaskType::kJavascriptTimerDelayedLowNesting。通过 MoveToNewTaskRunner(context->GetTaskRunner(task_type)) 来获取相应的任务运行器，存在 web_task_runner_ 中。[TimerBase::MoveToNewTaskRunner](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/platform/timer.cc#81) 源码如下：

```C++
void TimerBase::MoveToNewTaskRunner(
    scoped_refptr<base::SingleThreadTaskRunner> task_runner) {
  // If the underlying task runner stays the same, ignore it.
  if (web_task_runner_ == task_runner) {
    return;
  }
  bool active = IsActive();
  weak_ptr_factory_.InvalidateWeakPtrs();
  web_task_runner_ = std::move(task_runner);
}
```

前文提到通过任务类型，找到相应的任务运行器。任务类型 [TaskType](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/public/platform/task_type.h#19) 是个枚举类型，源码如下：

```C++
enum class TaskType : unsigned char {
  kDeprecatedNone = 0,

  kDOMManipulation = 1,

  kUserInteraction = 2,

  kMicrotask = 9,

  kJavascriptTimerImmediate = 72,

  kJavascriptTimerDelayedLowNesting = 73,

  kJavascriptTimerDelayedHighNesting = 10,

  kRemoteEvent = 11,

  kWebSocket = 12,

  kPostedMessage = 13,

  kUnshippedPortMessage = 14,

  kFileReading = 15,

  kWebGL = 20,

  kIdleTask = 21,

  kMiscPlatformAPI = 22,
}
```

从 TaskType 的声明及 TimerBase::MoveToNewTaskRunner 源码可以看出，浏览器中的任务有多种类型，不同类型的任务，放在不同地方。

> 浏览器中的任务有多种类型，不同的类型的任务，放在不同的队列里。比如因为用户点击事件引起的任务，和 setTimeout 引起的任务，就在不同的队列中

找到任务运行器 web_task_runner_ 后，在 StartOneShot 一路追下去，最终在 [TimerBase::SetNextFireTime](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/platform/timer.cc#106)，调用 web_task_runner_->PostDelayedTask 提交了一次延迟任务，PostDelayedTask 第 2 个参数 BindTimerClosure(weak_ptr_factory_.GetWeakPtr() 是一个任务对象，包含 setTimeout 的回调函数，delay 表示延迟时间。

```C++
void TimerBase::SetNextFireTime(base::TimeTicks now, base::TimeDelta delay) {
  base::TimeTicks new_time = now + delay;
  if (next_fire_time_ != new_time) {
    next_fire_time_ = new_time;
    // Cancel any previously posted task.
    weak_ptr_factory_.InvalidateWeakPtrs();
    // 提交一次延迟任务
    web_task_runner_->PostDelayedTask(
        location_, BindTimerClosure(weak_ptr_factory_.GetWeakPtr()), delay);
  }
}
```

本小节总结：

- 通过 setTimeout 的参数，生成定时器对象 DOMTimer，并将新生成的 DOMTimer 插入哈希表 timers_ 中
- 通过 setTimeout 的延迟时间，找到合适的任务运行器，存入 web_task_runner_
- 通过 DOMTimer 对象，生成 task，最后向任务运行器 web_task_runner_，投递 task，即 web_task_runner_->PostDelayedTask


### 插入延迟任务队列

web_task_runner_->PostDelayedTask 的底层调用的是 [TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#396)

```C++
void TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread(
  Task pending_task,
  TimeTicks now,
  bool notify_task_annotator) {
  // pending_task 表示任务
  // delayed_incoming_queue 是延迟任务队列
  // 向延迟任务队列添加任务
  main_thread_only().delayed_incoming_queue.push(std::move(pending_task));
  LazyNow lazy_now(now);
  UpdateDelayedWakeUp(&lazy_now);
  TraceQueueSize();
}
```

main_thread_only() 返回 [MainThreadOnly](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.h#344) 对象，声明如下：

```C++
struct MainThreadOnly {
  MainThreadOnly(TaskQueueImpl* task_queue, TimeDomain* time_domain);
  ~MainThreadOnly();
  // Another copy of TimeDomain for lock-free access from the main thread.
  // See description inside struct AnyThread for details.
  TimeDomain* time_domain;
  std::unique_ptr<WorkQueue> delayed_work_queue;
  std::unique_ptr<WorkQueue> immediate_work_queue;
  // 存放所以延迟任务
  DelayedIncomingQueue delayed_incoming_queue;
  ObserverList<TaskObserver>::Unchecked task_observers;
  base::internal::HeapHandle heap_handle;
  EnqueueOrder current_fence;
}
```

MainThreadOnly 对象有很多任务队列，TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread 的核心逻辑，是向 MainThreadOnly 对象的延迟任务队列 delayed_incoming_queue，提交一次任务。delayed_incoming_queue 的底层是最小堆，为了便于理解，本文将延迟队列 delayed_incoming_queue 当优先级队列看待，延迟时间最小的任务，优先级最高。事实上 delayed_incoming_queue 的方法，和 C++ 优先队列 priority_queue 的主要方法基本一样。下面摘自百度百科：

> 普通的队列是一种先进先出的数据结构，元素在队列尾追加，而从队列头删除。在优先队列中，元素被赋予优先级。当访问元素时，具有最高优先级的元素最先删除。优先队列具有最高级先出 （first in, largest out）的行为特征。通常采用堆数据结构来实现。

对于优先队列，本文只需要了解 3 个方法，以下为 C++ 优先队列方法的说明，适用于本文：

- top 访问队头元素，也就是访问队列中，延迟时间最小的任务
- push 插入元素到队尾 (并排序)
- pop 弹出队头元素

因为 JavaScript 没有优先队列，所以这里强调一个 top 和 pop 方法。top 只是访问队头元素，并不会从优先队列中弹出队头元素，pop 弹出队头元素。

既然已经知道 delayed_incoming_queue 是一个优先队列，那么很容易看出 main_thread_only().delayed_incoming_queue.push(std::move(pending_task)) 的作用是向优先队列加入一个 task。

本小节总结：

- Chromium 有延时任务队列 delayed_incoming_queue，存放延时任务
- 延时任务队列类似于 C++ 的优先队列，延时最小的任务，优先级最高
- 每一个延时任务都会插入延时任务队列


### (可能)插入唤醒任务队列

从上文代码继续往下看，执行 main_thread_only().delayed_incoming_queue.push 方法，延时任务被插入到延时任务队列后，接着调用了 [UpdateDelayedWakeUp](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#1124)，源码如下：

```C++
void TaskQueueImpl::UpdateDelayedWakeUp(LazyNow* lazy_now) {
  return UpdateDelayedWakeUpImpl(lazy_now, GetNextScheduledWakeUpImpl());
}
```

UpdateDelayedWakeUp 的功能是获取线程唤醒时间，想像这样一个场景，假如用户 3 次调用 setTimeout，延迟时间分别是 700ms，100ms，400ms。此时延迟任务队列中会存在 3 个任务，这 3 个任务中，明显延迟时间为 100ms 的任务应该最先执行，其优先级最高，其次才是延迟时间为 400ms 和 700ms 的任务。所以应该在 100ms 后唤醒线程。

UpdateDelayedWakeUp 先调用的是 [GetNextScheduledWakeUpImpl](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#546)，源码如下：

```C++
Optional<DelayedWakeUp> TaskQueueImpl::GetNextScheduledWakeUpImpl() {
  // Note we don't scheduled a wake-up for disabled queues.
  if (main_thread_only().delayed_incoming_queue.empty() || !IsQueueEnabled())
    return nullopt;
  // High resolution is needed if the queue contains high resolution tasks and
  // has a priority index <= kNormalPriority (precise execution time is
  // unnecessary for a low priority queue).
  WakeUpResolution resolution =
      has_pending_high_resolution_tasks() &&
              GetQueuePriority() <= TaskQueue::QueuePriority::kNormalPriority
          ? WakeUpResolution::kHigh
          : WakeUpResolution::kLow;
  // 从优先队列中，获取延迟时间最短的任务
  const auto& top_task = main_thread_only().delayed_incoming_queue.top();
  return DelayedWakeUp{top_task.delayed_run_time, top_task.sequence_num,
                       resolution};
}
```

GetNextScheduledWakeUpImpl 的功能是生成一个唤醒任务，主要逻辑是读取延迟任务队列的队头，得到延迟任务队列中延迟时间最小的任务 top_task，以 top_task 的延迟时间 delayed_run_time，返回一个延迟任务 DelayedWakeUp。

得到唤醒任务后，调用 [UpdateDelayedWakeUpImpl](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#1128)，源码如下：


```C++
void TaskQueueImpl::UpdateDelayedWakeUpImpl(LazyNow* lazy_now,
                                            Optional<DelayedWakeUp> wake_up) {
  // 如果新的唤醒时间和老的唤醒时间一样，则不处理
  if (main_thread_only().scheduled_wake_up == wake_up)
    return;
  main_thread_only().time_domain->SetNextWakeUpForQueue(this, wake_up,
                                                        lazy_now);
}
```

UpdateDelayedWakeUpImpl 先判断新的唤醒时间 wake_up，和之前的唤醒时间 scheduled_wake_up 是否相同，如果相同，则返回，不会变更任务唤醒队列。如果不同则调用 [SetNextWakeUpForQueue](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/time_domain.cc#56)，源码如下：

```C++
void TimeDomain::SetNextWakeUpForQueue(
    internal::TaskQueueImpl* queue,
    Optional<internal::DelayedWakeUp> wake_up,
    LazyNow* lazy_now) {
  if (wake_up) {
    delayed_wake_up_queue_.insert({wake_up.value(), queue});
  } 

  if (*new_wake_up <= lazy_now->Now()) {
    RequestDoWork();
  } else {
    SetNextDelayedDoWork(lazy_now, *new_wake_up);
  }
}

```

SetNextWakeUpForQueue 的功能是向唤醒队列 delayed_wake_up_queue_ 插入唤醒任务。

本小节的标题(可能)插入唤醒任务队列，之所以加上”可能“二字，是因为如果延迟任务队列已经有延迟时间很短的任务，比如延迟任务队列有延迟时间为 700ms 和 100ms 的两个任务，那么当调用以下代码：

```JavaScript
setTimeout(_ => {}, 400)
```

会向延迟任务队列加入一个延迟时间为 400ms 的任务，但此时延迟任务队列的队头依然是延迟时间为 100ms 的任务。延迟时间为 400ms 的任务只会进入延迟任务队列，不会进入唤醒队列。此时的唤醒队列只有一个任务：延迟时间为 100ms 的任务。

打个比方，如果延迟任务队列保存的是丐帮全部成员，那么唤醒任务队列保存的就是汪剑通、乔峰、洪七公和黄蓉这些当过帮主的人。唤醒任务队列是从延迟任务队列中优中选优，延迟任务队列的每一任队头，都曾经是唤醒任务队列的一员。

本小节总结：

- Chromium 有唤醒任务队列，记录下次唤醒线程的时间
- 唤醒任务队列保存的是延迟任务队列的历任队头
- 每一个延迟任务都会插入延迟任务队列，但只有延迟任务队列的队头，才会插入唤醒任务队列


### 调用操作系统的定时器函数

#### Mac

#### Windows

#### Android


### 线程睡眠 100 ms

### 唤醒线程，继续消息循环

### 为什么 setTimeout 不准确

## clearTimeout



Chromium 与线程相关的类有两个，[Thread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.h#62) 和 [PlatformThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread.h#121)。Thread 底层调用 PlatformThread 创建线程。PlatformThread 封装了操作系统级别的线程相关 API，在不同的操作系统有不同的实现，但接口保持一致。本文从创建线程的源码开始。

### Thread

在 Chromium 中，开启一个线程的方法是 [Thread::Start](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#131)，源码如下：

```C++
bool Thread::Start() {
  Options options;
  return StartWithOptions(options);
}
```

options 对象表示创建线程需要的一些参数，如线程优先级、线程栈大小和消息循环等，[声明如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.h#76)：

```C++
struct BASE_EXPORT Options {
  using MessagePumpFactory =
      RepeatingCallback<std::unique_ptr<MessagePump>()>;

  Options(MessagePumpType type, size_t size);
  ~Options();

  // 消息循环的类型，不同的操作系统有不同的消循环
  // Chromium Thread 类自带消息循环
  MessagePumpType message_pump_type = MessagePumpType::DEFAULT;

  TimerSlack timer_slack = TIMER_SLACK_NONE;

  // 线程栈大小
  size_t stack_size = 0;

  // 线程优先级
  ThreadPriority priority = ThreadPriority::NORMAL;

  // If false, the thread will not be joined on destruction.
  bool joinable = true;
};
```

Thread::Start 调用了 [Thread::StartWithOptions](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#142)，源码如下：

```C++
bool Thread::StartWithOptions(const Options& options) {
  timer_slack_ = options.timer_slack;  
  // MessagePump::Create(type) 创建了消息循环
  delegate_ = std::make_unique<SequenceManagerThreadDelegate>(
    options.message_pump_type,
    BindOnce([](MessagePumpType type) { return MessagePump::Create(type); },
             options.message_pump_type),
    options.task_queue_time_domain);
  {
    bool success =
      options.joinable
        ? PlatformThread::CreateWithPriority(options.stack_size, this,
                                             &thread_, options.priority)
        : PlatformThread::CreateNonJoinableWithPriority(
          options.stack_size, this, options.priority);
  }
  joinable_ = options.joinable;
  return true;
}
```

Thread::StartWithOptions 解析 options 参数，做的主要是一些线程初始化的工作，并没有真正创建线程。真正创建线程的方法是 [PlatformThread::CreateWithPriority](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread_posix.cc#249)，见下节。

### PlatformThread

PlatformThread 的中文含义是平台线程，顾名思义，其在不同的操作系统会有不同的实现，实际也是如此。但 [PlatformThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread.h#121) 的声明在不同操作系统是一样的。也就是说 PlatformThread 在不同的操作系统有着相同的接口，不同的实现。PlatformThread 声明部分源码如下：

```C++
// 不同操作系统下接口一致，实现不一致
class BASE_EXPORT PlatformThread {
 public:
  static PlatformThreadHandle CurrentHandle();

  // 线程让步
  static void YieldCurrentThread();

  // 线程睡眠
  static void Sleep(base::TimeDelta duration);

  // 创建操作系统级别线程
  static bool CreateWithPriority(size_t stack_size, Delegate* delegate,
                                 PlatformThreadHandle* thread_handle,
                                 ThreadPriority priority);

  // Joins with a thread created via the Create function.  
  static void Join(PlatformThreadHandle thread_handle);

  // Detaches and releases the thread handle
  static void Detach(PlatformThreadHandle thread_handle);
}
```

PlatformThread 类的方法基本都是静态方法，代码逻辑大多是对操作系统线程相关函数的包装，不同操作系统的实现不同。本文以 Mac 和 Windows 线程创建函数为例。

在 Mac 下，[PlatformThread::CreateWithPriority](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread_posix.cc#249) 最终是通过调用 Mac OS 的 [pthread_create](https://baike.baidu.com/item/pthread_create/5139072?fr=aladdin) 函数创建线程，源码如下：

```C++
// static
bool PlatformThread::CreateWithPriority(size_t stack_size, Delegate* delegate,
                                        PlatformThreadHandle* thread_handle,
                                        ThreadPriority priority) {
  return CreateThread(stack_size, true /* joinable thread */, delegate,
                      thread_handle, priority);
}

bool CreateThread(size_t stack_size,
                  bool joinable,
                  PlatformThread::Delegate* delegate,
                  PlatformThreadHandle* thread_handle,
                  ThreadPriority priority) {

  pthread_attr_t attributes;
  pthread_attr_init(&attributes);

  std::unique_ptr<ThreadParams> params(new ThreadParams);
  params->delegate = delegate;
  params->joinable = joinable;
  params->priority = priority;

  pthread_t handle;
  // 核心代码在此，调用 Mac OS 的 pthread_create 创建线程
  // ThreadFunc 函数内部会开始消息循环
  int err = pthread_create(&handle, &attributes, ThreadFunc, params.get());
  bool success = !err;
  return success;
}
```

CreateThread 的核心代码是调用 Mac OS 的 [pthread_create](https://baike.baidu.com/item/pthread_create/5139072?fr=aladdin) 创建线程。pthread_create 的 4 个参数解释如下，handle 表示指向线程标识符的指针，attributes 表示线程属性，ThreadFunc 表示线程运行函数的起始地址，params.get() 表示线程运行时的参数。

在 Windows 下，[PlatformThread::CreateWithPriority](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread_win.cc#289) 归根到底是调用了 Windows 操作系统的 [CreateThread](https://docs.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 函数创建线程，源码如下：

```C++
// static
bool PlatformThread::CreateWithPriority(size_t stack_size, Delegate* delegate,
                                        PlatformThreadHandle* thread_handle,
                                        ThreadPriority priority) {
  DCHECK(thread_handle);
  return CreateThreadInternal(stack_size, delegate, thread_handle, priority);
}
// 对比上文 Mac OS 的实现可以看出，PlatformThread 的接口在不同平台是一样的
// 如前面的 PlatformThread::CreateWithPriority，但实现不一样
// 如在 Mac OS 的实现中 PlatformThread::CreateWithPriority 调用的是
// CreateThread。Windows 平台下 PlatformThread::CreateWithPriority 调用的是 
// CreateThreadInternal
bool CreateThreadInternal(size_t stack_size,
                          PlatformThread::Delegate* delegate,
                          PlatformThreadHandle* out_thread_handle,
                          ThreadPriority priority) {
  unsigned int flags = 0;
  ThreadParams* params = new ThreadParams;
  params->delegate = delegate;
  params->joinable = out_thread_handle != nullptr;
  params->priority = priority;
  // 核心代码在此，调用 Windows 的 CreateThread 创建线程
  void* thread_handle =
    ::CreateThread(nullptr, stack_size, ThreadFunc, params, flags, nullptr);
  if (out_thread_handle)
    *out_thread_handle = PlatformThreadHandle(thread_handle);
  else
    CloseHandle(thread_handle);
  return true;
}
```

CreateThread 是 Windows 平台创建线程的函数，接收 5 个参数。nullptr 既然已是空我们就不谈了，stack_size 表示线程初始时的栈大小，ThreadFunc 表示线程运行函数的起始地址，params 表示线程的参数，flags 表示线程的属性。Windows 平台 CreateThread 接收的参数与 Mac OS pthread_create 接收的参数大体类似。

> PlatformThread 类在不同操作系统下接口一致，实现不一致

> Thread 类的实现依赖 PlatformThread，PlatformThread 已经抹平了不同操作系统线程操作的差异，所以 Thread 类是跨平台的线程类


## chromium 消息循环

从上文的 [Thread::StartWithOptions](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#142) 方法中，可以看到 Thread 类是自带消息循环的。

```C++
bool Thread::StartWithOptions(const Options& options) {
  timer_slack_ = options.timer_slack;  
  // 重点在 MessagePump::Create(type) 消息泵创建这里
  delegate_ = std::make_unique<SequenceManagerThreadDelegate>(
    options.message_pump_type,
    BindOnce([](MessagePumpType type) { return MessagePump::Create(type); },
            options.message_pump_type),
    options.task_queue_time_domain);
  // 后面略
}
```

[MessagePump::Create](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump.cc#44) 方法创建了一个消息循环对象，代码如下：


```C++
// 消息循环的类型有多种，这里删除了很多代码
std::unique_ptr<MessagePump> MessagePump::Create(MessagePumpType type) {
  switch (type) {
    case MessagePumpType::UI:
      return message_pump_for_ui_factory_();
    // 本文代码走向这个分支，默认的消息循环类型
    case MessagePumpType::DEFAULT:
      return std::make_unique<MessagePumpDefault>();
  }
}
```

MessagePump::Create 创建消息循环后，线程入口函数，也就是 Mac OS 的 pthread_create 和 Windows 的 CreateThread 接收的线程入口函数 ThreadFunc，开始运行消息循环，消息循环[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump_default.cc#31)：

```C++
void MessagePumpDefault::Run(Delegate* delegate) {
  AutoReset<bool> auto_reset_keep_running(&keep_running_, true);
  for (;;) {
    Delegate::NextWorkInfo next_work_info = delegate->DoWork();
    bool has_more_immediate_work = next_work_info.is_immediate();
    if (!keep_running_)
      break;

    if (has_more_immediate_work)
      continue;

    has_more_immediate_work = delegate->DoIdleWork();
    if (!keep_running_)
      break;

    if (has_more_immediate_work)
      continue;

    if (next_work_info.delayed_run_time.is_max()) {
      event_.Wait();
    } else {
      event_.TimedWait(next_work_info.remaining_delay());
    }
    // Since event_ is auto-reset, we don't need to do anything special here
    // other than service each delegate method.
  }
}
```

被 for (;;) 死循环包裹的代码块是消息循环，Chromium 在消息循环中处理当前的各种任务。消息循环是一个死循环，但消息循环所处的线程大多数时间都不处于运行状态。笔者亲测，只有发生了鼠标点击/滚动/键盘输入等事件时，消息循环中的代码才会执行。


## 总结

Chromium 中的 Thread 类是一个可以跨平台创建线程，并自带消息循环的类。

Chromium 消息循环是一个死循环，但消息循环所处的线程多数情况下并不处于运行状态。
























