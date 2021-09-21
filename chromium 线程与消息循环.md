# Chromium 线程与消息循环

Chromium 源码版本 91.0.4437.3，大约发布于 2021 年春节。

## Chromium 线程

Chromium 与线程相关的类有两个，[Thread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.h#62) 类和 [PlatformThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread.h#121)。Thread 类是 Chromium 对线程的封装，PlatformThread 类封装了操作系统的线程相关函数，不同的操作系统有不同的实现。Thread 类基于 PlatformThread 类。我们从创建线程的源码开始。

### Thread

Chromium 中，开启一个线程的方法是 [Thread::Start](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.cc#131)

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
  if (stack_size > 0) {
    flags = STACK_SIZE_PARAM_IS_A_RESERVATION;
#if defined(ARCH_CPU_32_BITS)
  } else {
    // The process stack size is increased to give spaces to |RendererMain| in
    // |chrome/BUILD.gn|, but keep the default stack size of other threads to
    // 1MB for the address space pressure.
    flags = STACK_SIZE_PARAM_IS_A_RESERVATION;
    static BOOL is_wow64 = -1;
    if (is_wow64 == -1 && !IsWow64Process(GetCurrentProcess(), &is_wow64))
      is_wow64 = FALSE;
    // When is_wow64 is set that means we are running on 64-bit Windows and we
    // get 4 GiB of address space. In that situation we can afford to use 1 MiB
    // of address space for stacks. When running on 32-bit Windows we only get
    // 2 GiB of address space so we need to conserve. Typically stack usage on
    // these threads is only about 100 KiB.
    if (is_wow64)
      stack_size = 1024 * 1024;
    else
      stack_size = 512 * 1024;
#endif
  }

  ThreadParams* params = new ThreadParams;
  params->delegate = delegate;
  params->joinable = out_thread_handle != nullptr;
  params->priority = priority;

  // Using CreateThread here vs _beginthreadex makes thread creation a bit
  // faster and doesn't require the loader lock to be available.  Our code will
  // have to work running on CreateThread() threads anyway, since we run code on
  // the Windows thread pool, etc.  For some background on the difference:
  // http://www.microsoft.com/msj/1099/win32/win321099.aspx
  void* thread_handle =
      ::CreateThread(nullptr, stack_size, ThreadFunc, params, flags, nullptr);

  if (!thread_handle) {
    DWORD last_error = ::GetLastError();

    switch (last_error) {
      case ERROR_NOT_ENOUGH_MEMORY:
      case ERROR_OUTOFMEMORY:
      case ERROR_COMMITMENT_LIMIT:
        TerminateBecauseOutOfMemory(stack_size);
        break;

      default:
        static auto* last_error_crash_key = debug::AllocateCrashKeyString(
            "create_thread_last_error", debug::CrashKeySize::Size32);
        debug::SetCrashKeyString(last_error_crash_key,
                                 base::NumberToString(last_error));
        break;
    }

    delete params;
    return false;
  }

  if (out_thread_handle)
    *out_thread_handle = PlatformThreadHandle(thread_handle);
  else
    CloseHandle(thread_handle);
  return true;
}
```



这个问题很简单，其实不需要看源码，querySelectorAll 和 Array 属于不同的规范。

querySelectorAll 是 [Dom](https://dom.spec.whatwg.org/) 规范中的内容，Dom 规范是平台中立的，笔者在 Dom 规范中全局搜索 V8，没有结果。全局搜索 JavaScript，有 4 个结果。以下摘自规范：

> DOM defines a platform-neutral model for events, aborting activities, and node trees.


[Dom](https://dom.spec.whatwg.org/#parentnode) 规范中与 querySelectorAll 相关的内容如下：

![querySelectorIDL](https://raw.githubusercontent.com/xudale/blog/master/assets/querySelectorIDL.png)
![querySelectorDesc](https://raw.githubusercontent.com/xudale/blog/master/assets/querySelectorDesc.png)

从上面截图中可见，querySelectorAll 返回的结果是 NodeList。


Array 的定义来自 [ECMAScript specification](https://tc39.es/ecma262/#sec-array-constructor)。

![arrayecma](https://raw.githubusercontent.com/xudale/blog/master/assets/arrayecma.png)


既然 querySelectorAll 和 Array 不属于同一个规范，Dom 规范要平台中立，JavaScript 要运行在服务端，甚至嵌入式系统。querySelectorAll 返回的结果不是 Array 类型， 也算正常。


## chromium 消息循环

Chromium 源码版本 91.0.4437.3。因为 querySelectorAll 的返回结果和 JavaScript 数组都有 length 属性，本小节比较下两者的代码。

querySelectorAll 源码位于 [blink](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/)。[querySelectorAll](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/dom/parent_node.h#91) 方法源码如下：



```C++
static StaticElementList* querySelectorAll(ContainerNode& node,
                                          const AtomicString& selectors,
                                          ExceptionState& exception_state) {
  return node.QuerySelectorAll(selectors, exception_state);
}
```

第一个参数 node 表示当前 dom 节点，第二个参数 selectors 是一个选择器对象，基本是用一个 C++ 对象把选择器字符串，如 ".classname > div" 包了一层。从 blink 源码来看，querySelectorAll 的返回结果不是 JavaScript 数组，而是 StaticElementList，StaticElementList 的定义[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/dom/static_node_list.h#40)：


```C++
using StaticElementList = StaticNodeTypeList<Element>;

template <typename NodeType>
class StaticNodeTypeList final : public NodeList {
 public:
  static StaticNodeTypeList* Adopt(HeapVector<Member<NodeType>>& nodes);

  ~StaticNodeTypeList() override;

  unsigned length() const override;

 private:
  // nodes_ 保存所有选中结点
  HeapVector<Member<NodeType>> nodes_;
};
```

如果在浏览器中执行下面 JavaScript 代码：

```JavaScript
const divList = document.querySelectorAll('div')
divList.length
```

JavaScript 代码 divList.length，底层调用的是 StaticNodeTypeList<NodeType>::length，[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/dom/static_node_list.h#69)：

```C++
template <typename NodeType>
unsigned StaticNodeTypeList<NodeType>::length() const {
  return nodes_.size();
}
```

JavaScript 数组源码位于 V8，[声明如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.1-lkgr/src/objects/js-array.h#24)：

```C++
class JSArray : public JSObject {
 public:
  // [length]: The length property.
  DECL_ACCESSORS(length, Object)

  // Overload the length setter to skip write barrier when the length
  // is set to a smi. This matches the set function on FixedArray.
  inline void set_length(Smi length);

  static bool MayHaveReadOnlyLength(Map js_array_map);
  static bool HasReadOnlyLength(Handle<JSArray> array);
}

// length 表示方法名称
// kLengthOffset 表示 length 属性在数组对象上的偏移量
ACCESSORS(JSArray, length, Object, kLengthOffset)
```

ACCESSORS 为数组定义的 length 方法，顺着宏定义一路往下追，最终[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.1-lkgr/src/objects/object-macros.h#179)：

```C++
#define ACCESSORS_CHECKED2(holder, name, type, offset, get_condition, \
                           set_condition)                             \
  DEF_GETTER(holder, name, type) {                                    \
    type value = TaggedField<type, offset>::load(isolate, *this);     \
    DCHECK(get_condition);                                            \
    return value;                                                     \
  }                                                                   

#define ACCESSORS_CHECKED(holder, name, type, offset, condition) \
  ACCESSORS_CHECKED2(holder, name, type, offset, condition, condition)

#define ACCESSORS(holder, name, type, offset) \
  ACCESSORS_CHECKED(holder, name, type, offset, true)
```

可见获取数组 length 属性的逻辑是读取 JSArray 对象上，偏移量为 kLengthOffset 的内容，与 StaticNodeTypeList 的 length 逻辑完全不同。

> querySelectorAll 的源码位于 blink，JavaScript Array 的源码位于 V8

> blink 是 Chromium 渲染引擎，V8 是 JavaScript 引擎，二者是不同的项目




















