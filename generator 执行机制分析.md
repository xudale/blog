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
## 初始化——第一次暂停

首先，注释掉 iterator.next()，

```JavaScript
function* test() {
  yield 123456
}
let iterator = test()
// iterator.next()
```

test 函数生成的字节码如下：

![generator-bytecode](https://raw.githubusercontent.com/xudale/blog/master/assets/generator-bytecode.png)

当 let iterator = test() 开始执行时，V8 会创建一个生成器对象，对应上图字节码中的 CreateJSGeneratorObject，CreateJSGeneratorObject [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/runtime/runtime-generator.cc#46)：

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

字节码 SuspendGenerator 的功能是暂停执行，其处理函数里面多次调用 StoreObjectField 来保存生成器函数当前运行的状态，最后返回累加器中的值，之前提到过，此时累加器存的是生成器对象 generator。所以生成器对象 generator 返回给了 JavaScript 代码中的 iterator。











## iterator.next()——第二次暂停





## async/await 与 generator 的关系

## 3 种函数














