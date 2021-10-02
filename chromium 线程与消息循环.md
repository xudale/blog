# Chromium 线程与消息循环

Chromium 源码版本 91.0.4437.3，大约发布于 2021 年春节。

## Chromium 线程

Chromium 与线程相关的类有两个，[Thread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.h#62) 和 [PlatformThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread.h#121)。Thread 底层调用 PlatformThread 创建线程。PlatformThread 封装了操作系统级别的线程相关 API，在不同的操作系统有不同的实现，但接口保持一致。本文从创建线程的源码开始。

### Thread

在 Chromium 中，开启一个线程的方法是 [Thread::Start](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#131)，源码如下：

```C++
bool Thread::Start() {
  Options options;
  return StartWithOptions(options);
}
```

options 对象表示创建线程需要的一些参数，如线程优先级、栈大小和消息循环等，[声明如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.h#76)：

```C++
struct BASE_EXPORT Options {
  using MessagePumpFactory =
      RepeatingCallback<std::unique_ptr<MessagePump>()>;

  Options();
  Options(MessagePumpType type, size_t size);
  Options(Options&& other);
  ~Options();

  // 消息循环的类型，Chromium Thread 类自带消息循环
  MessagePumpType message_pump_type = MessagePumpType::DEFAULT;

  // Specifies timer slack for thread message loop.
  TimerSlack timer_slack = TIMER_SLACK_NONE;

  // 线程栈大小
  size_t stack_size = 0;

  // 线程优先级
  ThreadPriority priority = ThreadPriority::NORMAL;

  // If false, the thread will not be joined on destruction.
  bool joinable = true;
};
```

Thread::Start 调用了 Thread::StartWithOptions，[Thread::StartWithOptions](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#142) 源码如下：

```C++
bool Thread::StartWithOptions(const Options& options) {
  timer_slack_ = options.timer_slack;  
  // 重点在 MessagePump::Create(type) 创建了消息循环
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



Thread::StartWithOptions 做的还是一线程初始化的工作，多数代码逻辑都与描述线程参数的 options 对象有关，并没有真正创建线程。真正创建操作系统线程在 PlatformThread::CreateWithPriority。




### PlatformThread

PlatformThread，平台线程，从名字也以看出不同的操作系统有不同的实现。在不同的操作系统下，[PlatformThread 声明](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread.h#121)部分是一致的，源码如下：

```C++
// A namespace for low-level thread functions.
class BASE_EXPORT PlatformThread {
 public:
  static PlatformThreadHandle CurrentHandle();

  // 让步
  static void YieldCurrentThread();

  // 睡眠
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

PlatformThread 类的特点是方法都是静态方法，代码逻辑基本是对操作系统的线程函数包装了一下，不同操作系统有不同的实现。

在 Mac 下，[CreateWithPriority](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread_posix.cc#249) 最终是通过调用操作系统的 pthread_create 函数创建了线程，源码如下：

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
  // 核心代码就这一行，调用操作系统 pthread_create 创建线程
  int err = pthread_create(&handle, &attributes, ThreadFunc, params.get());
  bool success = !err;
  return success;
}
```

CreateThread 的核心代码就是调用操作系统的 [pthread_create](https://baike.baidu.com/item/pthread_create/5139072?fr=aladdin) 创建线程。pthread_create 的 4 个参数，handle 表示指向线程标识符的指针，attributes 表示线程属性，ThreadFunc 表示线程运行函数的起始地址。

在 Windows 平台下，[PlatformThread::CreateWithPriority](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread_win.cc#289) 归根到底还是调用了 Windows 操作系统的 [CreateThread](https://docs.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) 函数创建了线程，源码如下：

```C++
// static
bool PlatformThread::CreateWithPriority(size_t stack_size, Delegate* delegate,
                                        PlatformThreadHandle* thread_handle,
                                        ThreadPriority priority) {
  DCHECK(thread_handle);
  return CreateThreadInternal(stack_size, delegate, thread_handle, priority);
}

bool CreateThreadInternal(size_t stack_size,
                          PlatformThread::Delegate* delegate,
                          PlatformThreadHandle* out_thread_handle,
                          ThreadPriority priority) {
  unsigned int flags = 0;
  ThreadParams* params = new ThreadParams;
  params->delegate = delegate;
  params->joinable = out_thread_handle != nullptr;
  params->priority = priority;
  // 核心代码就这一行，调用操作系统 CreateThread 创建线程
  void* thread_handle =
      ::CreateThread(nullptr, stack_size, ThreadFunc, params, flags, nullptr);
  if (out_thread_handle)
    *out_thread_handle = PlatformThreadHandle(thread_handle);
  else
    CloseHandle(thread_handle);
  return true;
}
```

> PlatformThread 的声明是公共的

> PlatformThread 不同操作系统下有不同的实现，实现的跨平台的线程




## chromium 消息循环

从前方的 [Thread::StartWithOptions](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#142) 方法中，可以看到 Thread 类是自带消息循环的。

```C++
bool Thread::StartWithOptions(const Options& options) {
  timer_slack_ = options.timer_slack;  
  // 重点在 MessagePump::Create(type) 创建了消息循环
  delegate_ = std::make_unique<SequenceManagerThreadDelegate>(
      options.message_pump_type,
      BindOnce([](MessagePumpType type) { return MessagePump::Create(type); },
               options.message_pump_type),
      options.task_queue_time_domain);
  // 后面略
}
```

[MessagePump::Create](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump.cc#44) 方法创建了一个消息循环，代码如下：


```C++
// static
std::unique_ptr<MessagePump> MessagePump::Create(MessagePumpType type) {
  switch (type) {
    case MessagePumpType::UI:
      return message_pump_for_ui_factory_();
    // 这个分支
    case MessagePumpType::DEFAULT:
      return std::make_unique<MessagePumpDefault>();
  }
}
```

MessagePump::Create 创建消息循环后，消息循环开始运行，[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/message_loop/message_pump_default.cc#31)：

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

被 for (;;) 包裹的代码块就是前端所说的消息循环，在消息循环中处理当前任务。消息循环是一个死循环，但消息循环所处的线程大多数时间是休眠状态。笔者实测，只有发生了鼠标点击/滚动/键盘输入等事件时，消息循环中的代码才会执行。


## 总结

Chromium 源码中的 Thread 类是一个跨平台，自带消息循环的类。

Chromium 消息循环是一个死循环，但消息循环所处的线程多数情况下并不处于运行状态。
























