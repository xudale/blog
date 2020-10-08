# Promise 源码分析(一)

node 版本 14.13.0，V8 版本 8.4.371。Promise 源码全部在 V8，所以 node 版本不重要。很早就想写 Promise 源码相关的文章，但 V8 中 Promise 的源码以前都是用 CodeStubAssembler 写的，根据过往经验，大多数前端看不懂 CodeStubAssembler 这种汇编风格的语言。直到两个月前无意中发现 V8 的新版本已经用 Torque 重写了 Promise 的大部分，遂有此文。

## 基本数据结构

### 三种状态

Promise 共有 3 种状态，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/base.tq#190)：

```C++
// Promise constants
extern enum PromiseState extends int31 constexpr 'Promise::PromiseState' {
  kPending,
  kFulfilled,
  kRejected
}
```

一个新创建的 Promise 处于 pending 状态。当调用 resolve 或 reject 函数后，Promise 就会处于 fulfilled 或 rejected 状态，此后 Promise 的状态将不可改变，也就是说 Promise 的状态改变是不可逆的。

### JSPromise

JSPromise 描述 Promise 的基本信息，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/objects/js-promise.tq#13)：

```C++
bitfield struct JSPromiseFlags extends uint31 {
  status: PromiseState: 2 bit;
  has_handler: bool: 1 bit;
  handled_hint: bool: 1 bit;
  async_task_id: int32: 22 bit;
}

@generateCppClass
extern class JSPromise extends JSObject {
  macro Status(): PromiseState {
    // 获取 Promise 的状态 kPending/kFulfilled/kRejected 中的一个
    return this.flags.status;
  }

  macro SetStatus(status: constexpr PromiseState): void {
    // 
    assert(this.Status() == PromiseState::kPending);
    assert(status != PromiseState::kPending);

    this.flags.status = status;
  }

  macro HasHandler(): bool {
    return this.flags.has_handler;
  }

  macro SetHasHandler(): void {
    this.flags.has_handler = true;
  }

  // Smi 0 terminated list of PromiseReaction objects in case the JSPromise was
  // not settled yet, otherwise the result.
  reactions_or_result: Zero|PromiseReaction|JSAny;
  flags: SmiTagged<JSPromiseFlags>;
}
```

### 其它

## 构造函数

## then

## resolve

## 总结

```JavaScript
const os = require('os');
const cpus = os.cpus()
console.log(cpus)
// 输出
[
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 987260, nice: 0, sys: 859740, idle: 2834280, irq: 0 }
  },
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 604020, nice: 0, sys: 288860, idle: 3624470, irq: 0 }
  },
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 977040, nice: 0, sys: 584940, idle: 2955370, irq: 0 }
  },
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 600650, nice: 0, sys: 272840, idle: 3643860, irq: 0 }
  }
]
```

相关源码也很有层次性，从 JS -> C++ -> C -> Mac OS，下面逐层分析。

## JS
cpus 方法的源码位于 lib/os.js，粘贴如下：

```JavaScript
const {
  getCPUs
} = internalBinding('os');

function cpus() {
  // [] is a bugfix for a regression introduced in 51cea61
  const data = getCPUs() || [];
  const result = [];
  let i = 0;
  while (i < data.length) {
    result.push({
      model: data[i++],
      speed: data[i++],
      times: {
        user: data[i++],
        nice: data[i++],
        sys: data[i++],
        idle: data[i++],
        irq: data[i++]
      }
    });
  }
  return result;
}

module.exports = {
  cpus
};
```

cpus 通过调用 getCPUs 得到 data 数组，data 数组里面存储的是 CPU 相关的信息，data 数组的长度就是 CPU 的核心数。JS 层的 cpus 只是简单包装了一下 C++ 层的 getCPUs。
## C++
getCPUs 方法的源码位于 src/node_os.cc，粘贴如下：

```c++
static void GetCPUInfo(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  Isolate* isolate = env->isolate();

  uv_cpu_info_t* cpu_infos;
  int count;
  // GetCPUInfo 不生产 CPU 信息，
  // 本质还是调用 libuv 的 uv_cpu_info 获取 CPU 信息
  int err = uv_cpu_info(&cpu_infos, &count);
  if (err)
    return;

  // 代码运行到这里 cpu_infos 已经保存了 CPU 的信息，count 是核心数
  // std::vector 类似 JavaScript 的数组
  std::vector<Local<Value>> result(count * 7);
  for (int i = 0, j = 0; i < count; i++) {
    uv_cpu_info_t* ci = cpu_infos + i;
    result[j++] = OneByteString(isolate, ci->model);
    // Number::New 在 C++ 中创建一个 JavaScript 的 Number 对象
    result[j++] = Number::New(isolate, ci->speed);
    result[j++] = Number::New(isolate, ci->cpu_times.user);
    result[j++] = Number::New(isolate, ci->cpu_times.nice);
    result[j++] = Number::New(isolate, ci->cpu_times.sys);
    result[j++] = Number::New(isolate, ci->cpu_times.idle);
    result[j++] = Number::New(isolate, ci->cpu_times.irq);
  }

  uv_free_cpu_info(cpu_infos, count);
  // Array::New 在 C++ 中创建一个 JavaScript 的 Array 对象
  // Array::New(isolate, result.data(), result.size()) 
  // 的结果就是 JS 层 cpus 的 data 变量
  // const data = getCPUs()
  args.GetReturnValue().Set(Array::New(isolate, result.data(), result.size()));
}

void Initialize(Local<Object> target,
                Local<Value> unused,
                Local<Context> context,
                void* priv) {
  Environment* env = Environment::GetCurrent(context);
  env->SetMethod(target, "getCPUs", GetCPUInfo);
}
```

JS 层的 getCPUs 方法，实际是 C++ 层的 GetCPUInfo 函数，GetCPUInfo 代码不多，仅仅是对 libuv 中 uv_cpu_info 的包装。由于 GetCPUInfo 要把返回结果传给 JS 层，所以要把 C++ 的整数转换成 JavaScript 的 Number:

```c++
result[j++] = Number::New(isolate, ci->speed);
```

还要把 C++ 的 std::vector 转换成 JavaScript 的 Array:

```c++
std::vector<Local<Value>> result(count * 7);
Array::New(isolate, result.data(), result.size());
```

在 V8.h 中，Array::New 声明如下：

```c++
/**
 * An instance of the built-in array constructor (ECMA-262, 15.4.2).
 */
class V8_EXPORT Array : public Object {
 public:
  uint32_t Length() const;
  /**
   * Creates a JavaScript array out of a Local<Value> array in C++
   * with a known length.
   */
  static Local<Array> New(Isolate* isolate, Local<Value>* elements,
                          size_t length);
};
```

JavaScript 与 C++ 函数的互相调用，C++ 层面的工作量要大一些。因为 C++ 是 JavaScript 的底层，只要研究足够深入，在 C++ 层面可以透澈看到 JavaScript 对象的一切，比如地址和内存布局，反之则不行。JavaScript 调用 C++ 的方法，需要在 C++ 层面把返回值转成 JavaScript 语言的类型，Number::New 和 Array::New 做的就是这样的类型转换工作。

C++ 层的 GetCPUInfo 函数工作量并不多，只是包装了一下 libuv 的 uv_cpu_info，还要继续往下看。

## C

进入了传说中的 libuv，笔者的电脑是 Mac，在 Mac 下 uv_cpu_info 真正运行的源码位于 deps/uv/src/unix/darwin.c，这部分代码是 C 语言，不是 C++，粘贴如下：

```c
int uv_cpu_info(uv_cpu_info_t** cpu_infos, int* count) {
  unsigned int ticks = (unsigned int)sysconf(_SC_CLK_TCK),
               multiplier = ((uint64_t)1000L / ticks);
  char model[512]; // brand 字符串，本机是 Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz
  uint64_t cpuspeed; // cpu 频率
  size_t size;
  unsigned int i;
  natural_t numcpus; // CPU 核心数，natural_t 是整数类型
  mach_msg_type_number_t msg_type; // mach_msg_type_number_t 也是整数类型
  processor_cpu_load_info_data_t *info;
  uv_cpu_info_t* cpu_info; // 无缝衔接上节的 cpu_info

  size = sizeof(model);
  // 获取 CPU brand_string 到 model，本机是 Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz
  if (sysctlbyname("machdep.cpu.brand_string", &model, &size, NULL, 0) &&
      sysctlbyname("hw.model", &model, &size, NULL, 0)) {
    return UV__ERR(errno);
  }

  size = sizeof(cpuspeed);
  //  获取 CPU 频率到 cpuspeed，本机是 2700000000
  if (sysctlbyname("hw.cpufrequency", &cpuspeed, &size, NULL, 0))
    return UV__ERR(errno);
  // 获取 CPU 时间相关的信息到 info，numcpus 表示核心数
  if (host_processor_info(mach_host_self(), PROCESSOR_CPU_LOAD_INFO, &numcpus,
                          (processor_info_array_t*)&info,
                          &msg_type) != KERN_SUCCESS) {
    return UV_EINVAL;  /* FIXME(bnoordhuis) Translate error. */
  }

  *cpu_infos = uv__malloc(numcpus * sizeof(**cpu_infos));
  if (!(*cpu_infos)) {
    vm_deallocate(mach_task_self(), (vm_address_t)info, msg_type);
    return UV_ENOMEM;
  }

  *count = numcpus;
  // 将 info 中的信息赋值给 cpu_info 相应字段
  for (i = 0; i < numcpus; i++) {
    cpu_info = &(*cpu_infos)[i];

    cpu_info->cpu_times.user = (uint64_t)(info[i].cpu_ticks[0]) * multiplier;
    cpu_info->cpu_times.nice = (uint64_t)(info[i].cpu_ticks[3]) * multiplier;
    cpu_info->cpu_times.sys = (uint64_t)(info[i].cpu_ticks[1]) * multiplier;
    cpu_info->cpu_times.idle = (uint64_t)(info[i].cpu_ticks[2]) * multiplier;
    cpu_info->cpu_times.irq = 0;

    cpu_info->model = uv__strdup(model);
    cpu_info->speed = cpuspeed/1000000;
  }
  vm_deallocate(mach_task_self(), (vm_address_t)info, msg_type);

  return 0;
}
```

uv_cpu_info 源码较长，还有我们之前没有见过的函数和类型，它们都来自 Mac OS。对好学的前端来说，这些 API 都不具备深入学习的价值，所以笔者抽取相关代码整理了一个 demo，可直接在 Xcode 中运行，请自行体会。

```c
#include <mach/mach.h>
#include <sys/sysctl.h>
#include <stdio.h>

int uv_cpu_info() {
    char model[512];
    uint64_t cpuspeed;
    size_t size;
    natural_t numcpus;
    mach_msg_type_number_t msg_type;
    processor_cpu_load_info_data_t *info;
    
    size = sizeof(model);
    sysctlbyname("machdep.cpu.brand_string", &model, &size, NULL, 0) &&
    sysctlbyname("hw.model", &model, &size, NULL, 0);
    
    printf("model is %s\n",model);
    
    size = sizeof(cpuspeed);
    sysctlbyname("hw.cpufrequency", &cpuspeed, &size, NULL, 0);
    printf("cpuspeed is %lld\n",cpuspeed);
    host_processor_info(mach_host_self(), PROCESSOR_CPU_LOAD_INFO, &numcpus,
                            (processor_info_array_t*)&info,
                        &msg_type);
    printf("numcpus is %u\n",numcpus);
    
    return 0;
}

int main(int argc, const char * argv[]) {
    uv_cpu_info();
    return 0;
}
```

在 Xcode 中运行结果如下：

![mach_host_self](https://raw.githubusercontent.com/xudale/blog/master/assets/mach_host_self.png)

## Mac OS

Mac 内核已开源，或者部分开源，地址 [darwin-xnu](https://github.com/apple/darwin-xnu)，但笔者下载后 make 就报错。

笔者只看了 sysctlbyname("machdep.cpu.brand_string", &model, &size, NULL, 0) 相关的源码。大概是 machdep.cpu.brand_string 会被解析，从树上找到相应的节点，从节点里找到一个函数，调用函数就得到了 cpu brand_string：Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz。笔者是内核的小白，不太确定。但有一点可以确定，Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz 这个字符串是通过 cpuid 指令获取的。

```c++
从 https://github.com/apple/darwin-xnu/blob/master/bsd/dev/arm64/sysctl.c#L146 复制的
/*
 * machdep.cpu.brand_string
 *
 * x86: derived from CPUID data.
 * ARM: cons something up from the CPUID register. Could include cpufamily
 *  here and map it to a "marketing" name, but there's no obvious need;
 *      the value is already exported via the commpage. So keep it simple.
 */
```


## cpuid

cpuid 是 x86 处理器的一条指令，可以获取 CPU 的信息，比如核心数、是否支持某些指令集、厂商等。V8 有调用 cpuid 来判断当前 CPU 是否支持某些特定指令，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/base/cpu.cc#358)：

```c++
__cpuid(cpu_info, 1); // __cpuid 是 V8 对 cpuid 指令的封装

has_fpu_ = (cpu_info[3] & 0x00000001) != 0;
has_cmov_ = (cpu_info[3] & 0x00008000) != 0;
has_mmx_ = (cpu_info[3] & 0x00800000) != 0;
has_sse_ = (cpu_info[3] & 0x02000000) != 0;
has_sse2_ = (cpu_info[3] & 0x04000000) != 0;
has_sse3_ = (cpu_info[2] & 0x00000001) != 0;
has_ssse3_ = (cpu_info[2] & 0x00000200) != 0;
has_sse41_ = (cpu_info[2] & 0x00080000) != 0;
has_sse42_ = (cpu_info[2] & 0x00100000) != 0;
has_popcnt_ = (cpu_info[2] & 0x00800000) != 0;
has_osxsave_ = (cpu_info[2] & 0x08000000) != 0;
has_avx_ = (cpu_info[2] & 0x10000000) != 0;
has_fma3_ = (cpu_info[2] & 0x00001000) != 0;
```

比如 has_fpu_ 表示是否有浮点数运算处理器，浮点数运算不精确和它有关系，has_sse 开头的几个变量和 SIMD 有关，JavaScript 的 SIMD 归根到底还是要编译成 CPU 的 SIMD 指令，纯靠软件做不到单条指令多个数据(Single Instruction Multiple Data)。软件是串行执行的，优化到极限也改变不了串行执行的特点，想要并行，要靠硬件。

高级语言的赋值编译成汇编语言的 mov，高级语言的分支编译成汇编语言的条件跳转，似乎高级语言并没有和 cpuid 相对应的语法。V8 的 __cpuid 函数使用内联汇编封装了 cpuid 指令，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/base/cpu.cc#55)：

```c++
static V8_INLINE void __cpuid(int cpu_info[4], int info_type) {
// Clear ecx to align with __cpuid() of MSVC:
// https://msdn.microsoft.com/en-us/library/hskdteyh.aspx
// 宏定义搭配内联汇编，不要在意细节，把内联汇编当普通汇编看就好
#if defined(__i386__) && defined(__pic__)
  // Make sure to preserve ebx, which contains the pointer
  // to the GOT in case we're generating PIC.
  __asm__ volatile(
      "mov %%ebx, %%edi\n\t"
      "cpuid\n\t"
      "xchg %%edi, %%ebx\n\t"
      : "=a"(cpu_info[0]), "=D"(cpu_info[1]), "=c"(cpu_info[2]),
        "=d"(cpu_info[3])
      : "a"(info_type), "c"(0));
#else
  __asm__ volatile("cpuid \n\t"
                   : "=a"(cpu_info[0]), "=b"(cpu_info[1]), "=c"(cpu_info[2]),
                     "=d"(cpu_info[3])
                   : "a"(info_type), "c"(0));
#endif  // defined(__i386__) && defined(__pic__)
}
```

大多数前端看到上面的代码应该是比较吃力的。笔者对 V8 中 cpuid 相关的代码稍作整理，并参考 Intel 软件手册，写了一个例子，获取 CPU 的 brand string，可直接在 Xcode 中运行。

![brandString](https://raw.githubusercontent.com/xudale/blog/master/assets/brandString.png)

```c++
#include <stdio.h>

void __cpuid(int cpu_info[4], int info_type) {
    __asm__ volatile("cpuid \n\t"
                     : "=a"(cpu_info[0]), "=b"(cpu_info[1]), "=c"(cpu_info[2]),
                     "=d"(cpu_info[3])
                     : "a"(info_type), "c"(0));
}

int main(int argc, const char * argv[]) {
    int brand[12];
    __cpuid(&brand[0], 0x80000002);
    __cpuid(&brand[4], 0x80000003);
    __cpuid(&brand[8], 0x80000004);
    printf("Brand: %s\n", (char *)brand); // brand 前端都是空格，为了简洁，就不过滤了
    return 0;
}
```

在 Xcode 中运行结果如下：

![brandExample](https://raw.githubusercontent.com/xudale/blog/master/assets/brandExample.png)

## 总结

![cpuidbrand](https://raw.githubusercontent.com/xudale/blog/master/assets/cpuidbrand.png)

## 参考文献

[从os.cpus()来分析nodejs源码结构](https://cnodejs.org/topic/569dc6e8e6816bdc6dab52fb)

[深入 Nodejs 源码探究 CPU 信息的获取与利用率计算](https://zhuanlan.zhihu.com/p/126334155)

因为第 2 篇文章写的很好，本文在相似内容上没有多费笔墨，感兴趣的同学可[移驾](https://zhuanlan.zhihu.com/p/126334155)






















