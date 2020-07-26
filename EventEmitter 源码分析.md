# EventEmitter 源码分析
EventEmitter 源码核心逻辑很简单，本文主要谈感想，版本为 nodejs 12.16.1，process.versions.v8 打印为 7.8.279.23-node.31。

## 核心逻辑
核心逻辑如下图，每个 EventEmitter 的实例有一个属性 _events，_events是一个对象，对象的 key 是事件名称，value 是函数或函数数组。几乎 EventEmitter 实例的所有方法都要操作 _events。
![eventEmitter](https://raw.githubusercontent.com/xudale/blog/master/assets/eventEmitter.png)

### EventEmitter 构造函数

EventEmitter 构造函数源码如下：
```JavaScript
function EventEmitter(opts) {
  EventEmitter.init.call(this, opts);
}

EventEmitter.init = function(opts) {
  if (this._events === undefined ||
      this._events === ObjectGetPrototypeOf(this)._events) {
    this._events = ObjectCreate(null);
    this._eventsCount = 0;
  }
};
```

EventEmitter 的构造函数实际上就是初始化 _events，当实例自身没有 _events 属性时，执行 this._events = ObjectCreate(null)，ObjectGetPrototypeOf 实际上相当于 Object.getPrototypeOf 的别名，用于获取实例原型链上是否存在 _events。

### EventEmitter.prototype.on

EventEmitter.prototype.on 的源码如下：

```JavaScript
EventEmitter.prototype.on = EventEmitter.prototype.addListener;

EventEmitter.prototype.addListener = function addListener(type, listener) {
  return _addListener(this, type, listener, false);
};

function _addListener(target, type, listener, prepend) {
  let m;
  let events;
  let existing;

  checkListener(listener);

  events = target._events;
  if (events === undefined) {
    events = target._events = ObjectCreate(null);
    target._eventsCount = 0;
  } else {
    // To avoid recursion in the case that type === "newListener"! Before
    // adding it to the listeners, first emit "newListener".
    if (events.newListener !== undefined) {
      target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);

      // Re-assign `events` because a newListener handler could have caused the
      // this._events to be assigned to a new object
      events = target._events;
    }
    existing = events[type];
  }
  // 核心逻辑从这里开始
  if (existing === undefined) {
    // 之前未注册过同名事件
    // Optimize the case of one listener. Don't need the extra array object.
    // 只有一个 listener 的情况，直接添加
    events[type] = listener; 
    ++target._eventsCount;
  } else {
    // 之前注册同名事件
    if (typeof existing === 'function') {
      // Adding the second element, need to change to array.
      existing = events[type] =
        prepend ? [listener, existing] : [existing, listener];
      // If we've already got an array, just append.
    } else if (prepend) {
      existing.unshift(listener);
    } else {
      existing.push(listener);
    }
  }

  return target;
}
```

逻辑很简单，读取 events[type] 判断之前是否注册同名事件，如果没有则 events[type] = listener；如果之前注册过同名事件，把 listener 插在数组中。最后返回 this，方便链式调用。

### EventEmitter.prototype.emit

EventEmitter.prototype.emit 的源码如下：

```JavaScript
EventEmitter.prototype.emit = function emit(type, ...args) {
  const events = this._events;
  const handler = events[type];

  if (handler === undefined)
    return false;

  if (typeof handler === 'function') {
    const result = ReflectApply(handler, this, args);

    // We check if result is undefined first because that
    // is the most common case so we do not pay any perf
    // penalty
    if (result !== undefined && result !== null) {
      addCatch(this, result, type, args);
    }
  } else {
    const len = handler.length;
    const listeners = arrayClone(handler, len);
    for (let i = 0; i < len; ++i) {
      const result = ReflectApply(listeners[i], this, args);

      // We check if result is undefined first because that
      // is the most common case so we do not pay any perf
      // penalty.
      // This code is duplicated because extracting it away
      // would make it non-inlineable.
      if (result !== undefined && result !== null) {
        addCatch(this, result, type, args);
      }
    }
  }

  return true;
};
```

删除了错误处理相关代码之后，逻辑很简单，根据事件名称 type，从 _events 对象中取出要执行的函数或者函数数组，对于触发的函数，使用 ReflectApply(listeners[i], this, args) 调用。ReflectApply 相当于 [Reflect.apply](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect/apply) 的别名。

### EventEmitter.prototype.off

EventEmitter.prototype.off 的源码如下：

```JavaScript
EventEmitter.prototype.off = EventEmitter.prototype.removeListener;

EventEmitter.prototype.removeListener =
    function removeListener(type, listener) {
      let originalListener;

      checkListener(listener);

      const events = this._events;

      const list = events[type];
      if (list === undefined)
        return this;

      if (list === listener || list.listener === listener) {
        if (--this._eventsCount === 0)
          this._events = ObjectCreate(null);
        else {
          delete events[type];
          if (events.removeListener)
            this.emit('removeListener', type, list.listener || listener);
        }
      } else if (typeof list !== 'function') {
        let position = -1;

        for (let i = list.length - 1; i >= 0; i--) {
          if (list[i] === listener || list[i].listener === listener) {
            originalListener = list[i].listener;
            position = i;
            break;
          }
        }

        if (position < 0)
          return this;

        if (position === 0)
          list.shift();
        else {
          if (spliceOne === undefined)
            spliceOne = require('internal/util').spliceOne;
          spliceOne(list, position); // spliceOne 类似 splice 方法，但在当前业务场景下效率更高些
        }

        if (list.length === 1)
          events[type] = list[0];
      }

      return this;
    };
```

off 方法和 on 方法源码除了逻辑相反外，看起来差不多；取 events[type]，如果是函数，直接 delete events[type]；如果是函数数组，则遍历数组，找到待删除的元素删除。

## 反射

从不同版本的 EventEmitter 的源码来看，反射用的越来越多，[Reflect](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect) 是一个内置的对象，它提供拦截 JavaScript 操作的方法，对象提供了以下静态方法：

![reflect](https://raw.githubusercontent.com/xudale/blog/master/assets/reflect.png)

Reflect 方法与 Object、Function 上同名方法功能很类似，使用 Reflect 会更加清晰优雅，而且把反射相关的方法收敛到 Reflect 对象，也与主流编程语言更接近。Reflect 方法在效率上应该是没有什么提升。比如本文使用的 Reflect.apply 方法，在 V8 中[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/x64/builtins-x64.cc#1739)：

```c++
void Builtins::Generate_ReflectApply(MacroAssembler* masm) {
  {
    Label done;
    StackArgumentsAccessor args(rsp, rax);
    __ LoadRoot(rdi, RootIndex::kUndefinedValue);
    __ movq(rdx, rdi);
    __ movq(rbx, rdi);
    __ cmpq(rax, Immediate(1));
    __ j(below, &done, Label::kNear);
    __ movq(rdi, args.GetArgumentOperand(1));  // target
    __ j(equal, &done, Label::kNear);
    __ movq(rdx, args.GetArgumentOperand(2));  // thisArgument
    __ cmpq(rax, Immediate(3));
    __ j(below, &done, Label::kNear);
    __ movq(rbx, args.GetArgumentOperand(3));  // argumentsList
    __ bind(&done);
    __ PopReturnAddressTo(rcx);
    __ leaq(rsp,
            Operand(rsp, rax, times_system_pointer_size, kSystemPointerSize));
    __ Push(rdx);
    __ PushReturnAddressFrom(rcx);
  }
  // 3. Apply the target to the given argumentsList.
  __ Jump(BUILTIN_CODE(masm->isolate(), CallWithArrayLike),
          RelocInfo::CODE_TARGET);
}
```

代码是 C++ 代码，风格是 X64 汇编的风格。这段代码在 V8 中的[宏处理](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/setup-builtins-internal.cc#280)后会生成一个内置函数。Generate_ReflectApply 的逻辑是取出 3 个参数，target、thisArgument、argumentsList 放在 3 个寄存器，最后调用 CallWithArrayLike，CallWithArrayLike 方法才是 Reflect.apply 的核心逻辑，Generate_ReflectApply 只能算是一个入口。Generate_ReflectApply 函数在不同平台有不同的汇编实现。

Function.prototype.apply 的[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/x64/builtins-x64.cc#1622):

```c++
void Builtins::Generate_FunctionPrototypeApply(MacroAssembler* masm) {
  {
    Label no_arg_array, no_this_arg;
    StackArgumentsAccessor args(rsp, rax);
    __ LoadRoot(rdx, RootIndex::kUndefinedValue);
    __ movq(rbx, rdx);
    __ movq(rdi, args.GetReceiverOperand());
    __ testq(rax, rax);
    __ j(zero, &no_this_arg, Label::kNear);
    {
      __ movq(rdx, args.GetArgumentOperand(1));
      __ cmpq(rax, Immediate(1));
      __ j(equal, &no_arg_array, Label::kNear);
      __ movq(rbx, args.GetArgumentOperand(2));
      __ bind(&no_arg_array);
    }
    __ bind(&no_this_arg);
    __ PopReturnAddressTo(rcx);
    __ leaq(rsp,
            Operand(rsp, rax, times_system_pointer_size, kSystemPointerSize));
    __ Push(rdx);
    __ PushReturnAddressFrom(rcx);
  }
  Label no_arguments;
  __ JumpIfRoot(rbx, RootIndex::kNullValue, &no_arguments, Label::kNear);
  __ JumpIfRoot(rbx, RootIndex::kUndefinedValue, &no_arguments, Label::kNear);

  // 4a. Apply the receiver to the given argArray.
  __ Jump(BUILTIN_CODE(masm->isolate(), CallWithArrayLike),
          RelocInfo::CODE_TARGET);

  // 4b. The argArray is either null or undefined, so we tail call without any
  // arguments to the receiver. Since we did not create a frame for
  // Function.prototype.apply() yet, we use a normal Call builtin here.
  __ bind(&no_arguments);
  {
    __ Set(rax, 0);
    __ Jump(masm->isolate()->builtins()->Call(), RelocInfo::CODE_TARGET);
  }
}
```

逻辑和 Generate_ReflectApply 差不多，取 3 个参数放在 3 个寄存器，最后调用 CallWithArrayLike。所以 Reflect 对象的方法性能不见得比 Function 和 Object 的对应方法性能更优秀，只是可读性更高而已。

今天（2020.07） nodejs 的最新版本 v14.5.0 的 EventEmitter 源码与本文的版本 v12.16.1 比较，可以看到 v14.5.0 把 Object.keys 替换为 Reflect.keys：

```c++
// v12.16.1
EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      const events = this._events;
      // Emit removeListener for all listeners on all events
      if (arguments.length === 0) {
        // 这里用 Object.keys 遍历 events
        for (const key of ObjectKeys(events)) {
          if (key === 'removeListener') continue;
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = ObjectCreate(null);
        this._eventsCount = 0;
        return this;
      }

      const listeners = events[type];

      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners !== undefined) {
        // LIFO order
        for (let i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };
```

```c++
// v14.5.0
EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      const events = this._events;
      // Emit removeListener for all listeners on all events
      if (arguments.length === 0) {
        // 最新版本用 Reflect.ownKeys 遍历 events
        for (const key of ReflectOwnKeys(events)) {
          if (key === 'removeListener') continue;
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = ObjectCreate(null);
        this._eventsCount = 0;
        return this;
      }

      const listeners = events[type];

      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners !== undefined) {
        // LIFO order
        for (let i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };
```

使用 Reflect.ownKeys 比 Object.keys 要更合理，因为 Object.keys 返回的数组中没有 Symbol。

考虑到事件名称可以是 Symbol，比如：

```JavaScript
const EventEmitter = require('events');
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
// 这里用 Symbol 当做事件名称
let sy = Symbol('abc')
myEmitter.on(sy, () => {
  console.log('Symbol event occured');
});
myEmitter.emit(sy);
```

v14.5.0 的 EventEmitter 源码前十几行是这样的：

```JavaScript
const {
  Boolean,
  Error,
  MathMin,
  NumberIsNaN,
  ObjectCreate,
  ObjectDefineProperty,
  ObjectGetPrototypeOf,
  ObjectSetPrototypeOf,
  Promise,
  PromiseReject,
  PromiseResolve,
  ReflectApply,
  ReflectOwnKeys,
  Symbol,
  SymbolFor,
  SymbolAsyncIterator
} = primordials;
```

从 nodejs 不同版本 EventEmitter 源码来看，ObjectDefineProperty、ObjectGetPrototypeOf 和 ObjectSetPrototypeOf 早晚会被 Reflect.defineProperty、Reflect.getPrototypeOf 和 Reflect.setPrototypeOf 方法所取代。


## 开枝散叶

EventEmitter 是很多类的父类，本小节分析 EventEmitter 是如何喜当爹的。笔者看了几处代码，写法都差不多，这里以 Stream 举例，Stream 构造函数源码如下：

```JavaScript
const EE = require('events');
function Stream(opts) {
  EE.call(this, opts);
}
ObjectSetPrototypeOf(Stream.prototype, EE.prototype);
ObjectSetPrototypeOf(Stream, EE);
```

Stream 继承 EventEmitter 的代码很简单，构造函数里面先是调用了 EventEmitter 的构造函数。分别设置了 Stream.prototype 和 Stream 的原型为 EE.prototype 和 EE。可以看出写法上已经极力向 class 写法靠拢。nodejs 源码很喜欢用比较新的 API 或 语法，笔者觉得未来有一天，这段代码可能会变成这样：

```JavaScript
const EE = require('events');
class MyStream extends EE {

}
```

笔者是半路出家做的前端，第一次见 JavaScript 的继承 Object.setPrototypeOf 的写法时，总是感觉万一要是写错了，就认贼做父了。而且继承这件事情通常在代码还没运行前的项目构想阶段就已经确定了，JavaScript 动态设置父类的写法总是有点奇怪。笔者还是更喜欢 class 写法。

## 简易版 EventEmitter

首先，注释掉 iterator.next() 后，在 d8 中运行代码，

```JavaScript
function* test() {
  yield 123456
}
let iterator = test()
// iterator.next() // {value: 123456, done: false}
```

生成器函数 test 编译后的字节码如下：

![generator-bytecode](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-bytecode.png)

当 let iterator = test() 开始执行时，V8 调用 Runtime_CreateJSGeneratorObject，创建一个生成器对象，Runtime_CreateJSGeneratorObject [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/runtime/runtime-generator.cc#46)：

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

Runtime_CreateJSGeneratorObject 的逻辑是创建 JSGeneratorObject 的实例，一个生成器对象。设置相关属性后，最后返回生成器对象 generator。些时生成器对象 generator 在累加器中，
我们忽略 Star r0 和 StackCheck，来看字节码 SuspendGenerator 的处理函数，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-generator.cc#3168)：

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

字节码 SuspendGenerator 的功能是暂停当前函数的执行，其字节码处理函数里面多次调用 StoreObjectField 来保存生成器函数当前运行的状态，最后返回累加器中的值，之前提到过，此时累加器存的是生成器对象 generator。所以 V8 代码中的生成器对象 generator 返回给了 JavaScript 代码中的 iterator。此时，生成器函数处于暂停状态，字节码执行到了本文第一张图所标识的“第一次暂停”的位置。

> let iterator = test()后，生成器函数开始执行，返回生成器对象，最后暂停
>
> 有了生成器对象后，可以调用其 next 方法让生成器函数继续执行

## iterator.next()——第二次暂停

上一节我们注释掉了 iterator.next()，当 JavaScript 代码继续执行 iterator.next() 时，生成器对象的 next 方法被调用，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/builtins-generator-gen.cc#286)：

```c++
void GeneratorBuiltinsAssembler::GeneratorPrototypeResume(
    CodeStubArguments* args, Node* receiver, Node* value, Node* context,
    JSGeneratorObject::ResumeMode resume_mode, char const* const method_name) {
  // CodeFactory::ResumeGenerator 使生成器函数从暂停处恢复执行
  Node* result = CallStub(CodeFactory::ResumeGenerator(isolate()), context,
                          value, receiver);
  // Make sure we close the generator if there was an exception.
  GotoIfException(result, &if_exception, &var_exception);
  TNode<Smi> result_continuation =
      CAST(LoadObjectField(receiver, JSGeneratorObject::kContinuationOffset));
  args->PopAndReturn(result);
}
```

生成器函数恢复执行需要 CPU 的寄存器操作，在笔者的 Mac 下，调用链路为GeneratorBuiltinsAssembler::GeneratorPrototypeResume->[CodeFactory::ResumeGenerator](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/codegen/code-factory.cc#278)->[Builtins::Generate_ResumeGeneratorTrampoline](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/x64/builtins-x64.cc#695)。生成器函数 test 在上一节已经处于暂停状态，调用 iterator.next() 使其从暂停处继续执行，高级语言当然没有这样的功能。V8 在这里对各大平台的汇编做了一层抽象，Builtins::Generate_ResumeGeneratorTrampoline 函数通过调用 X64 汇编，使生成器函数在暂停处恢复执行。Builtins::Generate_ResumeGeneratorTrampoline 代码很长，笔者只能看懂大概，这里只截取了几行，大致逻辑是将未来要返回的地址压栈，然后跳转到生成器函数 test 暂停的地方，继续执行。

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

ResumeGenerator 字节码处理函数与 SuspendGenerator 字节码处理函数逻辑基本相反，SuspendGenerator 保存当前生成器函数的各种执行状态，ResumeGenerator 恢复之前保存的状态，最后调用 Dispatch 函数，取出下一条字节码，执行。Dispatch 函数[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-assembler.cc#1396)：

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

如果把 V8 看做一台虚拟机，那么执行一条字节码后，必须要取出下一条字节码并执行，虚拟机上的程序才可能一直跑下去，ResumeGenerator 就是这样的，在其字节码处理函数的最后，调用 Dispatch，使程序继续执行，不会停止。事实上，几乎所有字节码处理函数最后都要调用 Dispatch。但 SuspendGenerator 的字节码处理函数最后没有调用 Dispatch，没有执行下面的字节码，生成器函数 test 暂停了。

ResumeGenerator 后，字节码一行一行往下执行，一直遇到下一个 SuspendGenerator，也就是本文第一张图所标示的“第二次暂停”，生成器函数又暂停了。第二次暂停是 yield 123456 带来的。


> yield 会被 V8 编译成 SuspendGenerator 和 ResumeGenerator 两条字节码，分别表示保存状态暂停，恢复状态继续执行

## async/await 与 generator 的关系
笔者在另外一篇文章分析过 async/await。async/await 和 generator 都有暂停当前函数执行，从暂停处恢复执行的能力，await 和 yield 对应的字节码都是 SuspendGenerator 和 ResumeGenerator，这方面它们没有区别。

生成器函数暂停时，需要调用生成器对象的 next 方法，才可以从暂停处恢复执行。async 函数依赖 Promise，与 microtask 联系紧密，V8 在执行 microtask 队列时，已经暂停的 async 函数恢复执行。

![generator-async](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-async.png)

> V8 源码中，async 函数依赖 Generator 和 Promise
>
> Generator 为 async 函数带来保存状态暂停，恢复状态执行的能力。Promise 为 async 函数带来自我驱动向下继续执行的能力，免去 next 的调动

## 3 种函数

JavaScript 的类型系统比较薄弱，比如 

```JavaScript
typeof 1 // number
typeof 0.1 // number
```

虽然 1 和 0.1 都是 number，但它们本质上是不同的类型，内存表示不一样，CPU 对整数和浮点数的运算指令也不一样，放在一个类型里面，有些勉为其难，但基本相安无事。

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

日常开发中，当一个组件/方法需要一个函数做为参数时，比如 forEach 和 map，多半需要的是 ES6 之前的函数，如果误传了 async 函数或者生成器函数，多半会出问题。因为 ES6 之前的函数、async 函数和生成器函数，虽然在 JavaScript 中 typeof 都返回 function，但在 V8 中它们是不同的类型，运行机制和返回值也不一样。

## 原生 generator 与 babel 转译的区别

日常开发中，生成器函数会被 babel 转译成类似下面的代码：
```JavaScript
async function test() {
  await 1
  await 2
  await 3
}

// babel 转译后
function test() {
  return _test.apply(this, arguments);
}

function _test() {
  _test = _asyncToGenerator( /*#__PURE__*/regeneratorRuntime.mark(function _callee() {
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
            _context.next = 2;
            return 1; // 第一次调用

          case 2:
            _context.next = 4;
            return 2; // 第二次调用

          case 4:
            _context.next = 6;
            return 3; // 第三次调用

          case 6:
          case "end":
            return _context.stop();
        }
      }
    }, _callee);
  }));
  return _test.apply(this, arguments);
}
```

看到这段代码笔者想起了一句话：

> 人不能两次踏进同一条河流    ——[赫拉克利特](https://baike.baidu.com/item/%E8%B5%AB%E6%8B%89%E5%85%8B%E5%88%A9%E7%89%B9)

babel 转译的代码中，test 函数被多次调用，但每次通过 switch case 语句时，都走向了不同的分支。由于 test 函数内部状态的改变，每次调用 test，都不再是过去的 test，而是一个新的 test。闭包保存了函数执行的状态，实现非常巧妙。但与 V8 中生成器函数的原理区别很大。babel 再怎么转，也转不出来字节码 SuspendGenerator 和 ResumeGenerator。

## 总结

1.生成器函数被调用时，生成器函数已经开始执行，返回生成器对象后，第一次暂停

2.iterator.next() 后，生成器函数回到第一次暂停的地方，恢复之前执行的状态，继续执行，遇到 yield（SuspendGenerator） 后，第二次暂停

![generator-rehab](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-rehab.png)











