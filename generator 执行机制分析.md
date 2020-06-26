# generator 执行机制分析
本文以下面代码为例

```JavaScript
function* test() {
  yield 123456
}
let iterator = test()
iterator.next()
```

分析 generator 执行机制相关的源码，版本为 V8 7.7.1。
## let iterator = test()——第一次暂停

首先，注释掉 iterator.next() 后，在 V8 中运行代码，

```JavaScript
function* test() {
  yield 123456
}
let iterator = test()
// iterator.next()
```

test 函数生成的字节码如下：

![generator-bytecode](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-bytecode.png)

test 有 * 修饰，是一个生成器函数。当 let iterator = test() 开始执行时，V8 会创建一个生成器对象，对应上图字节码中的 CreateJSGeneratorObject，CreateJSGeneratorObject [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/runtime/runtime-generator.cc#46)：

```c++
RUNTIME_FUNCTION(Runtime_CreateJSGeneratorObject) {
  // 有删减
  Handle<JSGeneratorObject> generator =
      isolate->factory()->NewJSGeneratorObject(function);
  generator->set_function(*function);
  generator->set_context(isolate->context());
  generator->set_receiver(*receiver);
  generator->set_parameters_and_registers(*parameters_and_registers);
  generator->set_continuation(JSGeneratorObject::kGeneratorExecuting);
  if (generator->IsJSAsyncGeneratorObject()) {
    Handle<JSAsyncGeneratorObject>::cast(generator)->set_is_awaiting(0);
  }
  return *generator;
}
```

Runtime_CreateJSGeneratorObject 的逻辑是创建一个新的生成器对象 V8 中的 JSGeneratorObject，设置相关属性后，最后返回生成器对象 generator。些时生成器对象 generator 在累加器中，

我们忽略 Star r0 和 StackCheck，来看字节码 SuspendGenerator 的处理函数，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/runtime/runtime-generator.cc#46)：

```c++
// Stores the parameters and the register file in the generator. Also stores
// the current context, |suspend_id|, and the current bytecode offset
// (for debugging purposes) into the generator. Then, returns the value
// in the accumulator.
IGNITION_HANDLER(SuspendGenerator, InterpreterAssembler) {
  Node* generator = LoadRegisterAtOperandIndex(0);
  TNode<FixedArray> array = CAST(LoadObjectField(
      generator, JSGeneratorObject::kParametersAndRegistersOffset));
  Node* closure = LoadRegister(Register::function_closure());
  Node* context = GetContext();
  RegListNodePair registers = GetRegisterListAtOperandIndex(1);
  Node* suspend_id = BytecodeOperandUImmSmi(3);

  Node* shared =
      LoadObjectField(closure, JSFunction::kSharedFunctionInfoOffset);
  TNode<Int32T> formal_parameter_count = UncheckedCast<Int32T>(
      LoadObjectField(shared, SharedFunctionInfo::kFormalParameterCountOffset,
                      MachineType::Uint16()));

  ExportParametersAndRegisterFile(array, registers, formal_parameter_count);
  StoreObjectField(generator, JSGeneratorObject::kContextOffset, context);
  StoreObjectField(generator, JSGeneratorObject::kContinuationOffset,
                   suspend_id);

  // Store the bytecode offset in the [input_or_debug_pos] field, to be used by
  // the inspector.
  Node* offset = SmiTag(BytecodeOffset());
  StoreObjectField(generator, JSGeneratorObject::kInputOrDebugPosOffset,
                   offset);

  UpdateInterruptBudgetOnReturn();
  Return(GetAccumulator());
}
```

字节码 SuspendGenerator 的功能是暂停执行，其处理函数里面多次调用 StoreObjectField 来保存生成器函数当前运行的状态，最后返回累加器中的值，之前提到过，此时累加器存的是生成器对象 generator。所以 V8 中的生成器对象 generator 返回给了 JavaScript 代码中的 iterator。此时，生成器函数处理暂停状态，字节码执行到了本文第一张图所标识的“第一次暂停”的位置。

> JavaScript 调用生成器函数（test）时，生成器函数开始执行，返回生成器对象（iterator），最后暂停
>
> 第一次调用生成器函数时，生成器函数的整体表现类似于构造函数。拿到生成器函数返回的生成器对象，可能让生成器函数继续执行

## iterator.next()——第二次暂停

上一节我们注释掉了 iterator.next()，当 JavaScript 代码中继续执行 iterator.next() 时，生成器对象的 next 方法被调用，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/builtins-generator-gen.cc#286)：

```c++
void GeneratorBuiltinsAssembler::GeneratorPrototypeResume(
    CodeStubArguments* args, Node* receiver, Node* value, Node* context,
    JSGeneratorObject::ResumeMode resume_mode, char const* const method_name) {
  // CodeFactory::ResumeGenerator 使 test 生成器函数从暂停处恢复执行
  Node* result = CallStub(CodeFactory::ResumeGenerator(isolate()), context,
                          value, receiver);
  // Make sure we close the generator if there was an exception.
  GotoIfException(result, &if_exception, &var_exception);
  TNode<Smi> result_continuation =
      CAST(LoadObjectField(receiver, JSGeneratorObject::kContinuationOffset));
  args->PopAndReturn(result);
}
```

由于笔者的 Mac 是 X64 平台，调用链路为[CodeFactory::ResumeGenerator](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/codegen/code-factory.cc#278)->[Builtins::Generate_ResumeGeneratorTrampoline](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/x64/builtins-x64.cc#695)。生成器函数 test 在上一节已经处于暂停状态，调用 iterator.next() 使其从暂停处继续执行，高级语言当然没有这样的功能。V8 在这里对各大平台的汇编做了一层抽象，Builtins::Generate_ResumeGeneratorTrampoline 函数通过调用 X64 汇编，使生成器函数在暂停处继续执行。Builtins::Generate_ResumeGeneratorTrampoline 代码很长，笔者只能看懂大概，这里只截取了几行，大致逻辑是将未来要返回的地址压栈，然后跳转到生成器函数 test 暂停的地方，继续执行。

```c++
#define __ ACCESS_MASM(masm)
void Builtins::Generate_ResumeGeneratorTrampoline(MacroAssembler* masm) {
  // Resume (Ignition/TurboFan) generator object.
  {
    __ PushReturnAddressFrom(rax);
    __ LoadTaggedPointerField(
        rax, FieldOperand(rdi, JSFunction::kSharedFunctionInfoOffset));
    __ movzxwq(rax, FieldOperand(
                        rax, SharedFunctionInfo::kFormalParameterCountOffset));
    // We abuse new.target both to indicate that this is a resume call and to
    // pass in the generator object.  In ordinary calls, new.target is always
    // undefined because generator functions are non-constructable.
    static_assert(kJavaScriptCallCodeStartRegister == rcx, "ABI mismatch");
    __ LoadTaggedPointerField(rcx, FieldOperand(rdi, JSFunction::kCodeOffset));
    __ JumpCodeObject(rcx);
  }
}
```

既然现在生成器函数已经从暂停处继续执行，那么下一条字节码 ResumeGenerator 的处理函数被调用，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-generator.cc#3243)：

```c++
// ResumeGenerator <generator> <first output register> <register count>
//
// Imports the register file stored in the generator and marks the generator
// state as executing.
IGNITION_HANDLER(ResumeGenerator, InterpreterAssembler) {
  CodeStubAssembler::Print("InterpreterAssembler::ResumeGenerator");
  Node* generator = LoadRegisterAtOperandIndex(0);
  Node* closure = LoadRegister(Register::function_closure());
  RegListNodePair registers = GetRegisterListAtOperandIndex(1);

  Node* shared =
      LoadObjectField(closure, JSFunction::kSharedFunctionInfoOffset);
  TNode<Int32T> formal_parameter_count = UncheckedCast<Int32T>(
      LoadObjectField(shared, SharedFunctionInfo::kFormalParameterCountOffset,
                      MachineType::Uint16()));

  ImportRegisterFile(
      CAST(LoadObjectField(generator,
                           JSGeneratorObject::kParametersAndRegistersOffset)),
      registers, formal_parameter_count);

  // Return the generator's input_or_debug_pos in the accumulator.
  // 恢复累加器
  SetAccumulator(
      LoadObjectField(generator, JSGeneratorObject::kInputOrDebugPosOffset));
  // 执行下一条字节码
  Dispatch();
}
```

ResumeGenerator 字节码的处理函数与 SuspendGenerator 字节码的处理函数基本相反，SuspendGenerator 保存当前生成器函数的各种执行状态，ResumeGenerator 恢复之前保存的状态，最后调用 Dispatch 函数，取出下一条字节码，执行。Dispatch 函数[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-assembler.cc#1396)：

```c++
Node* InterpreterAssembler::Dispatch() {
  Comment("========= Dispatch");
  DCHECK_IMPLIES(Bytecodes::MakesCallAlongCriticalPath(bytecode_), made_call_);
  Node* target_offset = Advance();
  Node* target_bytecode = LoadBytecode(target_offset);

  if (Bytecodes::IsStarLookahead(bytecode_, operand_scale_)) {
    target_bytecode = StarDispatchLookahead(target_bytecode);
  }
  return DispatchToBytecode(target_bytecode, BytecodeOffset());
}

Node* InterpreterAssembler::Advance() { return Advance(CurrentBytecodeSize()); }
```

如果把 V8 看做一个虚拟机，执行一条字节码后，必须要取出下一条字节码并执行，虚拟机上的程序才可能一直跑下去，ResumeGenerator 就是这样的，在其字节码处理函数的最后，调用 Dispatch，使程序继续执行，不会停止。而 SuspendGenerator 的字节码处理函数最后没有调用 Dispatch，所以从 JavaScript 的层面看来，生成器函数 test 暂停了。

字节码一行一行往下执行，一直遇到下一个 SuspendGenerator，也就是本文第一张图所标示的“第二次暂停”，生成器函数又暂停了，此时 yield 123456 执行完毕。

在 V8 层面 yield 对应的字节码是 SuspendGenerator 和 ResumeGenerator，第二次暂停是 yield 带来的。



## async/await 与 generator 的关系
笔者在另外一篇文章分析过 async/await。async/await 和 generator 都有暂停当前函数执行，从暂停处恢复执行的能力，await 和 yield 对应的字节码都是 SuspendGenerator 和 ResumeGenerator，这方面它们没有区别。

生成器函数需要生成器对象的 next 方法，才可以从暂停处恢复执行。async 函数依赖 Promise，与 microtask 联系紧密，V8 在执行 microtask 队列时，就可以让已经暂停的 async 函数恢复执行。正因为 async 函数集成了 Promise 和 microtask，使得其看起来更同步，写起来逻辑更清晰。

![generator-async](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-async.png)

## 3 种函数

JavaScript 的类型系统比较薄弱，比如 

```JavaScript
typeof 1 // number
typeof 0.1 // number
```

虽然 1 和 0.1 都是 number，但它们本质上是不同的类型，内存表示不一样，CPU 对整数和浮点数的运算指令也不一样。

```JavaScript
function* test() {
  yield 123456
}
typeof test // function

async function test1() {
  let res = await 123456;
}
typeof test1 // function

function test2() {}
typeof test2 // function
```

test、test1、test2 在 JavaScript 中的类型都是 function，但在 V8 中它们 3 个是不同的类型，如下图：

![generator-class](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-class.png)

日常开发中，当一个方法需要一个函数做为参数时，比如 forEach 和 map，多半需要的是 ES6 之前的函数，如果误传了 async 函数或者生成器函数，多半会出问题。因为 ES6 之前的函数、async 函数和生成器函数，虽然在 JavaScript 中 typeof 都返回 function，但在 V8 中它们是不同的类型，运行机制和返回值也不一样。
## 原生 generator 与 babel 转译 generator 的区别















