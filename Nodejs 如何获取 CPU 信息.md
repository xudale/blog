# Nodejs 如何获取 CPU 信息

基于 Mac，环境如下：
| 环境       |    版本 |
| --------  | -------- |
| Nodejs    | 12.16.1 |
| V8        | 7.8.279.23-node.31 |
| MacOS      | 10.13.5 |
| MacBook Pro      | 13-inch, Early 2011 |
| CPU      | 2.7 GHz Intel Core i7 |
| Xcode      | 9.4.1 |

Nodejs 获取 CPU 信息十分简单，加载 os 模块后，调用其 cpus 方法就可获取 CPU 信息，代码如下：

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

相关源码也很有层次性，从 JS -> C++ -> C -> 操作系统，下面逐层分析。

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

cpus 通过调用 getCPUs 得到 data 数组，里面存储的是 CPU 相关的信息，data 数组的长度就是 CPU 的核心数。JS 层只是简单包装了一下 C++ 层的 getCPUs。
## C++
getCPUs 方法的源码位于 src/node_os.cc，粘贴如下：

```c++
static void GetCPUInfo(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);
  Isolate* isolate = env->isolate();

  uv_cpu_info_t* cpu_infos;
  int count;
  // 本质还是调用 libuv 的 uv_cpu_info 获取 CPU 信息
  int err = uv_cpu_info(&cpu_infos, &count);
  if (err)
    return;

  // 代码运行到这里 cpu_infos 已经保存了 CPU 的信息，count 是核心数
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
  // Array::New(isolate, result.data(), result.size()) 就是 JS 层的 data
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

JS 层的 getCPUs 方法，实际调用的是 C++ 层的 GetCPUInfo 函数，GetCPUInfo 代码不多，仅仅是对 libuv 中 uv_cpu_info 的包装。由于 GetCPUInfo 要把返回结果传给 JS 层，所以要把相应的 C++ 语言的整数转换成 JavaScript 的 Number:

```c++
result[j++] = Number::New(isolate, ci->speed);
```

还要把 C++ 语言的 std::vector 转换成 JavaScript 的 Array:

```c++
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
  V8_INLINE static Array* Cast(Value* obj);
};
```

JavaScript 与 C++ 函数的互相调用，C++ 的工作量要大一些。因为 C++ 是 JavaScript 的底层，只要研究足够深入，在 C++ 层面可以透澈看到 JavaScript 对象的一切，反之则不行。 JavaScript 调用 C++ 的方法，需要在 C++ 层面把返回值转成 JavaScript 语言的类型，Number::New 和 Array::New 做的就是这样的转换工作。

C++ 层的 GetCPUInfo 函数工作量并不多，只是包装了一下 libuv 的 uv_cpu_info。

## C

进入了传说中的 libuv，笔者的电脑是 Mac，uv_cpu_info 真正运行的源码位于 deps/uv/src/unix/darwin.c，这部分代码是 C 语言，不是 C++，粘贴如下：

```c
int uv_cpu_info(uv_cpu_info_t** cpu_infos, int* count) {
  unsigned int ticks = (unsigned int)sysconf(_SC_CLK_TCK),
               multiplier = ((uint64_t)1000L / ticks);
  char model[512];
  uint64_t cpuspeed;
  size_t size;
  unsigned int i;
  natural_t numcpus; // natural_t 是整数类型
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

uv_cpu_info 源码较长，还有我们之前没有见过的函数和类型，它们都来自 Mac OS。对好学的前端来说，这些 API 都不具备深入学习的价值，所以笔者抽取相关代码整理了一个 demo，可直接在 Xcode 中运行，可自行体会。

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

笔者不确定与本文相关的代码是否开源，本小节存在的目的是为了保持本文 JS -> C++ -> C -> 操作系统的层次结构。

## Assembly

在 Intel 平台上获取 CPU 的信息，除了 cpuid，似乎没有别的办法，cpuid 是 x86 处理器的一条指令，可以获取 CPU 的信息，V8 有调用 cpuid 来判断当前 CPU 是否支持某些特定指令，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/base/cpu.cc#358)：

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

比如 has_fpu_ 表示是否有浮点数运算处理器，浮点数运算不精确和它有关系，has_sse 开头的几个变量和 SIMD 有关，JavaScript 的 SIMD 归根到底还是要编译成 CPU 的 SIMD 指令，纯靠软件是做不到单条指令多个数据的(Single Instruction Multiple Data)。

高级语言的赋值编译成汇编语言的 mov，高级语言的分支编译成汇编语言的条件跳转，似乎高级语言并没有和 cpuid 相对应的语法。V8 的 __cpuid 函数使用内联汇编封装了下 cpuid 指令，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/base/cpu.cc#55)：

```c++
static V8_INLINE void __cpuid(int cpu_info[4], int info_type) {
// Clear ecx to align with __cpuid() of MSVC:
// https://msdn.microsoft.com/en-us/library/hskdteyh.aspx
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

笔者对 V8 中 cpuid 相关的代码稍作整理，并参考 Intel 软件手册，写了一个例子，获取 CPU 的 brand string，可直接在 Xcode 中运行。

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



















