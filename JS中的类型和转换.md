### js中的类型

#### undefined 类型

- 在js中`null`是关键字，而`undefined`只是一个普通的标识符，并且是一个全局属性，这意味着它的值可以被修改
- 可以使用`void 0`去安全得获取`undefined`

#### NaN

- js中唯一**不等于自身**的值

#### BigInt

- BigInt 并不是 `===` 有着相同数学值的 Number，而是 `==` 相等

#### Object

- js中除了`Object`，其他均为原始值

### 对象的强制转换

#### 对象强制类型转换的可能性

路径一： 原始值强制转换：`ToPrimitive("default")` → `valueOf()` → `toString()`

> - 原始值强制转换用于得到一个期望的原始值，但对实际类型应该是什么并没有强烈的偏好
> - 只有`Date()`传入Object时会走该路径（暂不确定，v8源码中只有Date的构造函数未传入`hint`）

路径二：数字类型强制转换、number 类型强制转换、BigInt 类型强制转换`ToPrimitive("number")` → `valueOf()` → `toString()`

路径三：字符串类型强制转换：`ToPrimitive("string")` → `toString()` → `valueOf()`

#### 强制转换的ECMA规范定义

1. `ToPrimitive(input, PreferredType)` ：
   - 如果输入值 `input` 的类型是对象，根据以下步骤进行转换：
     - 如果没有指定 `PreferredType`，则将 `hint` 设置为 "default"。
     - 如果 `PreferredType` 是字符串类型，则将 `hint` 设置为 "string"。
     - 如果 `PreferredType` 是数字类型，则将 `hint` 设置为 "number"。
     - 检查对象是否定义了 `@@toPrimitive` 方法，如果是，则调用该方法并传递 `hint` 作为参数，将返回值作为结果，如果未定义就使用`OrdinaryToPrimitive`(见下文)。
     - 如果返回的结果不是对象类型，直接返回该结果。
     - 否则，抛出 TypeError 异常。
   - 如果输入值的类型不是对象，则直接返回该值。

2. `OrdinaryToPrimitive(input, hint)` ：
   - 当使用参数 `input` 和 `hint` 调用 `OrdinaryToPrimitive` 时，执行以下步骤：
     - 如果 `hint` 是 "string"，则按照顺序尝试调用对象的 `toString()` 和 `valueOf()` 方法。
     - 否则，按照顺序尝试调用对象的 `valueOf()` 和 `toString()` 方法。
     - 对于每个方法名按顺序执行以下步骤：
       - 令 `method` 为从对象 `input` 中获取的方法。
       - 如果 `method` 是可调用的，则调用该方法并将对象 `input` 作为上下文。
       - 如果调用结果的类型不是对象，则直接返回该结果。
     - 抛出 TypeError 异常。

> - 只有`Date`和`Symbol`类型有[`[@@toPrimitive]`](#相关的v8源码)方法
> - `Date.prototyp[@@toPrimitive]` 将 `hint` 的`"default"` 视为 `"string"` 并且调用 `toString()` 而不是 `valueOf()`
> - `Symbol.prototype[@@toPrimitive]` 忽略 `hint`，并总是返回一个 `symbol`
> - `@@toPrimitive`如果存在，必须返回原始值，否则会报错TypeError




### 相关的V8源码

#### ToPrimitive

```c++
MaybeHandle<Object> JSReceiver::ToPrimitive(Isolate* isolate,
                                            Handle<JSReceiver> receiver,
                                            ToPrimitiveHint hint) {
  Handle<Object> exotic_to_prim;
  ASSIGN_RETURN_ON_EXCEPTION(
      isolate, exotic_to_prim,
      Object::GetMethod(isolate, receiver,
                        isolate->factory()->to_primitive_symbol()),
      Object);
  if (!IsUndefined(*exotic_to_prim, isolate)) {
    Handle<Object> hint_string =
        isolate->factory()->ToPrimitiveHintString(hint);
    Handle<Object> result;
    ASSIGN_RETURN_ON_EXCEPTION(
        isolate, result,
        Execution::Call(isolate, exotic_to_prim, receiver, 1, &hint_string),
        Object);
    if (IsPrimitive(*result)) return result;
    THROW_NEW_ERROR(isolate,
                    NewTypeError(MessageTemplate::kCannotConvertToPrimitive),
                    Object);
  }
  return OrdinaryToPrimitive(isolate, receiver,
                             (hint == ToPrimitiveHint::kString)
                                 ? OrdinaryToPrimitiveHint::kString
                                 : OrdinaryToPrimitiveHint::kNumber);
}
```

> 逻辑说明：
>
> 1. 获取 `receiver` 对象中的 `to_primitive_symbol` 方法。
> 2. 如果存在 `to_primitive_symbol` 方法，则调用该方法并返回结果。
> 3. 如果不存在 `to_primitive_symbol` 方法，则调用 `OrdinaryToPrimitive` 方法进行转换，并返回转换后的结果。

#### @@toPrimitive

```c++
// v8/src/init/bootstrapper.cc
// Install the @@toPrimitive function at Symbol.
    InstallFunctionAtSymbol(
        isolate_, prototype, factory->to_primitive_symbol(),
        "[Symbol.toPrimitive]", Builtin::kSymbolPrototypeToPrimitive, 1, true,
        static_cast<PropertyAttributes>(DONT_ENUM | READ_ONLY));
// Install the @@toPrimitive function at Date.
    InstallFunctionAtSymbol(
        isolate_, prototype, factory->to_primitive_symbol(),
        "[Symbol.toPrimitive]", Builtin::kDatePrototypeToPrimitive, 1, true,
        static_cast<PropertyAttributes>(DONT_ENUM | READ_ONLY));
```



