# Chromium setTimeout/clearTimeout 源码分析

Chromium 源码版本 91.0.4437.3，大约发布于 2021 年 2 月。

setTimeout 函数相关的源码量巨大，涉及线程、消息循环、任务队列以及操作系统的定时器函数。笔者曾经看过 V8 microtask 队列的源码，并写过一篇[文章](https://zhuanlan.zhihu.com/p/134647506)，粗略估计 setTimeout 函数的代码量大约是 microtask 队列的 100 倍，所以本文只分析 setTimeout 关键步骤的代码。

## setTimeout

本文绝大部分篇幅将分析以下 JavaScript 代码片断。

```JavaScript
setTimeout(_ => {}, 100)
```
### 创建延时任务

setTimeout 调用的是 Blink 的 [WindowOrWorkerGlobalScope::setTimeout](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/window_or_worker_global_scope.cc#136)，源码如下：

```C++
int WindowOrWorkerGlobalScope::setTimeout(
    ScriptState* script_state,
    EventTarget& event_target,
    V8Function* handler,
    int timeout,
    const HeapVector<ScriptValue>& arguments) {
  // handler:JavaScript 层 setTimeout 的回调函数
  // timeout:JavaScript 层 setTimeout 的延时时间
  ExecutionContext* execution_context = event_target.GetExecutionContext();
  auto* action = MakeGarbageCollected<ScheduledAction>(
      script_state, execution_context, handler, arguments);
  return DOMTimer::Install(execution_context, action,
                           base::TimeDelta::FromMilliseconds(timeout), true);
}
```

handler 参数是 setTimeout 定时器到期的回调函数，timeout 是定时器延迟时间，通过这两个参数，生成 action，最后调用 [DOMTimer::Install](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer.cc#53)。

DOMTimer::Install 会调用 [DOMTimerCoordinator::InstallNewTimeout](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer_coordinator.cc#14)，向定时器哈希表 timers_ 插入一个定时器，源码如下。


```C++
using TimeoutMap = HeapHashMap<int, Member<DOMTimer>>;
// 存放所有的定时器
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
  // 这里是 JavaScript 层 setTimeout 的返回值
  return timeout_id;
}
```

timeout_id 是 setTimeout 的返回值，它由 [NextID](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer_coordinator.cc#42) 生成，源码如下。

```C++
// 在连续调用 setTimeout 的场景下
// circular_sequential_id_ 的值会不断增加，并返回
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

NextID 函数的核心代码就一行：++circular_sequential_id_，circular_sequential_id_ 是 DOMTimerCoordinator 的私有属性。可以看到，多次调用 setTimeout，它的返回值是递增的，下图来自浏览器控制台。

![setTimeout](https://raw.githubusercontent.com/xudale/blog/master/assets/setTimeout.png)

timers_.insert() 中的 timers_ 是一个哈希表，存放所有的定时器对象。key 是 setTimeout 的返回值 timeout_id， value 是定时器对象 DOMTimer。这里要注意，timers_ 存放的是定时器对象，虽然定时器对象与任务是一一对应的关系，但本文目前为止的所有代码，还没有涉及任务。在后面的 clearTimeout 源码中，我们还会见到定时器哈希表：timers_。

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
  // 定时时间至少要 1ms，与网上传言的 4 ms 不符
  timeout = std::max(timeout, base::TimeDelta::FromMilliseconds(1));
  if (single_shot)
    // setTimeout 逻辑分支在此
    StartOneShot(timeout, FROM_HERE);
  else
    // setInterval 逻辑分支在此
    StartRepeating(timeout, FROM_HERE);
}
```

参数 action 是一个包含定时器到期回调函数和延时间的对象，timeout 表示延时时间，因为 setTimeout 是单次触发，所以 single_shot 为 true，timeout_id 是定时器的返回值。

DOMTimer::DOMTimer 的逻辑分为两部分。首先通过定时器的延时时间，获取任务类型 task_type。本文示例代码的延时时间是 100，所以 task_type 是 TaskType::kJavascriptTimerDelayedLowNesting。通过 MoveToNewTaskRunner(context->GetTaskRunner(task_type)) 来获取相应的任务运行器(源码里是 TaskRunner），存在 web_task_runner_ 中。[TimerBase::MoveToNewTaskRunner](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/platform/timer.cc#81) 源码如下：

```C++
void TimerBase::MoveToNewTaskRunner(
    scoped_refptr<base::SingleThreadTaskRunner> task_runner) {
  // If the underlying task runner stays the same, ignore it.
  if (web_task_runner_ == task_runner) {
    return;
  }
  bool active = IsActive();
  weak_ptr_factory_.InvalidateWeakPtrs();
  // 不同的任务类型，有不同的任务运行器
  // 获取相应的任务运行器
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

从 TaskType 的声明及 TimerBase::MoveToNewTaskRunner 源码可以看出，浏览器中的任务有多种类型，不同类型的任务，放在不同的任务队列。比如因用户点击事件而引起的任务，和因调用 setTimeout 引起的任务，这两种任务在不同的队列中。

找到任务运行器 web_task_runner_ 后，从 StartOneShot 一路追下去，最终在 [TimerBase::SetNextFireTime](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/platform/timer.cc#106)，调用 web_task_runner_->PostDelayedTask 提交了一次延迟任务，PostDelayedTask 第 2 个参数 BindTimerClosure(weak_ptr_factory_.GetWeakPtr() 的结果是一个任务对象，包含 setTimeout 的回调函数，最后一个参数 delay 表示延迟时间。

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
- 通过 setTimeout 的延迟时间，找到相应的任务运行器，存入 web_task_runner_
- 通过 DOMTimer 对象，生成 task，最后向任务运行器 web_task_runner_，提交一个 task，即 web_task_runner_->PostDelayedTask


### 插入延迟任务队列

web_task_runner_->PostDelayedTask 的底层调用的是 [TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#396)，向延时任务队列插入一个任务。并调用 UpdateDelayedWakeUp，更新当前线程的唤醒时间。

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
  // 本文男一号，延时任务队列
  DelayedIncomingQueue delayed_incoming_queue;
  ObserverList<TaskObserver>::Unchecked task_observers;
  base::internal::HeapHandle heap_handle;
  EnqueueOrder current_fence;
}
```

MainThreadOnly 对象有很多任务队列，本文重点关注的是延时任务队列：delayed_incoming_queue。TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread 的核心逻辑，是向 MainThreadOnly 对象的延迟任务队列 delayed_incoming_queue，插入一个延时任务。delayed_incoming_queue 的底层是最小堆，为了便于理解，本文将延迟任务队列 delayed_incoming_queue 当优先队列看待，并且延迟时间最小的任务，优先级最高。事实上 delayed_incoming_queue 的方法，和 C++ 优先队列 priority_queue 的主要方法基本一样。下面一段话摘自优先队列的百度百科：

> 普通的队列是一种先进先出的数据结构，元素在队列尾追加，而从队列头删除。在优先队列中，元素被赋予优先级。当访问元素时，具有最高优先级的元素最先删除。优先队列具有最高级先出 （first in, largest out）的行为特征。通常采用堆数据结构来实现。

对于优先队列，本文只需要了解 3 个方法，以下为 C++ 优先队列方法的说明，适用于本文：

- top 访问队头元素，也就是访问队列中，延迟时间最小的任务
- push 插入元素到队尾 (并排序)
- pop 弹出队头元素

因为 JavaScript 没有优先队列，所以这里强调一下 top 和 pop 方法。top 只是访问队头元素，并不会从优先队列中弹出队头元素，pop 弹出队头元素。简单说，top 只是看看队头元素，pop 拿走了队头元素。

既然 delayed_incoming_queue 是一个优先队列，那么很容易看出 main_thread_only().delayed_incoming_queue.push(std::move(pending_task)) 的作用是向优先队列插入一个延时任务。

本小节总结：

- Chromium 有延时任务队列 delayed_incoming_queue，存放延时任务
- 延时任务队列类似于 C++ 的优先队列，延时最小的任务，优先级最高
- 每一个延时任务都会被插入延时任务队列


### 创建唤醒任务

从上文代码继续往下看，执行 main_thread_only().delayed_incoming_queue.push 方法，延时任务被插入到延时任务队列后，接着调用了 [UpdateDelayedWakeUp](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#1124)，源码如下：

```C++
void TaskQueueImpl::UpdateDelayedWakeUp(LazyNow* lazy_now) {
  return UpdateDelayedWakeUpImpl(lazy_now, GetNextScheduledWakeUpImpl());
}
```

UpdateDelayedWakeUp 的功能是设置线程唤醒时间，想像这样一个场景，假如用户 3 次调用 setTimeout，延迟时间分别是 700ms，100ms，400ms。此时延迟任务队列中会存在 3 个任务，这 3 个任务中，明显应该是延迟时间为 100ms 的任务应该最先执行，其优先级最高，其次才是延迟时间为 400ms 和 700ms 的任务。所以眼下的工作，应该在 100ms 后唤醒线程。

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

GetNextScheduledWakeUpImpl 的功能是创建一个唤醒任务，主要逻辑是读取延迟任务队列的队头，得到延迟任务队列中延迟时间最小的任务 top_task，以 top_task 的延迟时间 delayed_run_time 为参数，创建一个唤醒任务 DelayedWakeUp，并返回。

唤醒任务和延时任务有些相似，最大的区别是唤醒任务只有唤醒时间，没有唤醒时应该调用的回调函数。就本文的示例代码来说，延时任务保存的是回调函数和延时时间 100ms，唤醒任务只保存延时时间 100ms，没有回调函数。

本小节总结：

- 创建唤醒任务
- 唤醒任务仅有延时时间(唤醒时间)字段，前端传给 setTimeout 的回调函数依然在延时任务中

### (可能)插入唤醒任务队列

得到唤醒任务后，调用 [UpdateDelayedWakeUpImpl](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#1128)，源码如下：


```C++
void TaskQueueImpl::UpdateDelayedWakeUpImpl(LazyNow* lazy_now,
                                            Optional<DelayedWakeUp> wake_up) {
  // wake_up 是前文提到的唤醒任务
  // 如果新的唤醒时间和老的唤醒时间一样，则不处理
  if (main_thread_only().scheduled_wake_up == wake_up)
    return;
  main_thread_only().time_domain->SetNextWakeUpForQueue(this, wake_up,
                                                        lazy_now);
}
```

UpdateDelayedWakeUpImpl 先判断新的唤醒时间 wake_up，和之前的唤醒时间 scheduled_wake_up 是否相同，如果相同，则返回，不会变更任务唤醒任务队列。因为 DelayedWakeUp 做了运算符重载，所以 main_thread_only().scheduled_wake_up == wake_up 比较的是时间而不是对象。如果不同则调用 [SetNextWakeUpForQueue](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/time_domain.cc#56)，源码如下：

```C++
void TimeDomain::SetNextWakeUpForQueue(
    internal::TaskQueueImpl* queue,
    Optional<internal::DelayedWakeUp> wake_up,
    LazyNow* lazy_now) {
  if (wake_up) {
    // 继本文男一号延迟任务队列后
    // 本文女一号唤醒任务队列现身
    delayed_wake_up_queue_.insert({wake_up.value(), queue});
  } 

  if (*new_wake_up <= lazy_now->Now()) {
    RequestDoWork();
  } else {
    SetNextDelayedDoWork(lazy_now, *new_wake_up);
  }
}

```

SetNextWakeUpForQueue 的功能是向唤醒任务队列 delayed_wake_up_queue_ 插入唤醒任务。

本小节的标题(可能)插入唤醒任务队列，之所以加上”可能“二字，是因为如果延迟任务队列已经有延迟时间很短的任务，比如延迟任务队列已经有延迟时间为 100ms 的任务，那么当调用以下代码：

```JavaScript
setTimeout(_ => {}, 400)
```

会向延迟任务队列加入一个延迟时间为 400ms 的任务，但此时延迟任务队列的队头依然是延迟时间为 100ms 的任务。延迟时间为 400ms 的任务无法进入唤醒任务队列。此时的唤醒任务队列只有一个唤醒任务：延迟时间为 100ms 的任务。

打个比方，如果延迟任务队列保存的是丐帮全部成员，那么唤醒任务队列保存的就是乔峰、洪七公和黄蓉这些当过帮主的人。郭靖同志虽然也很优秀，但 No。唤醒任务队列保存的是延迟任务队列的历任队头。这一点从实际工作中也很好理解，无论浏览器当前有多少个定时器，在眼下，最重要最急迫的是那个延迟时间最短的那个定时器，也就是唤醒任务队列的队头。

本小节总结：

- Chromium 有唤醒任务队列，队头元素记录下次唤醒线程的时间
- 唤醒任务队列保存的是延迟任务队列的历任队头
- 每一个延迟任务都会插入延迟任务队列，但只有延迟任务队列的队头，才会插入唤醒任务队列


### 调用操作系统的定时器函数

SetNextDelayedDoWork 调用了消息泵(MessagePump)类的 [ScheduleDelayedWork](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump.h#219)，不同的操作系统，定时器的方法肯定是不一样的，笔者的电脑是 Mac，所以从 Mac 开始。因为没有 Windows 和 Androd 真机，所以 Windows 和 Android 部分的定时器相关代码，只是看过，没有打过 log。

#### Mac

Mac 下调用的是 [MessagePumpCFRunLoopBase::ScheduleDelayedWork](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump_mac.mm#172)，源码如下：

```C++
// Must be called on the run loop thread.
void MessagePumpCFRunLoopBase::ScheduleDelayedWork(
  const TimeTicks& delayed_work_time) {
  ScheduleDelayedWorkImpl(delayed_work_time - TimeTicks::Now());
}

void MessagePumpCFRunLoopBase::ScheduleDelayedWorkImpl(TimeDelta delta) {
  // The tolerance needs to be set before the fire date or it may be ignored.
  CFRunLoopTimerSetNextFireDate(
      delayed_work_timer_, CFAbsoluteTimeGetCurrent() + delta.InSecondsF());
}
```

ScheduleDelayedWork 和 ScheduleDelayedWorkImpl 的参数都是延迟时间，只是格式不一样。ScheduleDelayedWork 的参数 delayed_work_time 表示的是延迟时间的绝对数值，ScheduleDelayedWorkImpl 的参数 delta 表示的是延迟时间距离当前时间的数值。ScheduleDelayedWorkImpl 调用 Mac 操作系统的函数 CFRunLoopTimerSetNextFireDate，因为本文示例代码 setTimeout 的第二个参数是 100，所以 CFRunLoopTimerSetNextFireDate 的作用是在 100 ms 后，唤醒当前线程，继续消息循环。以下摘自[苹果官方文档](https://developer.apple.com/documentation/corefoundation/1542501-cfrunlooptimersetnextfiredate)：

![CFRunLoopTimerSetNextFireDate](https://raw.githubusercontent.com/xudale/blog/master/assets/CFRunLoopTimerSetNextFireDate.png)

#### Windows

Windows 操作系统下调用的是 [MessagePumpForUI::ScheduleDelayedWork](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump_win.cc#133)，源码如下：


```C++
void MessagePumpForUI::ScheduleDelayedWork(const TimeTicks& delayed_work_time) {
  if (in_native_loop_ && !work_scheduled_) {
    // TODO(gab): Consider passing a NextWorkInfo object to ScheduleDelayedWork
    // to take advantage of |recent_now| here too.
    ScheduleNativeTimer({delayed_work_time, TimeTicks::Now()});
  }
}

void MessagePumpForUI::ScheduleNativeTimer(
    Delegate::NextWorkInfo next_work_info) {
  UINT delay_msec = strict_cast<UINT>(GetSleepTimeoutMs(
      next_work_info.delayed_run_time, next_work_info.recent_now));
  if (delay_msec == 0) {
    ScheduleWork();
  } else {
    // 调用 Windows 定时函数 SetTimer
    const UINT_PTR ret =
        ::SetTimer(message_window_.hwnd(), reinterpret_cast<UINT_PTR>(this),
                   delay_msec, nullptr);
    if (ret) {
      installed_native_timer_ = next_work_info.delayed_run_time;
      return;
    }
  }
}

```

ScheduleDelayedWork 调用 ScheduleNativeTimer，ScheduleNativeTimer 最终调用 Windows 定时器函数 SetTimer，因为 setTimeout 的第二个参数是 100，所以 SetTimer 的作用是在 100 ms 后，唤醒当前线程，继续消息循环。以下摘自[微软官方文档](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-settimer)：

![setTimer](https://raw.githubusercontent.com/xudale/blog/master/assets/setTimer.png)


#### Android

Android 操作系统调用的是 [MessagePumpForUI::ScheduleDelayedWork](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump_android.cc#334)，源码如下：

```C++
void MessagePumpForUI::ScheduleDelayedWork(const TimeTicks& delayed_work_time) {
  if (ShouldQuit())
    return;
  if (delayed_scheduled_time_ && *delayed_scheduled_time_ == delayed_work_time)
    return;
  delayed_scheduled_time_ = delayed_work_time;
  int64_t nanos = delayed_work_time.since_origin().InNanoseconds();
  struct itimerspec ts;
  ts.it_interval.tv_sec = 0;  // Don't repeat.
  ts.it_interval.tv_nsec = 0;
  ts.it_value.tv_sec = nanos / TimeTicks::kNanosecondsPerSecond;
  ts.it_value.tv_nsec = nanos % TimeTicks::kNanosecondsPerSecond;
  int ret = timerfd_settime(delayed_fd_, TFD_TIMER_ABSTIME, &ts, nullptr);
}

// See sys/timerfd.h
int timerfd_settime(int ufc,
                    int flags,
                    const struct itimerspec* utmr,
                    struct itimerspec* otmr) {
  return syscall(__NR_timerfd_settime, ufc, flags, utmr, otmr);
}
```

ScheduleDelayedWork 调用 timerfd_settime 定时器函数，因为 setTimeout 的第二个参数是 100，所以 timerfd_settime 的作用是在 100 ms 后，唤醒当前线程。timerfd_settime 本质上是通过 linux 的系统调用实现的。


本小节总结：

- 调用操作系统的定时器函数，不同操作系统有不同的实现，比如 Windows 调用 SetTimer，Android 走 linux 的系统调用
- 如果没有其它逻辑，调用完操作系统的定时器函数后，线程会休眠，消息循环暂停
- 源码多处都有注释：操作系统的定时器函数，实际定时粒度较粗，不保证精确

### 线程睡眠(大约) 100 ms

本小节是线程睡前和睡后的分界点，在本文的定位是承上启下。在线程睡眠的时候，我们盘点一下过去，展望一下未来。本文示例代码目前做了以下几件事：

```JavaScript
setTimeout(_ => {}, 100)
```

- 创建了一个延时任务，将延时任务插入延时任务队列
- 取延时任务队列的队头，创建一个唤醒任务
- (假装刚才创建的唤醒时间为 100ms 后的唤醒任务，是距当前时间点，最近的唤醒任务)
- 将唤醒任务，插入唤醒任务队列
- 唤醒任务只保存延时(唤醒)时间，不保存回调函数
- 取唤醒任务队列的队头，也就是距当前时间点最近，眼下最迫切的唤醒任务
- 调用操作系统的定时器函数

现在有延时任务队列和唤醒任务队列，因为延时任务队列包含所有延时任务的时间和回调函数，其实逻辑上可以完全忽略唤醒任务队列。在线程被唤醒时，利用延时任务队列的数据结构是优先队列的特点，循环取延时任务队列的队头，根据队头的时间判断队头任务是否到期，就可以得到所有已经到期甚至过期延时任务。

由于操作系统的定时器函数未必准确，实际的睡眠时间可能超过 100ms。

![setTimeoutFlow](https://raw.githubusercontent.com/xudale/blog/master/assets/setTimeoutFlow.png)

### 唤醒线程，继续消息循环

定时时间到，消息循环重启，已经过期的延时任务，会被添加到工作队列，[TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/task_queue_impl.cc#572) 源码如下：

```C++
void TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue(LazyNow* lazy_now) {
  // Enqueue all delayed tasks that should be running now, skipping any that
  // have been canceled.
  WorkQueue::TaskPusher delayed_work_queue_task_pusher(
      main_thread_only().delayed_work_queue->CreateTaskPusher());
  // 如果仔细看了前面的代码，应该还记得
  // delayed_incoming_queue 是前面提过多次的延时队列
  // 遍历队列，将已经到期甚至过期的任务，添加至 delayed_work_queue_task_pusher
  while (!main_thread_only().delayed_incoming_queue.empty()) {
    // 获取延时任务队列的队头
    Task* task =
        const_cast<Task*>(&main_thread_only().delayed_incoming_queue.top());
    // TODO(crbug.com/990245): Remove after the crash has been resolved.
    sequence_manager_->RecordCrashKeys(*task);
    if (!task->task || task->task.IsCancelled()) {
      // 删除已经取消的任务
      main_thread_only().delayed_incoming_queue.pop();
      continue;
    }
    // delayed_incoming_queue 是优先队列
    // 如果队头元素的延迟时间大于当前时间
    // 说明延迟任务队列中的所有任务都未到期，结束循环
    if (task->delayed_run_time > lazy_now->Now())
      break;

    delayed_work_queue_task_pusher.Push(task);
    main_thread_only().delayed_incoming_queue.pop();
  }
  UpdateDelayedWakeUp(lazy_now);
}
```


TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue 的逻辑是收集延迟任务队列中所有到期，甚至过期的任务，放入工作队列中。最终，工作队列，包含了当前时间所有已到期并待执行的延时任务。

每一次调用 setTimeout 都会向延迟任务队列 main_thread_only().delayed_incoming_queue 中添加任务。在线程被唤醒后，会从延迟任务队列中取任务。由于延迟任务队列是个优先队列，只需循环取队头就可以，只要队头任务还没有到期，说明延迟任务中的所有任务都没有到期，if (task->delayed_run_time > lazy_now->Now()) break，循环结束。

收集完所有到期和过期的延迟任务后，执行 [ThreadControllerWithMessagePumpImpl::DoWorkImpl](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/task/sequence_manager/thread_controller_with_message_pump_impl.cc#305)，源码如下：

```C++
TimeDelta ThreadControllerWithMessagePumpImpl::DoWorkImpl(
    LazyNow* continuation_lazy_now) {
  for (int i = 0; i < main_thread_only().work_batch_size; i++) {
    const SequencedTaskSource::SelectTaskOption select_task_option =
        power_monitor_.IsProcessInPowerSuspendState()
            ? SequencedTaskSource::SelectTaskOption::kSkipDelayedTask
            : SequencedTaskSource::SelectTaskOption::kDefault;
    // 选择待执行的任务
    // SelectNextTask 会调用 TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue
    // SelectNextTask 结束后，得到了待执行的任务，存入 task
    Task* task =
        main_thread_only().task_source->SelectNextTask(select_task_option);
    // 如果没有任务，退出
    if (!task)
      break;
    {
      // 执行任务
      task_annotator_.RunTask("SequenceManager RunTask", task);
      // 注意这段注释，说的是执行完任务后，处理所有 microtask
      // 这两行代码引发大量关于执行顺序的面试题
      // This processes microtasks, hence all scoped operations above must end
      // after it.
      main_thread_only().task_source->DidRunTask();
    }
  }
}

```

ThreadControllerWithMessagePumpImpl::DoWorkImpl 的逻辑是找到待执行的任务，存入 task，然后 task_annotator_.RunTask("SequenceManager RunTask", task) 执行任务。


本小节总结：

- 从延迟任务队列 delayed_incoming_queue 取出已经到期的任务
- 由于操作系统定时器不精确，有些任务不止到期，甚至已经过期许久
- 执行已经过期的任务

### 面试题：为什么 setTimeout 不准确

这是一道前端常见面试题，笔者认为有 3 个原因：

1.硬件层面上，没有绝对准确的时钟，比如手机/电脑断网但不断电，过几个月再看，手机/电脑上的时间都与实际时间相差较大。当然，这个理由有点属于抬杠。

2.运行浏览器的操作系统，Windwos、Mac 和 Android 等，是 OS，但不是 RTOS(实时操作系统)。操作系统厂商不保证定时器/sleep 函数的时间一定准确。

3.如果短时间内，CPU 要处理大量定时任务，即使用汇编写，只要单个定时任务耗时足够长，必然会出现有的定时任务没有如期执行的情况。

为什么 setTimeout 不准确这句话，似乎在暗示操作系统存在某个与时间相关的函数，是绝对精确的，然而并没有这样的函数。从 Chromium 源码相关注释来看，Chromium 已经很努力的在让 setTimeout 尽可能的准确。既然操作系统不保证时间准确，隔了一层浏览器，却要问前端为什么 setTimeout 不准确，前端何苦为难前端。

本小节总结：

- 前端何苦为难前端

## clearTimeout

clearTimeout 调用的是 [WindowOrWorkerGlobalScope::clearTimeout](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/window_or_worker_global_scope.cc#212)，源码如下：

```C++
void WindowOrWorkerGlobalScope::clearTimeout(EventTarget& event_target,
                                             int timeout_id) {
  // timeout_id 是 Javascript 层 clearTimeout 的参数
  if (ExecutionContext* context = event_target.GetExecutionContext())
    DOMTimer::RemoveByID(context, timeout_id);
}
// clearTimeout 和 clearInterval 的源码是完全一样的
void WindowOrWorkerGlobalScope::clearInterval(EventTarget& event_target,
                                              int timeout_id) {
  if (ExecutionContext* context = event_target.GetExecutionContext())
    DOMTimer::RemoveByID(context, timeout_id);
}
```

从源码中可见，clearTimeout 和 clearInterval 是完全一样的，可以混用。[规范](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html)中也是如此规定的。

![clearTimeoutclearInterval](https://raw.githubusercontent.com/xudale/blog/master/assets/clearTimeoutclearInterval.png)

DOMTimer::RemoveByID 调用的是 [DOMTimerCoordinator::RemoveTimeoutByID](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/frame/dom_timer_coordinator.cc#27)，源码如下：

```C++
DOMTimer* DOMTimerCoordinator::RemoveTimeoutByID(int timeout_id) {
  if (timeout_id <= 0)
    return nullptr;
  // timers_ 是本文最开始提到的，存放所以定时器对象的哈希表
  DOMTimer* removed_timer = timers_.Take(timeout_id);
  if (removed_timer)
    removed_timer->Stop();
  return removed_timer;
}
```

本文最开始提到，有个哈希表 timers_，存放所有的定时器，本文结束的时候我们又看见了它，首尾呼应。DOMTimerCoordinator::RemoveTimeoutByID 的逻辑是从哈希表 timers_ 中取出 key 为 timeout_id 的定时器对象 removed_timer，removed_timer->Stop() 的作用是将定时器对象的回调函数字段置为空，定时器对象的回调函数，和延时任务的回调函数，是同一个函数。将回调函数的字段置为空，尽管延时任务还在，但从前端视角来看，任务已经取消了。

定时器对象和延时任务是一一对应的，定时器对象的 delayed_task_handle_ 字段引用相应延时任务。找到延时任务，并将延时任务标记为清除。标记为清除的延时任务，会在下一次循环中，在前文提到的 TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue 方法中，从延时任务队列中移除。


本小节总结：

- 从哈希表 timers_ 找到待移除的定时器
- 将待移除定时器的回调函数字段置为空
- 将待移除定时器关联的延时任务标记为清除 
- 标记为清除的延时任务，会在下一次消息循环，从延时任务队列中移除

## 全文总结

![overviewSettimeout](https://raw.githubusercontent.com/xudale/blog/master/assets/overviewSettimeout.png)

## 参考文献

[https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html](https://html.spec.whatwg.org/multipage/timers-and-user-prompts.html)

[Threading and Tasks in Chrome](https://chromium.googlesource.com/chromium/src/+/lkgr/docs/threading_and_tasks.md#Threads)



























