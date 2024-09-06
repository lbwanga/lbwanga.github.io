## 基本数据类型

用 new 创建对象（特别是小的、简单的变量）并不是非常有效，因为new 将对象置于“堆”里。对于这些类型，Java 采纳了与 C 和 C++相同的方法。也就是说，不是用 new 创建变量，而是创建一个并非句柄的“自动”变量。这个变量容纳了具体的值，并置于堆栈中，能够更高效地存取。

boolean（1）

char（16）

byte（8）、short（16）、int（32）、long（64）

float（32）、double（64）



数值类型全都是有符号（正负号）的，所以不必费劲寻找没有符号的类型。



## BigInteger 和 BigDecimal

BigInteger 适合保存比较大的整数

BigDecimal 适合保存精度更高的浮点数

能对int 或 float 做的事情，对BigInteger 和BigDecimal 一样可以做。只是必须使用方法调用，不能使用运算符。例如`add, subtract, multiply, divide `，不能直接使用`+ - * /`



**BigDecimal**

我们在使用 `BigDecimal` 时，为了防止精度丢失，推荐使用它的`BigDecimal(String val)`构造方法或者 `BigDecimal.valueOf(double val)` 静态方法来创建对象，valueOf内部执行toString方法。

使用 `divide` 方法的时候尽量使用 3 个参数版本，并且`RoundingMode` 不要选择 `UNNECESSARY`，否则很可能会遇到 `ArithmeticException`（无法除尽出现无限循环小数的时候），其中 `scale` 表示要保留几位小数，`roundingMode` 代表保留规则。

```java
public BigDecimal divide(BigDecimal divisor, int scale, RoundingMode roundingMode) {
    return divide(divisor, scale, roundingMode.oldMode);
}
```



BigDecimal大小比较应使用compareTo()方法，而不是equals()方法。因为 `equals()` 方法不仅仅会比较值的大小（value）还会比较精度（scale），而 `compareTo()` 方法比较的时候会忽略精度。



## 作用域

作用域是由花括号的位置决定的。Java 对象不具备与基本类型一样的存在时间，用 new 关键字创建一个Java 对象的时候，它会超出作用域的范围之外。

```java
{
	String s = new String("a string");
} /* 作用域的终点 */
```

句柄s会在作用域的终点处消失。然而，s 指向的 String 对象依然占据着内存空间。在上面这段代码里，我们没有办法访问对象，因为指向它的唯一一个句柄已超出了作用域的边界。

Java 有一个特别 的“垃圾收集器”，它会查找用new 创建的所有对象，并辨别其中哪些不再被引用。随后，它会自动释放由那些闲置对象占据的内存，以便能由新对象使用。



## 类型转换

自动类型转换

* 精度小的类型自动转换为精度大的类型

* **byte、short、char不会相互转换**

* **byte, short, char 三者之间可以计算，在计算时首先转换为int类型**

* boolean类型不参与自动转换

强制类型转换: 使用强制转换符`()`

* 强制符号只针对最近的操作数有效，可以使用小括号提升优先级

基本数据类型和字符串之间转换

* 基本数据类型->字符串：基本数据类型 + ""

* 字符串->基本数据类型：通过基本数据类型的包装类调用parseXXX方法即可

  ```java
  int num = Integer.parseInt(s);
  ```



## 运算符

**赋值**：

对基本数据类型的赋值是非常直接的。由于基本类型容纳了实际的值，而且并非指向一个对象的句柄，所以在为其赋值的时候，可将来自一个地方的内容复制到另一个地方。

但在为对象“赋值”的时候，情况却发生了变化。对一个对象进行操作时，我们真正操作的是它的句柄。所以倘若“从一个对象到另一个对象”赋值，实际就是将句柄从一个地方复制到另一个地方。

**算术运算符**：

中包括加号（+）、减号（-）、除号 （/）、乘号（*）以及模数（%，从整数除法中获得余数）

取模：本质：a%b = a - a / b * b。a为小数，则：a%b = a - (int)a / b * b

**自动递增、递减**：

对于前递增和前递减（如++A 或--A），会先执行运算，再生成值。而对于后递增和后递减（如A++或A--）， 会先生成值，再执行运算。

**关系运算符**：

关系运算符包括小于（<）、大于 （>）、小于或等于（<=）、大于或等于（>=）、等于（==）以及不等于（!=）。等于和不等于适用于所有内建的数据类型，但其他比较不适用于boolean 类型。

若想对比两个对象的实际内容是否相同，又该如何操作呢？此时，必须使用所有对象都适用的特殊方法 equals()。但这个方法不适用于“主类型”，那些类型直接使用==和!=即可。

**逻辑运算符**：

逻辑运算符 AND（&&）、OR（||）以及 NOT（!）能生成一个布尔值（true 或 false）。操作逻辑运算符时，我们会遇到一种名为“短路”的情况。因此，一个逻辑表达式的所有部分都有可能不进行求值。

**按位运算符**：

与（&）、或（|）、非（~）、异或（^）

**移位运算符**：

左移位运算符（<<）能将运算符左边的运算对象向左移动运算符右侧指定的位数（在低位补 0）。

“有符号”右移位运算符（>>）则将运算符左边的运算对象向右移动运算符右侧指定的位数。“有符号”右移位运算符使用了 “符号扩展”：若值为正，则在高位插入 0；若值为负，则在高位插入1。

Java 也添加了一种“无符号”右移位运算符（>>>），它使用了“零扩展”：无论正负，都在高位插入0。



若对char，byte 或者short 进行移位处理，那么在移位进行之前，它们会自动转换成一个 int。只有移位数字右侧的 5 个低位才会用到。这样可防止我们在一个 int 数里移动不切实际的位数。

若对一个 long 值进行处理，最后得到的结果也是long。此时只会用到移位数字右侧的 6 个低位，防止移动超过 long 值里现成的位数。

**三元运算符**：

布尔表达式 ? 值 0:值 1

**字符运算符+**：

有一方为字符串时，连接不同的字串。

**造型运算符**：

“造型”（Cast）的作用是“与一个模型匹配”。在适当的时候，Java 会将一种数据类型自动转换成另一 种。例如，假设我们为浮点变量分配一个整数值，计算机会将 int 自动转换成 float。通过造型，我们可明确设置这种类型的转换，或者在一般没有可能进行的时候强迫它进行。 为进行一次造型，要将括号中希望的数据类型（包括所有修改符）置于其他任何值的左侧。

Java 允许我们将任何主类型“造型”为其他任何一种主类型，但布尔值（bollean）要除外，后者根本不允许进行任何造型处理。“类”不允许进行造型。

对主数据类型执行任何算术或按位运算，只要它们“比int 小”（即 char，byte 或者 short），那么在正式执行运算之前，那些值会自动转换成int。这样一来，最终生成的值就是int 类型。所以只要把一个值赋回较小的类型，就必须使用“造型”。此外，由于是将值赋回给较小的类型，所以可能出现信息丢失的情况。通常，表达式中最大的数据类型是决定了表达式最终结果大小的那个类型。若将一个 float 值与一个double 值相乘，结果就是 double；如将一个 int 和一个 long 值相加，则结果为 long。



## 可变参数

* java中允许将同一个类中**多个同名同功能但参数个数不同**的方法，封装成一个方法。通过可变参数实现

  ```
  修饰符 返回类型 方法名（数据类型... 形参名）{}
  public int sum(int... nums){}//nums当作数组使用
  ```

* 可变参数的实参可以是0个或任意多个

* 可变参数实参可以是数组，可变参数的本质就是数组

* 可变参数可以和普通类型的参数一起放在形参列表，但必须保证**可变参数在最后**

* 一个形参列表**只能出现一个**可变参数



## 访问修饰符

| 访问级别 | 修饰符    | 本类 | 同包 | 子类 | 不同包 |
| -------- | --------- | ---- | ---- | ---- | ------ |
| 公开     | public    | √    | √    | √    | √      |
| 受保护   | protected | √    | √    | √    | ×      |
| 默认     |           | √    | √    | ×    | ×      |
| 私有     | private   | √    | ×    | ×    | ×      |

修饰符 public 表示对所有类可⻅。

修饰符 protected 表示对同⼀包内的类和所有⼦类可⻅。⼦类可以访问⽗类中声明为 protected 的成员， ⽽不管⼦类与⽗类是否在同⼀包中。 

如果没有使⽤任何访问修饰符（即没有写 public 、 protected 、 private ），则默认为包级别访问。这意 味着只有同⼀包中的类可以访问。 

修饰符 private 表示对同⼀类内可⻅。私有成员只能在声明它们的类中访问

修饰符可以用来修饰属性，成员方法和类，只有默认和`public`才能修饰类



## 面向对象

面向对象：把类或对象作为基本单元来组织代码。

**封装**

将对象的字段和方法封装在一个类中，并通过访问控制来隐藏对象的内部实现细节。外部不能直接访问对象的内部数据，只能通过提供的公共方法来操作数据。

**继承**

继承是一种机制，允许子类继承父类的属性和方法，通过继承，子类可以复用父类的代码，并可以在子类中扩展或重写父类的方法。

```java
class 子类 extends 父类{
    // 
}
```

* 子类继承了所有属性和方法，但是私有属性和方法不能在子类直接访问，要通过父类公共的方法
* 子类必须调用父类的构造器完成父类的初始化
* 当创建子类的对象时，默认总会去调用父类的无参构造器，如果父类没有提供无参构造器，则必须在子类的构造器中用`super(参数列表);`去指定使用父类的哪个构造器完成父类的初始化工作，否则编译不会通过。
* `super()`和`this()`都只能放在构造器第一行，因此不能共存在一个构造器
* 所有类都是`Object`类的子类
* 子类只能继承一个父类，即单继承机制



构建器的调用遵照下面的顺序：

(1) 调用基础类构建器。这个步骤会不断重复下去，首先得到构建的是分级结构的根部，然后是下一个衍生类，等等。直到抵达最深一层的衍生类。 

(2) 按声明顺序调用成员初始化模块。 

(3) 调用衍生构建器的主体。



this的两个含义：

1. 指示隐式参数的引用，可为已调用了其方法的那个对象生成相应的句柄
2. 调用该类的其他构造

super的两个含义：

1. 调用超类的方法
2. 调用超类的构造器

调用构造器的语句只能作为另一个构造器的第一条语句出现

| 区别点     | this                                     | super                        |
| ---------- | ---------------------------------------- | ---------------------------- |
| 访问属性   | 访问本类中属性，如果没有从父类中继续查找 | 从父类开始查找属性           |
| 调用方法   | 访问本类中方法，如果没有从父类中继续查找 | 从父类开始查找方法           |
| 调用构造器 | 调用本类构造器，必须放在首行             | 调用父类构造器，必须放在首行 |
| 特殊       | 表示当前对象                             | 子类中访问父类对象           |



**多态**

指同一个方法或对象在不同场景下可以表现出不同的行为。Java中主要通过重载和重写实现。

多态的具体表现：

* 方法的多态：
  1. 重写：子类重写父类的方法
  2. 重载：同一个类中可以有多个同名方法，但参数列表不同
* 对象的多态：
  1. 一个对象的编译类型（对象类型）和运行类型（引用类型）可以不一致
  2. 编译类型在定义对象时就确定了，不能改变
  3. 运行类型是可以变化的
  4. 编译类型看等号左边，运行类型看等号右边



重写：

* 子父类中的重写方法在对应的class文件常量池的位置相同，一旦子类没有重写，那么子类的实例就会沿着这个位置往上找，直到找到父类的同名方法。
* 重写只发生在可见的实例方法中：静态方法不存在重写；私有方法不存在重写；
* 重写满足一个规则：两同 两小 一大
  * 两同：方法名、参数列表相同
  * 两小：重写方法的返回值和抛出异常类型要和被重写方法的相同或是其子类。
  * 一大：修饰符 >= 被重写方法的修饰符



## 深拷贝和浅拷贝

**浅拷贝**：浅拷⻉创建⼀个新对象，然后将原对象的⾮静态字段复制到新对象。如果字段是基本数据类型，那么就复制其值；如果字段是引⽤类型，复制的就是引⽤地址，和原对象共用同一个内部对象

**深拷贝**：创建⼀个新对象，并递归复制原对象中的所有引⽤类型的字段指向的对象，⽽不是共享引⽤。因此， 新对象和原对象中的引⽤类型字段引⽤的是两组不同的对象。

**引用拷贝**：引用拷贝就是两个不同的引用指向同一个对象。

![](./images/shallow&deep-copy.png)



## Object类

==运算符

* 对于基本数据类型来说，`==` 比较的是值。
* 对于引用数据类型来说，`==` 比较的是对象的内存地址。

equals方法

* 不能用于判断基本数据类型的变量，只能用来判断两个对象是否相等。
* 默认判断地址是否相等，子类往往重写该方法，用于判断内容是否相等

hashCode

* 哈希值主要根据地址号来的，但不能将哈希值等价子地址

toString

* 默认返回：全类名@哈希值的十六进制

finalize

* 当对象被回收时，系统自动调用该对象的finalize方法，子类可以重写该方法用于一些资源释放操作
* 当对象没有任何引用时，jvm就认为该对象是一个垃圾对象，就会使用垃圾回收机制销毁对象
* 垃圾回收机制的调用，由系统决定，也可以通过 `System.gc()` 主动触发垃圾回收机制



## static

**类变量/静态变量**

* `static`变量被对象所共享，在**类加载的时候就生成**
* 静态变量保存在class实例的尾部，而class对象**保存在堆中**

* 生命周期为类加载到类消亡



**类方法/静态方法**

静态方法是不在对象上执行的方法，没有隐式参数，没有this（非静态方法中，this指向这个方法的隐式参数）。不能在对象上执行操作，但是对象可以调用静态方法，建议使用类名调用静态方法

当方法中不涉及任何对象相关的成员或一些通用的方法，可以设计成静态方法提高开发效率

* 类方法和普通方法都随着类加载而加载，将结构信息存储到方法区
* 类方法中**不允许使用和对象有关的关键字**，如`this`和`super`
* **静态方法只能访问静态变量或静态方法**，非静态方法可以访问静态成员和非静态成员



**静态代码块**

类加载时执行并且只执行一次



**静态内部类**

静态成员可以被类直接访问，而不需要创建类实例；而静态成员无法直接访问非静态类，因为非静态成员依赖类实例。



## main方法

* main方法是虚拟机调用，访问权限必须是public
* 执行main方法时不必创建对象，所以必须是static
* 接收String类型的数组形参，保存运行时传递的参数
* main方法中可以直接使用所在类的静态属性和静态方法；访问非静态成员，必须创建对象去调用



## 代码块

代码块又叫初始化块，在加载类或创建对象时隐式调用

```
[修饰符]{
	代码
}
```

* 修饰符只能选`static`，分别称为静态代码块和普通代码块。**静态代码块随类的加载而执行，只执行一次，而普通代码块每创建对象都会执行**

* 静态代码块只能调用静态成员，普通代码块可以调用任意成员
* 好处：
  * 相当于另一种形式的构造器，可以做初始化操作，代码块的**调用优先于构造器**
  * 如果多个构造器中都有重复语句，可以抽取到代码块中，提高复用性

**类什么时候加载？**

1. 创建对象实例（new)

2. 创建子类对象实例，父类也会被加载

3. 使用类的静态成员时

**创建一个对象时，在一个类调用顺序**

1. 调用静态代码块和静态属性初始化（多个则按照定义顺序）
2. 普通代码块和普通属性初始化
3. 调用构造方法。构造器的最前面其实隐藏了`super()`和调用普通代码块

**创建一个子类对象时，调用顺序**

1. 父类的静态代码块和静态属性
2. 子类的静态代码块和静态属性
3. 父类的普通代码块和普通属性初始化
4. 父类的构造方法
5. 子类的普通代码块和普通属性初始化
6. 子类的构造方法



## final

**final数据**：

许多程序设计语言都有自己的办法告诉编译器某个数据是“常数”。常数主要应用于下述两个方面： 

(1) 编译期常数，它永远不会改变 

(2) 在运行期初始化的一个值，我们不希望它发生变化

对于编译期的常数，编译器（程序）可将常数值“封装”到需要的计算过程里。也就是说，计算可在编译期间提前执行，从而节省运行时的一些开销。在 Java 中，这些形式的常数必须属于基本数据类型（Primitives），而且要用 final 关键字进行表达。在对这样的一个常数进行定义的时候，必须给出一个值。

无论static 还是 final 字段，都只能存储一个数据，而且不得改变。 

对于基本数据类型，final 会将值变成一个常数；但对于对象句柄，final 会将句柄变成一个常数。进行声明时，必须将句柄初始化到一个具体的对象。而且永远不能将句柄变成指向另一个对象。然而，对象本身是可以修改的。Java 对此未提供任何手段，可将一个对象直接变成一个常数（但是，我们可自己编写一个类，使其中的对象具有 “常数”效果）。这一限制也适用于数组，它也属于对象。

final数据必须赋初值，以后不能修改，初始化位置：

1. 定义时
2. 在构造器中
3. 在代码块中



**final方法**：

之所以要使用final 方法，可能是出于对两方面理由的考虑。第一个是为方法“上锁”，防止任何继承类改变它的本来含义。设计程序时，若希望一个方法的行为在继承期间保持不变，而且不可被覆盖或改写，就可以采取这种做法。 

采用final 方法的第二个理由是程序执行的效率。将一个方法设成 final 后，编译器就可以把对那个方法的所有调用都置入“嵌入”调用里。只要编译器发现一个final 方法调用，它会用方法主体内实际代码的一个副本来替换方法调用。这样做可避免方法调用时的系统开销。当然，若方法体积太大，那么程序也会变得雍肿，可能受到到不到嵌入代码所带来的任何性能提升。因为任何提升都被花在方法内部的时间抵消了。Java 编译器能自动侦测这些情况，并颇为“明智”地决定是否嵌入一个 final 方法。

通常，只有在方法的代码量非常少，或者想明确禁止方法被覆盖的时候，才应考虑将一个方法设为 final。

类内所有private 方法都自动成为final。由于我们不能访问一个 private 方法，所以它绝对不会被其他方法覆盖。



**final类**：

将类定义成 final 后，结果只是禁止进行继承——没有更多的限制。然 而，由于它禁止了继承，所以一个 final 类中的所有方法都默认为final。因为此时再也无法覆盖它们。所以与我们将一个方法明确声明为final 一样，编译器此时有相同的效率选择。 




## 抽象类

某些方法不确定实现时可以声明为抽象方法让子类来实现，含抽象方法的类称为抽象类。

抽象类本质是一个类（包含属性、方法、构造器、代码块、内部类），只不过是一种特殊的类，即使有构造器这种类也不能被实例化为对象，只能被子类继承。

声明为抽象方法：`public abstract void eat();` 没有方法体，此时类也必须声明为`abstract`类

注意：

* 抽象类不能被实例化
* 抽象类不一定要包含`abstract`方法，一旦包含`abstract`方法就必须声明为抽象类
* `abstract`只能用于声明类和方法
* 如果一个类继承了抽象类，则它必须实现抽象类的所有抽象方法，除非它自己也声明为抽象类

* 抽象方法不能用`private`, `final`, `static`来修饰，因为这些关键字于重写相违背。静态方法可以被子类继承，但不能被重写



## 接口

“interface”（接口）关键字使抽象的概念更深入了一层。我们可将其想象为一个“纯”抽象类。它允许创建者规定一个类的基本形式：方法名、自变量列表以及返回类型，但不规定方法主体。接口也包含了基本数据类型的数据成员，但它们都默认为static 和final。接口只提供一种形式，并不提供实施的细节。

可决定将一个接口中的方法声明明确定义为“public”。但即便不明确定义，它们也会默认为 public。所以在实现一个接口的时候，来自接口的方法必须定义成public。

注意：

* 接口方法默认public，接口字段默认 public static final
* `JDK7.0`之前 接口里的所有方法都没有方法体，即都是抽象方法；`JDK8.0`后接口可以有静态方法（加`static`），默认方法（加`default`关键字修饰），也就是说接口中可以有方法的具体实现
* 接口不能被实例化
* 接口中所有方法都是 public 方法和 abstract 方法，接口中抽象方法可以不用abstract修饰
* 一个普通类实现接口就必须将该接口的所有方法都实现
* 抽象类实现接口可以不用实现接口的方法
* 一个类可以同时实现多个接口
* 接口不能继承类，但可以继承别的接口
* 接口的修饰符只能是public和默认，和类一样



**接口和抽象类有什么共同点和区别**？

1. 抽象类主要用于代码复用，是一种is-a关系。接口仅仅是对方法的抽象，是一种 has-a 关系，表示具有某一组行为特性，是为了解决解耦问题，隔离接口和具体的实现，提高代码的扩展性。

2. 接⼝⽀持多继承，⼀个类可以实现多个接口，⼀个类只能继承⼀个抽象类。

3. 接⼝中的⽅法默认是 `public abstract` 的。接⼝中的变量默认是 `public static final` 的。 抽象类中的抽象⽅法默认是 `protected` 的，具体⽅法的访问修饰符可以是 public 、 protected 或 private 

4. 接口不含构造器，抽象类可以包含构造器



## 内部类

一个类的内部又完整的嵌套了另一个类结构，被嵌套的类称为内部类。内部类最大的特点是可以直接访问外部类的属性，并且可以直接体现类之间的包含关系。

实际上内部类是一个编译层面的概念，像一个语法糖一样，经过编译器之后其实内部类会提升为外部顶级类，和外部类没有区别。

### 局部内部类

类似局部变量，定义在外部类的局部位置，在方法、代码块中，有类名

* **不能添加访问修饰符，因为它的地位是一个局部变量**，局部变量是不能使用修饰符的。但可以使用final修饰
* 只能访问final变量和形参
* 作用域：仅仅在定义它的方法或代码块中
* 如果外部类和内部类重名时，默认遵守就近原则，如果想访问外部类的成员，则可以用`外部类名.this.成员`访问

### 匿名内部类

定义在外部类的局部位置，比如方法中，没有类名，同时还是一个对象

一个接口/类的方法的某个实现方式在程序中**只会执行一次**，但为了使用它，我们需要创建它的实现类/子类去实现重写。此时可以使用匿名内部类的方式，可以**无需创建新的类，减少代码冗余**。

* 匿名内部类既是一个类的定义，同时也是一个对象，只能创建匿名内部类的一个实例

* **没有构造器**，没有静态资源

* 可以访问外部类的所有成员，包括私有的
* **不能添加修饰符和static，因为它的地位就是一个局部变量**
* 作用域：仅仅是在定义它的方法或代码块中
* 外部其他类不能访问匿名类
* 如果外部类和内部类重名时，默认遵守就近原则，如果想访问外部类的成员，则可以用`外部类名.this.成员`访问

* 可以当作实参直接传递，简洁高效

### 成员内部类

* 定义在外部类的成员位置，并且**没有static修饰**
* 可以直接使用外部类的所有成员，包括私有成员
* **可以添加任意的访问修饰符，因为它的地位是一个成员**
* **本身内部不能有静态属性**，因为自己本身就需要依靠外部类的实例化
* 作用域：和外部类的其他成员一样，为整个类体
* 外部其他类访问内部类
  * `外部类名.内部类名 对象名 = 外部类名.new 内部类名();`
  * `外部类名.内部类名 对象名 = 外部对象.get();`
  * `new 外部类名().new 内部类();`

### 静态内部类

类似类的静态成员属性

* 定义在外部类的成员位置，**有static修饰**
* 可以**访问外部类所有静态成员**，不能访问非静态成员
* 可以添加任意修饰符
* 作用域：整个类体
* 外部其他类访问内部类
  * 通过类名直接访问，但要满足访问权限
  * get方法

* 如果外部类和内部类重名时，默认遵守就近原则，如果想访问外部类的成员，则可以用`外部类名.成员`访问



## 枚举类

实现：

1. 构造器私有化
2. 对外暴露对象 `public final static `
3. 提供get方法，不提供set方法

细节：

* 使用`enum`代替`class`声明枚举类，就不能继承其他类了（因为**隐式继承`Enum`类**，`java`是单继承机制），但是可以继承接口

* 使用`enum`关键字开发一个枚举类时，默认会继承`Enum`类, 而且是一个final类
* 枚举简化为`枚举对象(参数列表)`
* 使用无参构造器创建枚举对象，则实参和小括号都可以省略
* 当有多个枚举对象时，使用`,`分隔，最后一个`;`结尾
* 枚举对象必须放在枚举类首行

```java
enum Season 
{ 
    SPRING("1","1"),WINTER("2","2"); 
    private String name;
    private String desc;
    private Season(String name, String desc){}
} 
```



## 异常

![](.\images\异常.jpeg)

执行过程中发生的异常分为两大类

1. Error(错误)：Java虚拟机无法解决的严重问题，如`JVM`系统内部错误、资源耗尽等。比如`StackOverflowError`和`OOM（out of memory)`

2. Exception：其他因编程或偶然的外在因素导致的一般问题，可以使用针对性的代码处理，分为两大类：运行时异常和编译时异常。编译异常程序中必须处理，运行时异常程序中没有处理默认是`throws`
   1. 运行时异常：
      1. 空指针异常 NullPointerException
      2. 数学运算异常 ArithmeticException
      3. 数组下标越界异常 ArrayIndexOutOfBoundsException
      4. 类型转换异常 ClassCastException
      5. 数字格式不正确异常 NumberFormatException

   2. 编译时异常：
      1. 数据库操作异常 SQLException
      2. 文件操作异常 IOException
      3. 文件不存在 FileNotFoundException
      4. 类不存在 ClassNotFoundException
      5. 文件末尾发生异常 EOPException
      6. 参数异常 IllegalArguementException




`try-catch-finally`：捕获异常，自行处理

```java
try{
	// 
}catch(EXception e){
	// 当异常发生时异常后面的代码不执行，直接进入catch，系统将异常封装成Exception对象e传递给catch，得到异常后程序员自己处理
	// 异常没有发生，catch不执行
	// 在实际开发中，通常将编译异常转换为允许异常抛出，调用者可以捕获或抛出 throw new RuntimeException(e);
}finally{
	// 不管代码是否有异常，finally都要执行
	// 通常将释放资源的代码放在finally，保证关闭
}
```

可以有多个`catch`语句，捕获不同的异常，要求父类异常在后，子类异常在前，比如`Exception`在后，`NullPointException`在前，如果发生异常，只会匹配一个`catch`

可以进行`try-finally`配合使用，相当于没有捕获异常，因此程序会直接奔溃。应用场景：执行一段代码不管是否发生异常都必须执行某个业务逻辑

finally子句中包含return语句时肯产生意想不到的结果，finally中的return语句将会覆盖原来的返回值。不要把改变控制流的语句（return, throw, break, continue）放在finally子句中



`throws`：将异常抛出，交给调用者处理，最顶级的处理者是`JVM`

编译异常程序中必须处理，运行时异常程序中没有处理默认是`throws`

子类重写父类的方法时，对抛出异常的规定：子类重写的方法抛出的异常类型要么和父类抛出的异常一致，要么为父类抛出异常类型的子类型

在`throws`过程中，如果有`try-catch`，相当于处理异常，就可以不必`throws`



自定义异常：`异常类名 extends Exception/RuntimeException`如果继承`Exception`，属于编译异常；如果继承`RuntimeException`，属于运行异常，一般继承`RuntimeException`

|        | 意义                   | 位置       | 后面跟的东西 |
| ------ | ---------------------- | ---------- | ------------ |
| throws | 异常处理的一种方式     | 方法声明处 | 异常类型     |
| throw  | 手动生成异常对象关键字 | 方法体中   | 异常对象     |



try-with-resources语句：

```java
try (Resource res = ...) { // 可以指定多个资源, ';'间隔
    //work with res
}
```

当try退出时会自动调用 res.close()



**finally总是会被执⾏吗？**

⼀般来说， finally 块都会在 try 或 catch 块执⾏完毕后被执⾏，即使发⽣了异常。然⽽，有⼀些情况下 finally 块可能不会执⾏，主要是在以下情况： 

1. 在 try 或 catch 块中调⽤了 System.exit() ： 调⽤ System.exit() 会导致Java虚拟机（JVM）退出， 此时 finally 块中的代码不会被执⾏。 
2. 在 try 块中发⽣了死循环： 如果在 try 块中发⽣了⽆限循环或者其他永远不会结束的操作， finally 块 可能⽆法执⾏。
3. 程序所在线程死亡或关闭CPU



注意：

1. 尽量不要捕获类似Exception这样的异常，而是捕获特定的异常。
2. 不要吞了异常，最好输入到日志中
3. 不要延迟处理异常，否则堆栈信息会很多
4. try-catch范围尽可能小
5. 不要通过异常来控制程序流程
6. 不要在finally代码块中处理返回值或者直接return



## String、StringBuffer、StringBuilder

**String**

有属性`private final char value[]`, value赋值后不可以修改，指不能指向新的地址，但单个字符内容可以改变。在 Java 9 之后，`String`、`StringBuilder` 与 `StringBuffer` 的实现改用 `byte` 数组存储字符串，不同编码占用字节不同-为了节省空间。

`String` 真正不可变有下面几点原因：

1. 保存字符串的数组被 `final` 修饰且为私有的，并且`String` 类没有提供/暴露修改这个字符串的方法。
2. `String` 类被 `final` 修饰导致其不能被继承，进而避免了子类破坏 `String` 不可变。

`String` 中的 `equals` 方法是被重写过的，比较的是 String 字符串的值是否相等。 `Object` 的 `equals` 方法是比较的对象的内存地址。

**StringBuffer**

解决了String大量拼接字符串时产生许多无用的中间对象问题，提供append和add等方法，可以将字符串添加到已有序列的末尾或指定位置。

本质是一个线程安全的可修改的字符序列，把所有修改数据的方法都加上`synchronized` ，保证了线程安全。

**StringBuilder**

和StringBuffer本质上没什么区别，就是去掉了保证线程安全的那部分，减少了开销。

`StringBuilder` 与 `StringBuffer` 都继承自 `AbstractStringBuilder` 类，在 `AbstractStringBuilder` 中使用char数组保存字符串(JDK9以后是byte数组)，不过没有使用 `final` 和 `private` 关键字修饰，是**可变的**，



**字符串常量池**：

```java
String s1 = "str";
// 先从常量池查看是否有"str"的引用，如果有直接返回引用；如果没有则堆中创建然后将引用保存到常量池中。s1最终指向的是常量池的空间地址

String s2 = new String("str");
// 先在堆中创建对象，里面维护的value属性保存常量池"str"的引用。如果常量池有"str"引用，直接返回；如果没有则堆中创建然后将引用保存到常量池中。s2最终指向的是堆中的空间地址。
```

```java
String str1 = "str";
String str2 = "ing";
String str3 = "str" + "ing";//常量相加: 编译器会优化，常量池中只有"string", str3指向常量池中的"string"
String str4 = str1 + str2;//变量相加: str4指向堆再通过堆中value指向常量池中的"string", 常量池中有”str", "ing", "string"
String str5 = "string";
System.out.println(str3 == str4);//false
System.out.println(str3 == str5);//true
System.out.println(str4 == str5);//false

final String str1 = "str";
final String str2 = "ing";
// 下面两个表达式其实是等价的
String c = "str" + "ing";// 常量池中的对象
String d = str1 + str2; // 常量池中的对象
System.out.println(c == d);// true 字符串使用 final 关键字声明之后，可以让编译器当做常量来处理。
```



**String.intern()**

`String.intern()` 是一个 native（本地）方法，其作用是将指定的字符串对象的引用保存在字符串常量池中，可以简单分为两种情况：

1. 如果字符串常量池中保存了对应的字符串对象的引用，就直接返回该引用。

2. 如果字符串常量池中没有保存了对应的字符串对象的引用，那就在常量池中创建一个指向该字符串对象的引用并返回



总结：

* String（不可变，线程安全）
* StringBuffer（可变、线程安全）
* StringBuilder(可变、⾮线程安全）

操作少量的数据: 适用 `String`

单线程操作字符串缓冲区下操作大量数据: 适用 `StringBuilder`

多线程操作字符串缓冲区下操作大量数据: 适用 `StringBuffer`



## 泛型

泛型又称参数化类型，解决数据的安全性问题

* 编译时，检查添加元素的类型，提高了安全性

* 减少了类型转换的次数，提高效率

* 可以在类声明时通过一个标识表示类中某个属性的类型，或者是某个方法的返回值类型，或是参数类型

* 泛型指定数据类型时要求是引用类型，不能是基本数据类型

* 给泛型指定类型后，可以传入该类型或其子类类型

* 如果不指定泛型，默认是Object

* 泛型没有继承性 `List<Object> list = new ArrayList<String>();//错误`

  

**泛型类**

```java
class 类名<T,R...>{//泛型标识符可以有多个，一般单个大写字母
	// 
}
```

注意细节：

1. 普通成员可以使用泛型(属性, 方法)
2. 使用泛型的数组，不能初始化。例如不能：`T[] t = new T[8]`; //因为数组在new时不能确定T的类型无法在内存开辟空间。
3. 静态方法和属性中不能使用泛型。因为静态是和类相关的，类加载时对象还没有创建
4. 泛型类的类型，是在创建对象时确定的
5. 如果创建对象时没有指定类型，默认Object

**泛型接口**

```java
interface 接口名<T,R...>{}
```

注意细节：

1. 接口中静态成员也不能使用泛型
2. 泛型接口的类型，是在继承接口或实现接口时确定的
3. 没有指定泛型默认Object

**泛型方法**

```java
修饰符 <T,R..> 返回类型 方法名（参数列表）{}
```

注意细节：

1. 泛型方法可以定义在普通类中，也可以定义在泛型类中
2. 当泛型方法被调用时，类型会确定
3. public void eat(E e){}不是泛型方法，而是使用了泛型

**通配符**

```java
<?>: 支持任意泛型类型
<? extends A>: 支持A类和A类的子类，规定了泛型的上限
<? super A>: 支持A类和A类的父类，不限于直接父类，规定了下限
```



**泛型擦除：**

泛型是一个语法糖，Java虚拟机不认识泛型，泛型会在编译阶段通过泛型擦除的方式进行解语法糖.

擦除类型参数：

在泛型类或泛型方法中，所有的类型参数都会在编译时被擦除：

- 如果泛型类型参数没有指定上界，编译器会将其替换为`Object`。
- 如果泛型类型参数指定了上界（例如`<T extends Number>`），编译器会将其替换为上界类型（这里是`Number`）。



类型检查与类型转换：

虽然在运行时泛型类型信息被擦除，但在编译时，编译器会进行类型检查，以确保泛型的使用符合类型安全的要求。在编译后的字节码中，会插入类型转换代码，以确保获取到正确的类型：

```java
Box stringBox = new Box();
stringBox.set("Hello");
String item = (String) stringBox.get();
```



擦除方法的重载：

由于泛型类型在编译时被擦除，可能导致方法签名的冲突。例如：

```java
public class Example {
    public void method(List<String> list) {}
    public void method(List<Integer> list) {}
}
```

这两个方法在类型擦除后，其方法签名都变为`method(List list)`，从而引发编译错误。Java不允许在同一类中声明这些方法。



## 序列化和方序列化

序列化：将数据结构或对象转换成二进制字节流的过程

反序列化：将在序列化过程中所生成的二进制字节流转换成数据结构或者对象的过程

序列化的主要目的是通过网络传输对象或者说是将对象存储到文件系统、数据库、内存中。

比较常用的序列化协议有 Hessian、[Kryo]([EsotericSoftware/kryo: Java binary serialization and cloning: fast, efficient, automatic (github.com)](https://github.com/EsotericSoftware/kryo))、[Protobuf]([protocolbuffers/protobuf: Protocol Buffers - Google's data interchange format (github.com)](https://github.com/protocolbuffers/protobuf))、[ProtoStuff]([protocolbuffers/protobuf: Protocol Buffers - Google's data interchange format (github.com)](https://github.com/protocolbuffers/protobuf))，这些都是基于二进制的序列化协议。JDK 自带的序列化方式一般不会用 ，因为序列化效率低并且存在安全问题。Kryo 是专门针对 Java 语言序列化方式并且性能非常好

**JDK自带的序列化**

JDK 自带的序列化，只需实现 `java.io.Serializable`接口即可。



**serialVersionUID 有什么作用？**

```java
private static final long serialVersionUID = 1905122041950251207L;
```

序列化号 `serialVersionUID` 用来验证序列化和反序列化对象的ID是否一致的。反序列化时，会检查 `serialVersionUID` 是否和当前类的 `serialVersionUID` 一致。如果 `serialVersionUID` 不一致则会抛出 `InvalidClassException` 异常。强烈推荐每个序列化类都手动指定其 `serialVersionUID`，如果不手动指定，那么编译器会动态生成默认的 `serialVersionUID`。



**Java序列化不包含静态变量？**

Java序列化机制只会保存对象的实例变量的状态，而不会保存静态变量的状态。

静态变量是类级别的变量，它们被所有类的实例共享。序列化的主要目的是保存和恢复对象的实例状态，以便在不同的时间或不同的环境中能够重建对象。由于静态变量不是实例的一部分，它们的值在类加载时就已经初始化并存在，因此它们不需要被序列化和恢复。



**serialVersionUID 不是被 static 变量修饰了吗？为什么还会被“序列化”？**

`static` 修饰的变量是静态变量，本身是不会被序列化的。但是，`serialVersionUID` 的序列化做了特殊处理，在序列化时，会将 `serialVersionUID` 的值写入到序列化的二进制字节流中；在反序列化时，也会解析它并做一致性判断。



**如果有些字段不想进行序列化怎么办？**

对于不想进行序列化的变量，可以使用 `transient` 关键字修饰。

`transient` 关键字的作用是：阻止实例中那些用此关键字修饰的的变量序列化；当对象被反序列化时，被 `transient` 修饰的变量值不会被持久化和恢复。

关于 `transient` 还有几点注意：

- `transient` 只能修饰变量，不能修饰类和方法。
- `transient` 修饰的变量，在反序列化后变量值将会被置成类型的默认值。例如，如果是修饰 `int` 类型，那么反序列后结果就是 `0`。
- `static` 变量因为不属于任何对象(Object)，所以无论有没有 `transient` 关键字修饰，均不会被序列化。



## IO流

### 字节流

**InputStream（字节输入流）**：

* `FileInputStream` 是一个比较常用的字节输入流对象，可直接指定文件路径，可以直接读取单字节数据，也可以读取至字节数组中。不过，一般我们是不会直接单独使用 `FileInputStream` ，通常会配合 `BufferedInputStream`

* `DataInputStream` 用于读取指定类型数据，不能单独使用，必须结合其它流，比如 `FileInputStream` 。
* `ObjectInputStream` 用于从输入流中读取 Java 对象（反序列化），`ObjectOutputStream` 用于将对象写入到输出流(序列化)。用于序列化和反序列化的类必须实现 `Serializable` 接口，对象中如果有属性不想被序列化，使用 `transient` 修饰。

**OutputStream（字节输出流）**：

* `FileOutputStream` 是最常用的字节输出流对象，可直接指定文件路径，可以直接输出单字节数据，也可以输出指定的字节数组。类似于 `FileInputStream`，`FileOutputStream` 通常也会配合 `BufferedOutputStream`（字节缓冲输出流，后文会讲到）来使用。
* `DataOutputStream` 用于写入指定类型数据，不能单独使用，必须结合其它流，比如 `FileOutputStream` 。
* `ObjectInputStream` 用于从输入流中读取 Java 对象（`ObjectInputStream`,反序列化），`ObjectOutputStream`将对象写入到输出流(`ObjectOutputStream`，序列化)



### 字符流

**Reader（字符输入流）**：

`InputStreamReader` 是字节流转换为字符流的桥梁，其子类 `FileReader` 是基于该基础上的封装，可以直接操作字符文件。

**Writer（字符输出流）**：

`OutputStreamWriter` 是字符流转换为字节流的桥梁，其子类 `FileWriter` 是基于该基础上的封装，可以直接将字符写入到文件。

### 字节缓冲流

`BufferedInputStream` 从源头（通常是文件）读取数据（字节信息）到内存的过程中不会一个字节一个字节的读取，而是会先将读取到的字节存放在缓存区，并从内部缓冲区中单独读取字节。这样大幅减少了 IO 次数，提高了读取效率。

`BufferedInputStream` 内部维护了一个缓冲区，这个缓冲区实际就是一个字节数组。缓冲区的大小默认为 **8192** 字节



`BufferedOutputStream` 类似于 `BufferedInputStream`



### 字符缓冲流

`BufferedReader` （字符缓冲输入流）和 `BufferedWriter`（字符缓冲输出流）类似于 `BufferedInputStream`（字节缓冲输入流）和`BufferedOutputStream`（字节缓冲输入流），内部都维护了一个字节数组作为缓冲区。

### 打印流

`System.out` 实际是用于获取一个 `PrintStream` 对象，`print`方法实际调用的是 `PrintStream` 对象的 `write` 方法。

`PrintStream` 属于字节打印流，与之对应的是 `PrintWriter` （字符打印流）。`PrintStream` 是 `OutputStream` 的子类，`PrintWriter` 是 `Writer` 的子类。

### 随机访问流

随机访问流指的是支持随意跳转到文件的任意位置进行读写的 `RandomAccessFile` 。

`RandomAccessFile` 比较常见的一个应用就是实现大文件的 **断点续传**



### I/O模型

1. **BIO(Blocking I/O)**：读取或写⼊数据时，线程将⼀直等待，直到数据准备就绪或者写⼊操作完成， 但在⾼ 并发环境下可能导致性能问题，因为线程在等待 I/O 操作完成时被阻塞，⽆法执⾏其他任务。

2. **NIO(Non-blocking I/O)**: 在⾮阻塞 I/O 模型中，线程执⾏⼀个 I/O 操作时不会等待，⽽是继续执⾏其他任务, 这需要通过轮询（polling）或 者使⽤回调函数等机制来检查 I/O 操作是否完成。
3. **IO多路复⽤**：I/O 多路复⽤模型使⽤了操作系统提供的选择器（Selector）机制，例如 Java 中的 Selector 类。通过选择器，⼀ 个线程可以监听多个通道上的 I/O 事件，从⽽在单线程中处理多个连接。
4. **AIO（Asynchronous I/O）**：AIO 允许程序发起⼀个I/O操作，并在操作完成时得到通知。在这个过程中，程序可以继续执⾏其他任务⽽⽆需等 待I/O操作完成，当操作完成之后，进⾏回调。



## 反射机制

反射机制允许程序在执行期间借助于ReflectionAPI取得任何类的内部信息，并能操作对象的属性及方法。

加载完类之后，在堆中产生一个Class类型的对象（一个类只有一个Class对象），这个对象包含类的完整结构信息。

反射相关的主要类:

```java
java.lang.Class 			//代表一个类，Class对象表示某个类加载后在堆中的对象
java.lang.reflect.Method	//代表类方法
java.lang.reflect.Field		//代表类成员变量
java.lang.reflect.Constructor//代表类构造方法
```



**优缺点**

​	优点：可以动态的创建和使用对象（框架底层核心），使用灵活

​	缺点：使用反射是解释执行，对执行速度有影响，安全问题



**反射调用优化：关闭访问检查**

Method和Field、Constructor对象都有setAccessible()方法，作用是启动和禁用访问安全检查，参数为true表示反射的对象在使用时取消访问检查，提高反射效率



**获取 Class 对象的四种方式**

如果我们动态获取到这些信息，我们需要依靠 Class 对象。Class 类对象将一个类的方法、变量等信息告诉运行的程序。Java 提供了四种方式获取 Class 对象:

1. 知道具体类的情况下可以使用：

   ```java
   Class<Target> clazz = TargetObject.class;
   ```

   但是我们一般是不知道具体类的，基本都是通过遍历包下面的类来获取 Class 对象，通过此方式获取 Class 对象不会进行初始化

2. 通过 `Class.forName()`传入类的全路径获取：

   这种方式会触发类的初始化，静态代码块会被执行

   ```java
   Class<?> clazz = Class.forName("com.javaguide.TargetObject");
   ```

3. 通过对象实例`instance.getClass()`获取：

   ```java
   TargetObject o = new TargetObject();
   Class<?> clazz = o.getClass();
   ```

4. 通过类加载器`xxxClassLoader.loadClass()`传入类路径获取:

   ```java
   ClassLoader.getSystemClassLoader().loadClass("com.javaguide.TargetObject");
   ```

   通过类加载器获取 Class 对象不会进行初始化，意味着不进行包括初始化等一系列步骤，静态代码块和静态对象不会得到执行



**通过反射获取类结构信息**：

```java
getName:获取全类名
getSimpleName:获取简单类名
getFields:获取所有public属性，包含本类和父类
getDeclaredFields:获取所有属性
getMethods:获取所有public方法
getDeclaredMethods:获取所有方法
getModifiers:以int形式返回修饰符（默认修饰符是0，public是1，private是2，protected是4，static是8，final是16）
getType:以Class形式返回类型
...
```



**通过反射创建对象方式**

1. 获取类的构造器

   ```java
   Class<?> clazz = Class.forName("cn.javaguide.TargetObject");
   Constructor constructor = clazz.getDeclaredConstructor(参数列表.class);//f即为对应属性
   ```

2. 使用newInstance(): 调用构造器

   ```java
   Object obj = constructor.newInstance(object);
   ```



**通过反射访问类中成员**

1. 根据属性名获取Field对象  //getField获取public属性, getDeclaredField获取所有类型属性

   ```java
   Class<?> clazz = Class.forName("cn.javaguide.TargetObject");
   Field f = clazz.getDeclaredField(属性名);//f即为对应属性
   ```

2. 爆破：`f.setAccessible(true)`

3. 访问：

   ```
   f.set(targetObject, 值);//object为对象
   f.get(targetObject);
   //如果是静态属性，set和get中的参数object，可以写成null
   ```



**通过反射访问方法**

1. 根据方法名和参数列表获取Method方法对象

   ```java
   Class<?> clazz = Class.forName("cn.javaguide.TargetObject");
   Method m = clazz.getDeclaredMethod(方法名, 参数列表.class);
   ```

2. 获取对象 ：`Object targetObject = clazz.newInstance();`

3. 爆破：`m.setAccessible(true);` //私有的需要爆破

4. 访问：`Object returnValue = m.invoke(targetObject, 实参列表); //如果是静态方法，o可以写成null`



## 代理模式

### 静态代理

需要对每个目标类都单独写一个代理类，不灵活且麻烦

### 动态代理

动态代理是在运行时动态生成类字节码，并加载到 JVM 中

#### JDK动态代理

基于接口的，代理类一定是有定义的接口，在 Java 动态代理机制中 `InvocationHandler` 接口和 `Proxy` 类是核心。

`Proxy` 类中使用频率最高的方法是：`newProxyInstance()` ，这个方法主要用来生成一个代理对象。

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
    throws IllegalArgumentException
{
    ......
}
```

这个方法一共有 3 个参数：

1. **loader** :类加载器，用于加载代理对象。
2. **interfaces** : 被代理类实现的一些接口；
3. **h** : 实现了 `InvocationHandler` 接口的对象；

要实现动态代理的话，还必须需要实现`InvocationHandler` 来自定义处理逻辑。 当我们的动态代理对象调用一个方法时，这个方法的调用就会被转发到实现`InvocationHandler` 接口类的 `invoke` 方法来调用。

```java
public interface InvocationHandler {

    /**
     * 当使用代理对象调用方法的时候实际会调用到这个方法
     */
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

1. **proxy** :动态生成的代理类
2. **method** : 与代理类对象调用的方法相对应
3. **args** : 当前 method 方法的参数

**通过`Proxy` 类的 `newProxyInstance()` 创建的代理对象在调用方法的时候，实际会调用到实现`InvocationHandler` 接口的类的 `invoke()`方法。** 可以在 `invoke()` 方法中自定义处理逻辑，比如在方法执行前后做什么事情。



**JDK 动态代理类使用步骤:**

1. 定义一个接口及其实现类；
2. 自定义代理类实现 `InvocationHandler`接口并重写`invoke`方法，在 `invoke` 方法中我们会调用原生方法（被代理类的方法）并自定义一些处理逻辑；
3. 通过 `Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)` 方法创建代理对象；



#### CGLIB 动态代理

CGLIB基于ASM字节码生成工具，它通过继承的方式实现代理类，所以不需要接口，可以代理普通类，但需要注意 final 方法（不可继承）。

在 CGLIB 动态代理机制中 `MethodInterceptor` 接口和 `Enhancer` 类是核心。

你需要自定义 `MethodInterceptor` 并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法。

```java
public class ServiceMethodInterceptor implements MethodInterceptor{
    // 拦截被代理类中的方法
    @Override
    public Object intercept(Object obj, java.lang.reflect.Method method, Object[] args,MethodProxy proxy) throws Throwable {
        
    }
}
```

1. **obj** : 被代理的对象（需要增强的对象）
2. **method** : 被拦截的方法（需要增强的方法）
3. **args** : 方法入参
4. **proxy** : 用于调用原始方法

可以通过 `Enhancer`类来动态获取被代理类，当代理类调用方法的时候，实际调用的是 `MethodInterceptor` 中的 `intercept` 方法。

```java
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(SampleClass.class);
enhancer.setCallback(new ServiceMethodInterceptor());

SampleClass proxy = (SampleClass) enhancer.create();
proxy.test();
```



**CGLIB 动态代理类使用步骤：**

1. 引入依赖
2. 定义类

3. 自定义 `MethodInterceptor` 并重写 `intercept` 方法，`intercept` 用于拦截增强被代理类的方法，和 JDK 动态代理中的 `invoke` 方法类似；

4. 通过 `Enhancer` 类的 `create()`创建代理类；



## 注解

Java中的注解（Annotation）是一种标记，是一种用于为代码元素（如类、方法、字段等）添加元数据的机制。这些元数据可以在编译时、类加载时或运行时被读取，并用于各种目的，如代码生成、运行时行为控制、配置管理等。

注解可以通过不同的保留策略来决定其在何时可见：

- `RetentionPolicy.SOURCE`：注解仅存在于源代码中，在编译成字节码后丢弃。
- `RetentionPolicy.CLASS`：注解保留在编译后的字节码中，但在运行时不可见（默认）。
- `RetentionPolicy.RUNTIME`：注解保留在字节码中，并且在运行时可通过反射访问。



定义注解：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MyAnnotation {
    String value();
    int count() default 1;
}
```

**`@Retention`**：指定注解的保留策略。上例中，注解在运行时可见。

**`@Target`**：指定注解可以应用的代码元素。上例中，注解只能用于方法。

**属性**：注解的属性类似于无参方法，可以有默认值（如`count`），也可以不设置默认值（如`value`）。



获取注解：

```java
public class Test {
    
    @MyAnnotation(value = "Hello", count = 3)
    public void annotatedMethod() {
        // method logic
    }
}


public class AnnotationProcessor {
    public static void main(String[] args) throws Exception {
        Method method = Test.class.getMethod("annotatedMethod");
        
        if (method.isAnnotationPresent(MyAnnotation.class)) {
            MyAnnotation annotation = method.getAnnotation(MyAnnotation.class);
            System.out.println("Value: " + annotation.value());
            System.out.println("Count: " + annotation.count());
        }
    }
}
```



## Class类

Class类也是类，因此继承Object类

Class类对象不是new出来的，而是系统创建的

对于某个类的Class对象，在内存只有一份，因为类只加载一次

每个类的实例都会记得自己是由哪个Class实例所产生

通过Class对象可以完整的得到一个类的完整结构，通过一系列的API

Class对象存放在堆中

类的字节码二进制数据，是放在方法区的，有的地方称为类的元数据（包括方法代码、变量名、方法名、访问权限等）



**获取Class对象的方法 :**

```java
// 1. 已知一个类的全类名，且在该类的路径下，可以通过Class类的静态方法forName()获取。
//	  应用场景：多用于读取配置文件，读取类全路径加载类
Class cls = Class.forName("java.lang.Cat");

// 2. 已知具体类，通过类的class获取，该方式最为安全可靠，性能最高
//	  多用于参数传递，比如通过反射得到对应构造器对象
Class cls = Cat.class;

// 3. 已知某个类的实例，调用该实例的getClass()方法获取Class对象
//	  通过创建好的对象获取Class对象
Class cls = 对象.getClass();

// 4. 通过类加载器得到Class对象
//
Car car = new Car();
// 1) 先得到类加载器
ClassLoader classLoader = car.getClass().getClassLoader();
// 2) 通过类加载器得到Class对象
Class aclass = classLoader.loadClass(classAllPath);

```



## SPI机制

SPI（Service Provider Interface）是Java提供的一种服务发现机制，它允许框架或应用程序动态地加载服务的实现。在Java生态系统中，SPI常用于开发可扩展、可插拔的框架和库，例如Java中的`JDBC`、`JAXP`、`JNDI`等。

SPI和API区别：

API是实现方提供接口和实现，我们可以通过调用实现方的接口从而拥有实现方给我们提供的能力 ，这种接口和实现都是放在实现方的。

SPI是接口存在于调用方这边 ，由接口调用方确定接口规则，然后由不同的厂商去根据这个规则对这个接口进行实现，从而提供服务。



SPI 机制的具体实现本质上还是通过反射完成的。即：**我们按照规定将要暴露对外使用的具体实现类在 `META-INF/services/` 文件下声明。**



使用步骤：

1. 定义服务接口

   ```java
   public interface MyService {
       void execute();
   }
   ```

2. 提供服务实现

   ```java
   public class MyServiceImpl implements MyService {
       @Override
       public void execute() {
           System.out.println("Executing MyServiceImpl");
       }
   }
   ```

3. 创建服务配置文件：在服务实现的JAR包中，创建`META-INF/services`目录，并在其中添加一个配置文件，文件名为服务接口的全限定名，内容为服务实现类的全限定名：

   ```bash
   META-INF/services/com.example.MyService
   ```

   文件内容：

   ```
   com.example.MyServiceImpl
   ```

4. 加载和使用

   ```java
   import java.util.ServiceLoader;
   
   public class ServiceLoaderExample {
       public static void main(String[] args) {
           ServiceLoader<MyService> serviceLoader = ServiceLoader.load(MyService.class);
   
           for (MyService service : serviceLoader) {
               service.execute();
           }
       }
   }
   ```

   

## 语法糖

语法糖的存在主要是方便开发人员使用。 Java 虚拟机并不支持这些语法糖，这些语法糖在编译阶段就会被还原成简单的基础语法结构，这个过程就是解语法糖。

Java 中最常用的语法糖主要有：

switch支持String与枚举：Java 中的`switch`自身原本就支持基本类型，字符串的 switch 是通过`equals()`和`hashCode()`方法来实现的

泛型：所有泛型类的类型参数在编译时都会被擦除

变长参数：数组

条件编译：根据 if 判断条件的真假，编译器直接把分支为 false 的代码块消除。

自动拆装箱：装箱过程是通过调用包装器的 valueOf 方法实现的，而拆箱过程是通过调用包装器的 xxxValue 方法实现的

内部类：编译后会生成两个不同class文件

枚举：当我们使用`enum`来定义一个枚举类型的时候，编译器会自动帮我们创建一个`final`类型的类继承`Enum`类，所以枚举类型不能被继承

断言：断言的底层实现就是 if 语言

数值字面量：编译器并不认识在数字字面量中的`_`，需要在编译阶段把他去掉

for-each：实现原理就是普通for循环和迭代器

try-with-resource：编译器帮助关闭资源

Lambda表达式：lambda 表达式的实现其实是依赖了一些底层的 api，在编译阶段，编译器会把 lambda 表达式进行解糖，转换成调用内部 api 的方式。



## 函数式编程

代码简洁，开发快速，易于理解，易于并发编程

### Lambda表达式

只关注参数列表和方法体

```
(参数列表) -> {代码}
```

* 参数类型可以省略
* 方法体中只有一句代码时，大括号、return和代码分号可以省略
* 方法只有一个参数时，小括号可以省略

### Stream流

用来对集合或数组进行链状流式的操作

**1. 创建流**

* 单列集合(Collection)：`集合对象.stream()`
* 数组：`Arrays.stream(数组)` 或 `Stream.of(数组)`
* 双列集合(Map)：转换为单列集合后再创建 `map.entrySet().stream()`
* 使用`Stream.builder()` 创建流，然后使用add方法添加元素，最后使用build方法构建流。使用build后不能再add元素

**2. 中间操作**：筛选、映射、排序

* filter：对流中的元素进行条件过滤，符合过滤条件的才能继续留在流中

* map：对流中的元素进行计算或转换

* distinct：去除流中的重复元素。**依赖Object的equals和hashCode方法判断是否是相同对象**

* sorted：对流中的元素进行排序

* limit：可以设置流的最大长度，超出的部分将被舍弃

* skip：跳过流中的前n个元素，返回剩下的元素

* flatMap：map适合单层结构的流，进行一对一转换。flatMap不仅能够执行map的转换功能，还能扁平化多层数据结构，将其转换合并为一个单层流。

  ```java
  // 打印书籍分类
  author.stream() // 创建流
      .flatMap(author -> author.getBooks().stream()) // 获取书籍的流
      .distinct()
      .flatMap(book -> Arrays.stream(book.getCategory().split(","))) // 书籍分类得到数组，再获取数组流 
      .distinct()
      .forEach(category -> System.out.println(category));
  ```

**3. 终结操作**：查找与匹配、聚合操作、规约操作、收集操作、迭代操作。终结操作会强制执行中间的惰性操作

* 迭代操作：

  * forEach：对流中元素进行任意顺序遍历操作，通过传入参数指定遍历到的元素的具体操作


  * forEachOrdered：按照流中的顺序遍历元素


* 聚合操作：规约操作的特殊形式

  * count：获取当前流中元素个数

  * max，min：获取最值

  * average

  * sum

* 收集操作：collect：把当前流转换为一个集合，使用Collectors工具类

  * List：`Collectors.toList()`
  * Set：`Collectors.toSet()`
  * Map：`Collectors.toMap()`
  * 连接操作收集流中所有字符：`Collectors.joining()`
  * 统计总和、平均值、数量、最值：`Collectors.summarizing(Int|Long|Double)`
  * 分组：`Collectors.grouopingBy()`，分类函数是断言函数时（返回boolean的函数）`partitioningBy` 更高效

* 查找与匹配操作：短路操作，匹配一个后就会返回

  * anyMatch：判断是否有任意符合匹配条件的元素，结果为boolean类型

  * allMatch：判断是否都符合匹配条件，结果为boolean

  * noneMatch：判断是否都不符合匹配条件，结果为boolean

  * findAny：获取流中的任意一个元素。无法保证一定获取流中的第一个元素。并行处理流时很有效

  * findFirst：获取流中第一个元素

* 规约操作：reduce：对流中的数据按照自己定制的计算方式计算出一个结果

  * reduce的作用是把stream中的元素给组合起来，我们可以传入一个初始值，它会按照我们定制的计算方式依次拿流中的元素进行计算，结果继续与后面元素计算

    ```java
    values.stream().reduce(0, (x, y) -> x+y); //第一个元素为幺元值，流为空则返回幺元值
    ```

  * 内部的计算方式

    ```java
    T result = identity;// identity为传入的参数
    for(T element : this stream)
        result = accumulator.apply(result, element)
    return result;
    ```

  * 要使用并行流，则操作必须是可结合的。且要提供第二个函数来合并处理

**注意事项：**

* 惰性求值：如果没有终结操作，中间操作是不会执行的
* 流是一次性的：一个流对象经过终结操作后，这个流就不能再被使用
* 不会影响原数据

### Optional

可以写出更优雅的代码，避免空指针异常

**创建对象**：`Optional.of(对象) `对象不能为空； `Optional.ofNullable(对象)`，无论传入参数是否为null都不会出问题

**安全消费值**：`ifPresent()`接受一个函数，如果可选值存在就传递给该函数，否则不发生任何事情；`ifPresentOrElse`() 接收两个函数

**获取值**：`get()` ，获取数据为空时，会空指针异常

**安全获取值：**

* `orElse()`，是否为空都会执行括号里的内容

* `orElseGet()`，数据不为空返回数据，数据为空则根据传入的默认值参数返回，为空才执行括号里的内容
* `orElseThrow()`，数据不为空返回数据，数据为空根据传入的参数抛出异常

**过滤：**`filter()`，数据不符合要求，返回value为null的Optional对象

**判断：**`isPresent()`，更推荐使用`ifPresent()`

**数据转换：**`map()`

### 基本类型流

用来直接存储基本类型值，无需使用包装类型

存储short、char、byte、boolean可以使用IntStream；float使用DoubleStream。

对象流可以转换为基本类型流：mapToInt、mapToLong、mapToDouble；基本类型流转换为对象流需要使用boxed方法

### 函数式接口

接口中只有一个抽象方法

JDK函数式接口都加上了`@FunctionalInterface`注解进行标识，但无论是否有该注解，只要接口中只有一个抽象方法都是函数式接口

**常用的默认方法**

* and：相当于&&来拼接判断条件
* or：||
* negate：！，取反

### 方法引用

```
类名或对象名::方法名
```

使用lambda表达式时，如果方法体中只有一个方法的调用的话（包括构造方法），可以使用方法引用进一步简化代码

**引用类的静态方法**

```
类名::方法名
```

重写方法的时候，方法体中只有一行代码，这行代码是调用了某个类的静态方法，并且重写的抽象方法中所有参数按照顺序传入了调用的静态方法中

**引用对象的实例方法**

```
对象名::方法名
```

重写方法的时候，方法体中只有一行代码，这行代码是调用了某个对象的成员方法，并且重写的抽象方法中所有参数按照顺序传入了调用的成员方法中

**引用类的实例方法**

```
类名::方法名
```

重写方法的时候，方法体中只有一行代码，这行代码是调用了第一个参数的成员方法，并且重写的抽象方法中剩余的所有参数按照顺序传入了调用的成员方法中

**构造器引用**

```
类名::new
```

重写方法的时候，方法体中只有一行代码，这行代码是调用了某个类的构造器，并且重写的抽象方法中所有参数按照顺序传入了调用的构造方法中

### 并行流

`parallel()` ：将任意的顺序流转换为并行流

`parallerStream()`：从集合中获取一个并行流



## 正则表达式

```java
String content = "";

// 1. \\d表示任意一个数字
String regexp = "\\d\\d\\d\\d";

// 2. 创建模式对象 Pattern.CASE_INSENSITIVE表示忽略大小写，标志可以设置一个或多个
Pattern pattern = Pattern.compile(regexp,Pattern.CASE_INSENSITIVE);

// 3. 创建匹配器
Matcher matcher = pattern.matcher(content);

// 4. 开始匹配
/**
* matcher.find() 完成的任务
* 1. 根据指定规则，定位满足规则的子字符串
* 2. 找到后将子字符串的开始索引记录到 matcher 对象的属性 int[] groups;
*    groups[0]记录开始索引, 把结束索引+1的值记录到 groups[1]中
* 3. 同时记录 oldLast 的值为：子字符串结束索引+1的值. 下次执行 find() 从此开始匹配
*/
/**
* 考虑分组 正则表达式中含小括号即表示分组: String regexp = "(\\d\\d)(\\d\\d)";
* groups[0]记录开始索引, 把结束索引+1的值记录到 groups[1]中
* groups[2]记录第一组开始索引, groups[3]记录第一组结束索引+1的值
* 依次类推...
*/
/**
* public String group(int group) {
*    checkMatch();
*    checkGroup(group);
*    if ((groups[group*2] == -1) || (groups[group*2+1] == -1))
*        return null;
*    return getSubSequence(groups[group * 2], groups[group * 2 + 1]).toString();
* }
*/
while (matcher.find()){
	System.out.println(matcher.group(0));
}
```



```
转义：\\	在java的正则表达式中\\代表其他语言中的一个\
		  需要用到转义符号的字符：., *, +, (, ), /, \, ?, [, ], ^, {, }	
```

**匹配符**

| 符号           | 意义                                                         | 示例               | 解释                                                   |
| -------------- | ------------------------------------------------------------ | ------------------ | ------------------------------------------------------ |
| []             | 可接收的字符列表                                             | [efgh]             | efgh中的任意字符                                       |
| [^]            | 不可接收的字符列表                                           | [^abc]             | abc除外的任意字符                                      |
| -              | 连字符                                                       | A-Z                | 任意大写字母                                           |
| .              | 匹配除\n之外的任何字符                                       | a..b               | a开头b结尾中间任意两个字符的字符串                     |
| \\\d           | 匹配单个数字字符，相当于[0-9]                                | \\\d{3}(\\\d)?     | 包含3个或4个数字的字符串                               |
| \\\D           | 匹配单个非数字字符                                           | \\\D(\\\d)*        | 以单个非数字字符开头，后接任意个数字字符的字符串       |
| \\\w           | 匹配单个数字、大小写字母字符、下划线<br>相当于[0-9a-zA-Z]    | \\\d{3}\\\w{4}     | 以3个数字字符开头长度为7的数字字符串                   |
| \\\W           | 匹配单个非数字、大小写字母字符、下划线<br/>相当于[ ^0-9a-zA-Z] | \\\W+\\\d{2}       | 以至少1个非数字字母字符开头，2个数字字符结尾的字符串   |
| \\\s           | 匹配任何空白字符（空格、制表符等）                           |                    |                                                        |
| \\\S           | 匹配任何非空白字符                                           |                    |                                                        |
| **选择匹配符** |                                                              |                    |                                                        |
| \|             | 选择匹配符,匹配之前或之后的                                  | ab\|cd             | ab或cd                                                 |
| **限定符**     |                                                              |                    |                                                        |
| *              | 指定字符重复0次或n次                                         | (abc)*             | 包含任意个abc的字符串                                  |
| +              | 指定字符重复1次或n次                                         | m+(abc)*           | 以至少1个m开头，后接任意个abc的字符串                  |
| ？             | 指定字符重复0次或1次                                         | m+abc?             | 以m开头，后接ab或abc的字符串                           |
| {n}            | 只能输入n个字符                                              | [abcd]{3}          | 由abcd中字母组成的任意长度为3的字符串                  |
| {n, }          | 指定至少n个匹配                                              | [abcd]{3, }        | 由abcd中字母组成的任意长度至少为3的字符串              |
| {n, m}         | 指定至少n个但不多于m个匹配，尽可能匹配多的（贪婪匹配）       | [abcd]{3, 5}       | 由abcd中字母组成的任意长度至少为3但不大于5的字符串     |
| ？             | 非贪婪匹配，限定符后加 ？                                    | \\\d+?             | 至少一个数字字符的字符串，尽可能匹配少的               |
| **定位符**     |                                                              |                    |                                                        |
| ^              | 指定起始字符                                                 | ^[0-9]+[a-z]*      | 以至少1个数字开头，后接任意个小写字母的字符串          |
| $              | 指定结束字符                                                 | ^[0-9]\\\\-[a-z]+$ | 以1个数字开头后接'-'，并以至少一个小写字母结尾的字符串 |
| \\\b           | 匹配目标字符串的边界                                         | han\\\b            | 边界指子串间有空格或是目标字符串的结束位置             |
| \\\B           | 匹配目标字符串的非边界                                       | han\\\B            |                                                        |



**java正则表达式默认区分大小写，实现不区分大小写：**

```
// 1.
(?i)abc		表示abc都不区分大小写
a(?i)bc		表示bc不区分大小写
a((?i)b)c	表示b不区分大小写

// 2.
Pattern pattern = Pattern.compile(regexp,Pattern.CASE_INSENSITIVE);
```



**分组**

| 常用分组构造形式   | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| (pattern)          | 非命名捕获。捕获匹配的子字符串，编号为0的第一个捕获是整个正则表达式模式匹配的文本，其他捕获结果根据左括号的顺序从1开始自动编号 |
| (?\<name\>pattern) | 命名捕获。将匹配子字符串捕获到一个组名称过编号名称中。用于name的字符串不能包含任何标点符号，并且不能以数字开头，可以使用单引号替代尖括号，例如 ?'name' |
| (?:pattern)        | 匹配但是不捕获子字符串，是一个非捕获匹配，不存储供以后使用的匹配。这对于用or字符(\|)组合模式部件的情况很有用。例如 'industr(?:y\|ies)' 是比 'industry\|industries' 更经济的表达式 |
| (?=pattern)        | 非捕获匹配。例如，'Windows (?=95\|98\|NT\|2000)' 匹配 "Windows 2000" 中的 "Windows"，但是不匹配 "Windows 3.1" 中的 "Windows" |
| (?!pattern)        | 非捕获匹配。例如，'Windows (!=95\|98\|NT\|2000' 匹配 "Windows 3.1" 中的"Windows", 但是不匹配"Windows 2000" 中的"Windows" |



**反向引用**

圆括号的内容被捕获后，可以在这个括号后被使用，从而写出一个比较实用的匹配模式，这个我们称为反向引用，这种引用既可以是正则表达式内部，也可以是在正则表达式外部，内部反向引用`\\分组号`，外部反向引用`$分组号` 
