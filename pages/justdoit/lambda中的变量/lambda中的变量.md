# 谈谈lambda中的变量

- lambda的引入简化了编程，减少了代码量，延长了键盘的寿命...

- 但是lambda也有一些限制，例如当我们试图在lambda表达式中修改一个外部变量的时，将得到`variable used in lambda expression should be final or effectively final`这个错误，这是为什么呢？IDEA会智能提示我们使用数组或者原子类可以避免这个错误，这又是为什么呢？以前我都是直接按照IDEA的智能提示修改了，也没细究，现在认真思考一下~

  ![img1](./img1.jpg)

- 如下面的代码所示，在代码中统计了一个List中"apple"的个数，这样的代码自然是没法通过编译的。

  ```java
  public void test() {
      List<String> fruits = Arrays.asList("apple", "banana", "apple", "watermelon", "grape");
      int count = 0;
      fruits.forEach(f -> {
          if ("apple".equals(f)) {
              count++;
          }
      });
      System.out.println("the total number of apple is " + count);
  }
  ```

- 当然，上面的代码可以由以下代码代替

```java
    public void test() {
        List<String> fruits = Arrays.asList("apple", "banana", "apple", "watermelon", "grape");
        int count = (int) fruits.stream().filter("apple"::equals).count();
        System.out.println("the total number of apple is " + count);
    }
```

# lambda的实现原理

- 首先我们定义一个函数式接口

```java
@FunctionalInterface
public interface MyFunctionInterface {
    void sayHello(String name);
}
```

- 在`FunctionInterfaceDemo`中使用lambda表达式。我们先设想一下lambda是怎么实现的呢？是新生成了一个类吗，不然为什么要搞一个函数式接口呢？我们加上JVM参数`-Djdk.internal.lambda.dumpProxyClasses`，这样代码运行时产生的中间类将会被保存下来。

```java
public class FunctionInterfaceDemo {
    public void welcome(String name, MyFunctionInterface fi) {
        fi.sayHello(name);
    }

    public static void main(String[] args) {
        long now = System.currentTimeMillis();
        if (now % 2 == 0) {
            new FunctionInterfaceDemo().welcome("equator", (name) -> System.out.println("welcome, " + name));
        } else {
            new FunctionInterfaceDemo().welcome("leo", (name) -> System.out.println("hi, " + name));
        }
    }
}
```

- 运行**一次**上面的代码之后，我们可以发现产生了**一个**中间类`FunctionInterfaceDemo$$Lambda$1`，它实现了我们的函数式接口`MyFunctionInterface`，并在实现的方法中调用了`FunctionInterfaceDemo`类的`lambda$main$0`方法，但是我们并没有写过这样的方法呀，应该是自动生成的方法。

```java
final class FunctionInterfaceDemo$$Lambda$1 implements MyFunctionInterface {
    private FunctionInterfaceDemo$$Lambda$1() {
    }

    @Hidden
    public void sayHello(String var1) {
        FunctionInterfaceDemo.lambda$main$0(var1);
    }
}
```

- 使用命令`javap -p`反编译`FunctionInterfaceDemo.class`文件，可以看到的确自动生成了两个私有的静态方法（**方法是私有的，但不一定是静态的。如果在静态方法中使用lambda表达式，会生成静态的方法；反之则是非静态方法，this指向使用lambda表达式那个类的对象实例**）

```
Compiled from "FunctionInterfaceDemo.java"
public class com.equator.lambda.FunctionInterfaceDemo {
  public com.equator.lambda.FunctionInterfaceDemo();
  public void welcome(java.lang.String, com.equator.lambda.MyFunctionInterface);
  public static void main(java.lang.String[]);
  private static void lambda$main$1(java.lang.String);
  private static void lambda$main$0(java.lang.String);
}
```

- 可以看到自动生成的方法的逻辑正是我们的lambda表达式的逻辑

```
private static void lambda$main$0(java.lang.String);
    Code:
       0: getstatic     #13                 // Field java/lang/System.out:Ljava/io/PrintStream;
       3: new           #14                 // class java/lang/StringBuilder
       6: dup
       7: invokespecial #15                 // Method java/lang/StringBuilder."<init>":()V
      10: ldc           #20                 // String welcome,
      12: invokevirtual #17                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      15: aload_0
      16: invokevirtual #17                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: invokevirtual #18                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      22: invokevirtual #19                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      25: return
```

- 现在我们知道了lambda的实现原理是编译时编译器会在使用到lambda的类中自动生成私有的方法，然后在运行期间JVM会生成实现了函数式接口的内部类，在实现的方法中调用了前面生成的私有方法（执行了lambda表达式的逻辑）。具体生成内部类的方法在`java.lang.invoke.LambdaMetafactory`中。

```
    public static CallSite metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType)
            throws LambdaConversionException {
        AbstractValidatingLambdaMetafactory mf;
        mf = new InnerClassLambdaMetafactory(caller, invokedType,
                                             invokedName, samMethodType,
                                             implMethod, instantiatedMethodType,
                                             false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
        mf.validateMetafactoryArgs();
        return mf.buildCallSite();
    }
```

# 使用了外部变量的lambda表达式

> 我们得知了lambda的实现原理，那么为什么在lambda中的使用到的外部变量需要是final或者是effectively final的呢？
>
> 注：final指的是显式地声明final，effectively final指的是没有显式声明final，但是不对这个变量进行修改

- 我们再定义一个类`VariableUseIntTest`，加上`-Djdk.internal.lambda.dumpProxyClasses`参数，然后运行

```java
public class VariableUseIntTest {
    public static void intro(String name, Consumer consumer) {
        consumer.accept(name);
    }

    public static void main(String[] args) {
        int age = 22;
        intro("leo", (name) -> {
            System.out.println(String.format("My name is %s. I am %s years old.", name, age));
        });
    }
}
```

- 可以看到生成的内部类如下

```java
// $FF: synthetic class
final class VariableUseIntTest$$Lambda$1 implements Consumer {
    private final int arg$1;

    private VariableUseIntTest$$Lambda$1(int var1) {
        this.arg$1 = var1;
    }

    private static Consumer get$Lambda(int var0) {
        return new VariableUseIntTest$$Lambda$1(var0);
    }

    @Hidden
    public void accept(Object var1) {
    	// 这里的var1就是name咯，VariableUseIntTest$$Lambda$1的构造函数中var1就是外部的int类型的age变量传进来的
        VariableUseIntTest.lambda$main$0(this.arg$1, var1);
    }
}
```

- 也就是说，lambda在使用外部变量的时候，走的是Java中方法传参的途径来捕获外部变量的，在方法内部修改一个基本类型这个操作对方法外部不可见，所以final的语义算是对Java程序员的一种提醒与要求~
- Java中方法传递参数都是值传递，会拷贝一个副本到栈上。

# 解决方法

> 下面的2、3、4几个方法其实都是同一个原理：对象实例在堆上分配。

1. 使用静态变量：静态变量保存在方法区，也是线程共享的内存区域
2. 使用类来承载变量
3. 利用数组绕过检查
4. 使用原子类（和线程安全没有关系哟，只是将AtomicInteger作为基本类型int变量的容器，这种方法可能是4种方法中最好的一个吧，数组不太美观，也不用自己创建一个容器类）

# invokedynamic指令

- 我们再来看看这段代码：这里运行一次的话，由于now要么是奇数要么是偶数，所以lambda实际上只会执行一次。所以中间类只有一个，因为lambda的内部类是运行时生成的；但是私有方法生成了两个，所以这个是编译时就生成的方法。（我们也可以只编译不运行来验证这个说法）

```java
public class FunctionInterfaceDemo {
    public void welcome(String name, MyFunctionInterface fi) {
        fi.sayHello(name);
    }

    public static void main(String[] args) {
        long now = System.currentTimeMillis();
        if (now % 2 == 0) {
            new FunctionInterfaceDemo().welcome("equator", (name) -> System.out.println("welcome, " + name));
        } else {
            new FunctionInterfaceDemo().welcome("leo", (name) -> System.out.println("hi, " + name));
        }
    }
}

// 自动生成了两个方法
public class com.equator.lambda.FunctionInterfaceDemo {
  public com.equator.lambda.FunctionInterfaceDemo();
  public void welcome(java.lang.String, com.equator.lambda.MyFunctionInterface);
  public static void main(java.lang.String[]);
  private static void lambda$main$1(java.lang.String);
  private static void lambda$main$0(java.lang.String);
}
```

- 可以看到在main方法中lambda表达式的调用使用到了invokedynamic指令，这里面有什么说法吗？

```java
public static void main(java.lang.String[]);
    Code:
       0: invokestatic  #3                  // Method java/lang/System.currentTimeMillis:()J
       3: lstore_1
       4: lload_1
       5: ldc2_w        #4                  // long 2l
       8: lrem
       9: lconst_0
      10: lcmp
      11: ifne          34
      14: new           #6                  // class com/equator/lambda/FunctionInterfaceDemo
      17: dup
      18: invokespecial #7                  // Method "<init>":()V
      21: ldc           #8                  // String equator
      23: invokedynamic #9,  0              // InvokeDynamic #0:sayHello:()Lcom/equator/lambda/MyFunctionInterface;
      28: invokevirtual #10                 // Method welcome:(Ljava/lang/String;Lcom/equator/lambda/MyFunctionInterface;)V
      31: goto          51
      34: new           #6                  // class com/equator/lambda/FunctionInterfaceDemo
      37: dup
      38: invokespecial #7                  // Method "<init>":()V
      41: ldc           #11                 // String leo
      43: invokedynamic #12,  0             // InvokeDynamic #1:sayHello:()Lcom/equator/lambda/MyFunctionInterface;
      48: invokevirtual #10                 // Method welcome:(Ljava/lang/String;Lcom/equator/lambda/MyFunctionInterface;)V
      51: return
```

