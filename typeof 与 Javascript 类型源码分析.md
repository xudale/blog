# typeof 与 Javascript 类型源码分析.md
本文分析 typeof 在 V8 中的实现，及 Javascript 类型相关的源码，版本为 V8 7.7.1。
## typeof 源码分析

每一个 Javascript 对象都是 V8 中的 [JSObject](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/js-objects.h#278)，JSObject 继承 JSReceiver：

```c++
// The JSObject describes real heap allocated JavaScript objects with
// properties.
// Note that the map of JSObject changes during execution to enable inline
// caching.
class JSObject : public JSReceiver {
 public:
  static bool IsUnmodifiedApiObject(FullObjectSlot o);

  V8_EXPORT_PRIVATE static V8_WARN_UNUSED_RESULT MaybeHandle<JSObject> New(
      Handle<JSFunction> constructor, Handle<JSReceiver> new_target,
      Handle<AllocationSite> site);
  // 后面略
}
```

[JSReceiver](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/js-objects.h#24) 继承 HeapObject：

```c++
// JSReceiver includes types on which properties can be defined, i.e.,
// JSObject and JSProxy.
class JSReceiver : public HeapObject {
 public:
  NEVER_READ_ONLY_SPACE
  // Returns true if there is no slow (ie, dictionary) backing store.
  inline bool HasFastProperties() const;
  // 后面略
}
```


所以每一个 Javascript 对象也是 [HeapObject](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/heap-object.h#21)。

```c++
// HeapObject is the superclass for all classes describing heap allocated
// objects.
class HeapObject : public Object {
 public:
  bool is_null() const {
    return static_cast<Tagged_t>(ptr()) == static_cast<Tagged_t>(kNullAddress);
  }
  // [map]: Contains a map which contains the object's reflective
  // information.
  inline Map map() const; // 本文的重点
  inline void set_map(Map value);

  inline MapWordSlot map_slot() const;
  // 后面略
}
```
HeapObject 偏移量为 0 的位置，是 Map 对象的指针，这里的 Map 是一个 C++ 对象，也是本文的主角，[声明如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/map.h#96)：

```c++
// All heap objects have a Map that describes their structure.
//  A Map contains information about:
//  - Size information about the object
//  - How to iterate over an object (for garbage collection)
//
// Map layout:
// 
// 和类型相关的信息在此
// +----+----------+---------------------------------------------+
// | Int           | The second int field                        |
//  `---+----------+---------------------------------------------+
//      | Short    | [instance_type]   本文重点关注                          |
//      +----------+---------------------------------------------+
//      | Byte     | [bit_field]                                 |
//      |          |   - has_non_instance_prototype (bit 0)      |
//      |          |   - is_callable (bit 1)                     |
//      |          |   - has_named_interceptor (bit 2)           |
//      |          |   - has_indexed_interceptor (bit 3)         |
//      |          |   - is_undetectable (bit 4)                 |
//      |          |   - is_access_check_needed (bit 5)          |
//      |          |   - is_constructor (bit 6)                  |
//      |          |   - has_prototype_slot (bit 7)              |
//      +----------+---------------------------------------------+
//      | Byte     | [bit_field2]                                |
//      |          |   - is_extensible (bit 0)                   |
//      |          |   - is_prototype_map (bit 1)                |
//      |          |   - unused bit (bit 2)                      |
//      |          |   - elements_kind (bits 3..7)               |
class Map : public HeapObject {
 public:
  // Instance size.
  // Size in bytes or kVariableSizeSentinel if instances do not have
  // a fixed size.
  DECL_INT_ACCESSORS(instance_size)
  // Size in words or kVariableSizeSentinel if instances do not have
  // a fixed size.
  DECL_INT_ACCESSORS(instance_size_in_words)
  // 后面略
}
```

从 Map 的注释可以知道，Map 存储了关于对象大小、垃圾回收和类型相关的信息。和类型关系最密切的是 instance_type。

以 BigInt 类型举例，当执行以下代码：
```JavaScript
let big = 2n
console.log(typeof big)
```

d8 会打印 big 的类型，返回 bigint。[typeof](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#12693) 运算符核心代码如下：
```c++
Node* CodeStubAssembler::Typeof(Node* value) {
  VARIABLE(result_var, MachineRepresentation::kTagged);

  Label return_number(this, Label::kDeferred), if_oddball(this),
      return_function(this), return_undefined(this), return_object(this),
      return_string(this), return_bigint(this), return_result(this);

  GotoIf(TaggedIsSmi(value), &return_number);

  Node* map = LoadMap(value); // 1.获取 map 对象

  GotoIf(IsHeapNumberMap(map), &return_number);

  Node* instance_type = LoadMapInstanceType(map);// 2.获取 instance_type 字段
  // 3.通过 instance_type 判断 value 是不是函数、对象、字符串、bigint、symbol...，并跳转，返回相应类型字符串

  GotoIf(InstanceTypeEqual(instance_type, ODDBALL_TYPE), &if_oddball);

  Node* callable_or_undetectable_mask = Word32And(
      LoadMapBitField(map),
      Int32Constant(Map::IsCallableBit::kMask | Map::IsUndetectableBit::kMask));

  GotoIf(Word32Equal(callable_or_undetectable_mask,
                     Int32Constant(Map::IsCallableBit::kMask)),
         &return_function);

  GotoIfNot(Word32Equal(callable_or_undetectable_mask, Int32Constant(0)),
            &return_undefined);

  GotoIf(IsJSReceiverInstanceType(instance_type), &return_object);

  GotoIf(IsStringInstanceType(instance_type), &return_string);

  GotoIf(IsBigIntInstanceType(instance_type), &return_bigint);

  CSA_ASSERT(this, InstanceTypeEqual(instance_type, SYMBOL_TYPE));
  result_var.Bind(HeapConstant(isolate()->factory()->symbol_string()));
  Goto(&return_result);

  BIND(&return_number);
  {
    result_var.Bind(HeapConstant(isolate()->factory()->number_string()));
    Goto(&return_result);
  }

  BIND(&if_oddball);
  {
    Node* type = LoadObjectField(value, Oddball::kTypeOfOffset);
    result_var.Bind(type);
    Goto(&return_result);
  }

  BIND(&return_function);
  {
    result_var.Bind(HeapConstant(isolate()->factory()->function_string()));
    Goto(&return_result);
  }

  BIND(&return_undefined);
  {
    result_var.Bind(HeapConstant(isolate()->factory()->undefined_string()));
    Goto(&return_result);
  }

  BIND(&return_object);
  {
    result_var.Bind(HeapConstant(isolate()->factory()->object_string()));
    Goto(&return_result);
  }

  BIND(&return_string);
  {
    result_var.Bind(HeapConstant(isolate()->factory()->string_string()));
    Goto(&return_result);
  }

  BIND(&return_bigint);
  {
    result_var.Bind(HeapConstant(isolate()->factory()->bigint_string()));
    Goto(&return_result);
  }

  BIND(&return_result);
  return result_var.value();
}
```

HeapObject::kMapOffset 的值是 0，表示 Map 对象的指针在 HeapObject 对象上的偏移量。CodeStubAssembler::Typeof 的主要逻辑也很简单。既然要获取变量的类型，首先调用 LoadMap 取对象相关的 Map，[LoadMap](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#1470) 源码如下：
```c++
TNode<Map> CodeStubAssembler::LoadMap(SloppyTNode<HeapObject> object) {
  return UncheckedCast<Map>(LoadObjectField(object, HeapObject::kMapOffset,
                                            MachineType::TaggedPointer()));
}
```

HeapObject::kMapOffset 是值是 0，LoadMap 实质上是取对象 object 偏移量为 0 处的指针，也是是 Map 对象的地址。

拿到 Map 对象的地址后，从 Map 对象取 instance_type 字段，(源码如下)[https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#1592]：
```c++
TNode<Int32T> CodeStubAssembler::LoadMapInstanceType(SloppyTNode<Map> map) {
  return UncheckedCast<Int32T>(
      LoadObjectField(map, Map::kInstanceTypeOffset, MachineType::Uint16()));
}
```

Map::kInstanceTypeOffset 的值是 12，表示 instance_type 字段在 Map 对象上的偏移量。CodeStubAssembler::LoadMapInstanceType 的功能是从 Map 对象上取出 instance_type，instance_type 占用 16 bit 的空间。

取出 instance_type 后，把 instance_type 和函数、对象、字符串、bigint 和 symbol 等类型的 instance_type 做比较，以高级语言描述，其实就是一个有多个分支的 switch case 语句。以本文的 BigInt 为例，下面这行代码会执行：
```c++
  GotoIf(IsBigIntInstanceType(instance_type), &return_bigint);
```
[IsBigIntInstanceType](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#6534) 的定义很简单，判断 instance_type 和 BIGINT_TYPE 是否相等，BIGINT_TYPE 的值是 66。
```c++
TNode<BoolT> CodeStubAssembler::IsBigIntInstanceType(
    SloppyTNode<Int32T> instance_type) {
  return InstanceTypeEqual(instance_type, BIGINT_TYPE);
}

TNode<BoolT> CodeStubAssembler::InstanceTypeEqual(
    SloppyTNode<Int32T> instance_type, int type) {
  return Word32Equal(instance_type, Int32Constant(type));
}
```

> 在 V8 中，每一个 Javascript 对象都有一个与之关联的 Map 对象，Map 对象描述 Javascript 对象的信息
>
> Map 对象主要使用 16 bit 的 instance_type 字段描述对应 Javascript 对象的类型
## typeof 的两个坑
### typeof document.all 等于 undefined
```c++
// +----+----------+---------------------------------------------+
// | Int           | The second int field                        |
//  `---+----------+---------------------------------------------+
//      | Short    | [instance_type]   本文重点关注                          |
//      +----------+---------------------------------------------+
//      | Byte     | [bit_field]                                 |
//      |          |   - has_non_instance_prototype (bit 0)      |
//      |          |   - is_callable (bit 1)                     |
//      |          |   - has_named_interceptor (bit 2)           |
//      |          |   - has_indexed_interceptor (bit 3)         |
//      |          |   - is_undetectable (bit 4)                 |
//      |          |   - is_access_check_needed (bit 5)          |
//      |          |   - is_constructor (bit 6)                  |
//      |          |   - has_prototype_slot (bit 7)              |
//      +----------+---------------------------------------------+
//      | Byte     | [bit_field2]                                |
//      |          |   - is_extensible (bit 0)                   |
//      |          |   - is_prototype_map (bit 1)                |
//      |          |   - unused bit (bit 2)                      |
//      |          |   - elements_kind (bits 3..7)               |
```

Map 对象的 instance_type 相邻的字节定义了一些 bit，比如 is_callable，is_undetectable 和 is_constructor 等。null 和 undefined 的 is_undetectable bit 是 1，这点很容易理解。但同时也要看到，这些 bit 不是互斥的，一个含有丰富数据的对象，它的 is_undetectable 也可以是 1，比如：

![typeof_documentall](https://raw.githubusercontent.com/xudale/blog/master/assets/typeof_documentall.png)

document.all 明显不为空，但 typeof document.all 却返回 undefined，这是因为 document.all 的 Map 对象的 is_undetectable bit 是 1，真坑！

### typeof null 等于 object


至于前端~~经典~~的 typeof null === 'object'，由于 null 和 undefinde 的 is_undetectable bit 是 1，null 和 undefined 的流程应该是一样的，从源码的写法来看，为了避免出现 typeof null === 'undefined' 这种不合理的情况，V8 对 null 提前做了一层判断，就在 CodeStubAssembler::Typeof 函数比较早的一行。

```c++
GotoIf(InstanceTypeEqual(instance_type, ODDBALL_TYPE), &if_oddball);
```

null 的 instance_type 与 ODDBALL_TYPE（值为 67）相等，跳转到 if_oddball 标号执行。完美避开了后续的判断，ODDBALL_TYPE 如果翻译成中文的话，可能会叫奇怪类型。至少从 ODDBALL_TYPE 的命名来看，V8 也认为 null 是一个不走寻常路的类型。


> 从 Javascript 层面看，null 和 undefined 不是 Javascript 对象，但二者都是 C++ 对象
>
> null 和 undefined 的共同点是 is_undetectable bit 是 1，区别点在于 null 的 instance_type 是 ODDBALL_TYPE，V8 对 ODDBALL_TYPE（暂译为奇葩类型）做了提前特殊处理
## 为什么 1 + 1 = 2，1 + '1' = '11'？
![onePlusOne](https://raw.githubusercontent.com/xudale/blog/master/assets/onePlusOne.png)
1 + 1 = 2 请参考春晚小品。本文只讨论 1 + '1' = '11' 的情况

既然已经知道每个 Javascript 对象都有与之关联的 Map 对象来描述类型信息，那么只要知道左右两个操作数的类型，就可以判断是做加法还是做字符串相连。


从 V8 加法的字节码处理函数一路追起，[加法核心代码](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-number-gen.cc#359)如下，有删减。
```c++
TF_BUILTIN(Add, AddStubAssembler) {
  CodeStubAssembler::Print("TF_BUILTIN(Add");
  Node* context = Parameter(Descriptor::kContext);
  // 1.取得两个参数var_left、var_left
  VARIABLE(var_left, MachineRepresentation::kTagged,
           Parameter(Descriptor::kLeft));
  VARIABLE(var_right, MachineRepresentation::kTagged,
           Parameter(Descriptor::kRight));

  // We might need to loop several times due to ToPrimitive, ToString and/or
  // ToNumeric conversions.
  VARIABLE(var_result, MachineRepresentation::kTagged);
  Variable* loop_vars[2] = {&var_left, &var_right};
  Label loop(this, 2, loop_vars),
      string_add_convert_left(this, Label::kDeferred),
      string_add_convert_right(this, Label::kDeferred),
      do_bigint_add(this, Label::kDeferred);
  Goto(&loop);
  BIND(&loop);
  {
    Node* left = var_left.value();
    Node* right = var_right.value();

    Label if_left_smi(this), if_left_heapobject(this);
    // 2.根据 left 的类型，跳转不同的分支，本文 left 是一个小整数，跳转 BIND(&if_left_smi)
    Branch(TaggedIsSmi(left), &if_left_smi, &if_left_heapobject);
    BIND(&if_left_smi);
    {
      Label if_right_smi(this), if_right_heapobject(this);
      // 3.根据 right 的类型，跳转不同的分支，本文 right 是字符串，跳转 BIND(&if_right_heapobject) 执行
      Branch(TaggedIsSmi(right), &if_right_smi, &if_right_heapobject);

      BIND(&if_right_smi);
      {
        Label if_overflow(this);
        TNode<Smi> result = TrySmiAdd(CAST(left), CAST(right), &if_overflow);
        Return(result);

        BIND(&if_overflow);
        {
          var_left_double.Bind(SmiToFloat64(left));
          var_right_double.Bind(SmiToFloat64(right));
          Goto(&do_double_add);
        }
      }  // if_right_smi

      BIND(&if_right_heapobject);
      {
        Node* right_map = LoadMap(right);

        Label if_right_not_number(this, Label::kDeferred);
        GotoIfNot(IsHeapNumberMap(right_map), &if_right_not_number);

        // {right} is a HeapNumber.
        var_left_double.Bind(SmiToFloat64(left));
        var_right_double.Bind(LoadHeapNumberValue(right));
        Goto(&do_double_add);

        BIND(&if_right_not_number);
        {
          CodeStubAssembler::Print("if_right_not_number");
          Node* right_instance_type = LoadMapInstanceType(right_map);
          // 4.right 是字符串，跳转 BIND(&string_add_convert_left);
          GotoIf(IsStringInstanceType(right_instance_type),
                 &string_add_convert_left);
          GotoIf(IsBigIntInstanceType(right_instance_type), &do_bigint_add);
          ConvertAndLoop(&var_right, right_instance_type, &loop, context);
        }
      }  // if_right_heapobject
    }    // if_left_smi
  }
  BIND(&string_add_convert_left);
  {
    // Convert {left} to a String and concatenate it with the String {right}.
    // 5.最后执行到了这里，left 转成字符串后，与 right 进行字符串连接
    TailCallBuiltin(Builtins::kStringAdd_ConvertLeft, context, var_left.value(),
                    var_right.value());
  }
}
```
代码逻辑很简单，首先判断左边操作数的类型，是小整数。然后判断右面操作数的类型，是字符串，最后代码把左边的小整数转换成字符串，与右面操作数做字符串连接的逻辑。
## 总结
![type_save](https://raw.githubusercontent.com/xudale/blog/master/assets/type_save.png)












