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

SetNextDelayedDoWork 调用了消息泵(MessagePump)类的 [ScheduleDelayedWork](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump.h#219)，不同的操作系统，定时唤醒线程的方法肯定是不一样的，笔者的电脑是 Mac，所以先从 Mac 开始。笔者在 Mac 下打过 log，因为没有 Windows 和 Androd，所以 Windows 和 Android 部分的操作系统相关代码，只是看过，没有真机跑过。

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

ScheduleDelayedWork 和 ScheduleDelayedWorkImpl 的参数都是延迟时间，只是格式不一样。ScheduleDelayedWork 的参数 delayed_work_time 表示的是延迟时间的绝对数值，ScheduleDelayedWorkImpl 的参数 delta 表示的是延迟时间距离当前时间的数值。ScheduleDelayedWorkImpl 调用 Mac 操作系统的函数 CFRunLoopTimerSetNextFireDate，因为 setTimeout 的第二个参数是 100，所以 CFRunLoopTimerSetNextFireDate 的作用是在 100 ms 后，唤醒当前线程，继续消息循环。以下摘自[苹果官方文档](https://developer.apple.com/documentation/corefoundation/1542501-cfrunlooptimersetnextfiredate)：

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
    base::debug::Alias(&delay_msec);
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

Android 操作系统调用的是 [MessagePumpForUI::ScheduleDelayedWork](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump_android.cc)，源码如下：

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


### 线程睡眠 100 ms

因为示例代码：

```JavaScript
setTimeout(_ => {
  console.log('test')
}, 100)
```

没有其它逻辑需要执行，消息循环可以睡眠 100ms，实际上由于操作系统定时器函数未必准确，实际的睡眠时间可能超过 100ms。

### 唤醒线程，继续消息循环

### 为什么 setTimeout 不准确

## clearTimeout



























