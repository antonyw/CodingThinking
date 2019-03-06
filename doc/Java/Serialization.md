# 为什么需要序列化
内存中的数据只有转换为二进制流才可以进行数据持久化和网络传输。
##### 序列化就是将数据对象转换为二进制流的过程。
##### 反之，将二进制流恢复为数据对象的过程称为反序列化。

# 序列化的方式
## Java 原生序列化
Java 类通过实现 Serializable 接口来实现该类对象的序列化，这个接口非常特殊，没有任何方法，只起标识作用。兼容性最好，但不支持跨语言，而且性能一般。
### serialVersionUID
实现 Serializable 建议设置这个字段，如果不设置，那么每次运行时，编译器会根据类的内部实现，包括类名、接口名、方法和属性等来自动生成。如果类的源代码有修改，那么重新编译后 serialVersionUID 的取值可能会发生变化。所以，显式的定义一个 serialVersionUID 就很重要了。
- 如果是兼容升级，就不修改 serialVersionUID，避免反序列化失败。
- 如果是不兼容升级，需要修改 serialVersionUID，避免反序列化混乱。

## Hessian 序列化
Hessian 序列化是一种动态类型、跨语言、基于对象传输的网络协议。
### 原理
Hessian 会把复杂对象所有属性存储在一个Map 中进行序列化。所以在父类、子类存在同名成员变量的情况下，Hessain 序列化时，先序列化子类，然后序列化父类，因此反序列化结果会导致子类同名成员变量被父类的值覆盖。

## JSON 序列化
JSON 序列化就是将数据对象转换为 JSON 字符串。在序列化过程中抛弃了类型信息，所以反序列化时只有提供类型信息才能准确地反序列化。
相比前两种方式，JSON 可读性比较好，方便调试。