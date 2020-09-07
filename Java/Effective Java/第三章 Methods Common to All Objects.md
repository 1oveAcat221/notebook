# 第三章 Methods Common to All Objects

## Item 10: Obey the general contract when overriding equals

**满足以下条件时，不需要重写`Object.equals()`**

- 每一个实例都是独一无二的
- 不需要提供判断两个实例“逻辑相等”的功能
- 父类已经重写过了，并且父类的实现逻辑适用于子类
- 类是私有的，并且确定`equals`方法永远用不上
- 采用了实例控制（Item 1）的类，每个特定的值都只有一个实例，如`Enum`（Item 34）。这种场景下“逻辑相等”和`==`是相同的。

可以这样重写防止被意外调用：

```java
@Override public boolean equals(Object o) {
    throw new AssertionError(); // Method is never called
}
```

在需要提供判断“逻辑相等”的功能，而不是判断两个引用是否相等时需要重写`Object.equals()`，这样能让依赖“逻辑相等”的代码能够正常工作，如 map 和 set。

**`equals`方法目的是实现一种等价关系，实现需要满足以下条件：**

- 自反性（Reflexive）：任意非空引用 `x`，`x.equals(x)`必须返回`true`。
- 对称性（Symmetric）：任意非空引用`x`和`y`，`x.equals(y)`和`y.equals(x)`必须都返回`true`。
- 传递性（Transitive）：任意非空引用`x`，`y`和`z`，若`x.equals(y)`和`y.equals(z)`都返回`true`，则`x.equals(z)`返回`true`。
    > 在面向对象中，**不可能**在继承父类（非抽象类）后添加一个需要判断相等的字段，同时遵守`equals`的约定。  
    > 在`equals`方法中利用`getClass`方法判断具体的类可以遵守约定，但是这样会打破*里氏替换原则*。  
    > **有一个变通的方法：采用组合而不是继承。**
- 一致性（Consistent）：多次调用必须返回相同的结果
    > 不要在判断时依赖不可靠的资源，如DNS解析
- 对于任意非空引用`x`，`x.equals(null)`必须返回`false`。
    > 没有必要显式判断空值：`if (o == null)`  
    > `o instanceof MyType`会直接返回`false`

**总结：**

- 先使用`==`判断是否是同一个对象  
    一个优化手段，特别是判断相等开销较高时
- 使用`instanceof`判断参数类型是否正确  
    如果实现了接口，可以判断是否是同一个接口
- 将参数强制类型转换  
    `instanceof`之后，转换一定是成功的
- 判断字段是否相等  
    如果是接口则需要通过接口方法访问字段  
    
    - 除`double`和`float`之外的基本类型使用`==`
    - `double`和`float`需要使用`Float.compare(float, float)`和`Double.compare(double, double)`。`Float.equals`和`Double.equals`也可以，但是有自动装箱，性能不好。
    - 引用类型则递归调用`equals`，使用`Objects.equals(Object, Object)`防止`NullPointerException`。
    - 数组类型需要逐一对比元素
- 优先判断容易不匹配的字段，可以提早返回，提高性能
- 不需要判断可以通过主要字段导出的字段，或者当导出字段可以区别对象时优先判断导出字段

**一个正确的例子：**
```java
// Class with a typical equals method
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;
    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix = rangeCheck(prefix, 999, "prefix");
        this.lineNum = rangeCheck(lineNum, 9999, "line num");
    }
    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
            return pn.lineNum == lineNum && pn.prefix == prefix
                    && pn.areaCode == areaCode;
    }
    ... // Remainder omitted
}
```

**注意事项：**
- 重写`equals`之后记得要重写`hashCode`
- 只应当用于同类对象字段相等的对比，不能用来比较不同对象的相等性
- `equals`方法的参数类型永远只能是`Object`，重写时加上`Override`注解
- 可以使用Google的 Auto Value 框架直接生成`equals`方法，也可以利用 IDE 生成

## Item 11: Always override hashCode when you override equals





