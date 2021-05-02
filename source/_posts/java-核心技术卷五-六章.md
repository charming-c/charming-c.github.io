---
title: 'java-核心技术卷五,六章'
date: 2020-11-14 13:28:35
tags:
	[核心技术]
categories: Java
---



# Java 讲解--核心技术第五章，第六章

## 第五章继承

### 一. 父类和子类，继承思想

> 利用继承，人们可以基于已经存在的类构造一个新类。继承已经存在的类就是复用（继承）这些类的方法和域。在此基础上，还可以添加一些新的方法和域，以满足新的需求。

java中继承是一个非常基础并且非常重要的内容，对于理解面向对象的思想非常重要

<!--more-->

比如我们定义一个Employee类：

```java
public class Employee {

    private String name;
    private double salary;
    private LocalDate hireDay;


    //构造方法，用于初始化类的对象
    public Employee(String name, double salary, int year, int month, int day) {
        this.name = name;
        this.salary = salary;
        this.hireDay = LocalDate.of(year, month, day);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        this.salary = salary;
    }

    public LocalDate getHireDay() {
        return hireDay;
    }

    public void setHireDay(LocalDate hireDay) {
        this.hireDay = hireDay;
    }

    public void raiseSalary(double byPercent) {
        double raise = salary * byPercent / 100;
        salary += raise;
    }

}

```

上面的类就规范了一个Employee（职员）具有的性质，比如姓名，薪水，雇佣日期。但是相比于普通（Employee）职员，还有一类特殊的人（Manager），他们除了拥有Employee的属性和方法，还有一些特殊的福利，比如他们有奖金（bonus），那我们写Manager类时，还要把上面的代码再写一遍，这样会导致代码出现大量的重复，而java针对这样的情形，就提供了继承的思路：

```java
public class Manager extends Employee {

    public Manager(String name, double salary, int year, int month, int day) {
        super(name, salary, year, month, day);
    }

}

```

上面几行代码就使得Manager拥有了Employee的属性和方法，同时构造器也覆盖了Employee的构造器，但要注意的是，

* **对于父类的（私有的）private修饰的属性和方法，子类是不可见的，子类如果要访问私有成员的话，只能调用具有更高访问权限的父类的方法去访问**
* 子类的方法不适用于父类的对象，而父类的方法子类是可适用于子类的
* 对于某些方法，子类可能与父类的有冲突，我们可以在子类利用通过重写同名方法将父类的方法覆盖掉，这样对于子类对象调用该方法的时候，会默认调用子类的方法
* super关键字：就是让编译器去这个类的父类class文件里去检索调用方法，或者调用父类的构造方法
* 子类构造器：利用supper调用父类构造器必须是子类构造器的第一条语句;如果子类的构造器没有显式调用父类的构造器，则会自动调用父类的默认的没有参数的构造器，如果父类没有不带参的构造器，那么会编译报错。

而针对子类所特有的属性和方法就在子类里加就好：

```java
public class Manager extends Employee {

    public double bonus;

    public Manager(String name, double salary, int year, int month, int day) {
        super(name, salary, year, month, day);
      	bonus = 0;
    }
  
    public void setBonus(double bonus){
      this.bonus = bonus;
    }
  
    @Override
    public double getSalary() {
        return bonus + super.getSalary();
    }
  
//	@Override
//    public double getSalary() {
//      return bonus + salary;
//	return bonus + getSalary();
//    }

}

```

**继承层次**：

> 由一个公共父类派生出来的所有类的集合称为继承层次，在继承层次中，从某个特定的类到其祖先的类的路径被称为该类的继承链。

Java是不支持多继承的，就是说一个子类只能继承自一个父类，不能有多个父类。

### 二.多态

**多态：**

父类的引用指向子类的对象，当我们声明一个父类的对象，它既可以引用一个父类的对象，也可以引用一个子类的对象，书上说的是，出现超类对象的任何地方都可以用子类的对象置换。反过来是不行的，不能将一个超类的引用赋给子类的变量。看下面的代码，

```java
public class Test {

    public static void main(String []args){

        Employee e;
        Manager staff2 = new Manager("b",5000,2020,11,13);

        e = staff2;

        System.out.println(e.getClass());						
        System.out.println(e instanceof Employee);
        System.out.println(e instanceof Manager);
    }
  
  //结果：
  // class extend.Manager
	// true
	// true

}
```

e被声明为一个Employee对象以后，把Employee的子类Manager的对象赋值给e后，我们发现它既是一个Manager实例，也是一个Employee实例，一个对象，两个状态，就是多态了

写了一个好玩的例子：

```java
public class Person {
    public void eat(Food food){
        System.out.println("I'm eating "+food.getName());
    }
}

public interface Food {
    public String getName();
}

public class Apple implements  Food{

    @Override
    public String getName() {
        return "Apple";
    }
}

public class Banana implements Food {

    @Override
    public String getName() {
        return "Banana";
    }
}

public class Peach implements Food {

    @Override
    public String getName() {
        return "Peach";
    }
}

public class Test {

    public static void main(String []args){

        Person p = new Person();
        p.eat(new Apple());
        p.eat(new Banana());
        p.eat(new Peach());
        p.eat(new Food() {
            @Override
            public String getName() {
                return "unknown";
            }
        });
    }

}
```

最后打印出来的结果：	

> I'm eating Apple
> I'm eating Banana
> I'm eating Peach
> I'm eating unknown

在实现了Food接口以后，Apple类，Banan类还有Peach类都可以向上转型变为Food，也就是说他们的对象都可以当作food对象来用，当person调用eat方法的时候，可以把这些水果的对象传进，因为他们本身就是Food对象的实例，而调用food的getName方法的时候，就会优先检索具体实现类的方法，这里的多态不是继承里面的，但是思想一样。

### 三.抽象类

在面向对象的概念中，所有的对象都是通过类来描述，但并非所有的类都用来来描述对象，如果一个类中没有包含足够的信息来描述一个具体的对象，这样的类就可以作为抽象类。

举个例子：所有的动物（animal）都可以移动（move），可以吃东西（eat），但是不同的动物，它移动或者吃东西都是不一样的，必须由具体的种类实现，比如鱼是游，吃蚯蚓，蛇是爬行，吃老鼠，这样，我们把移动和吃东西抽象出来写成一个抽象类，让他的子类去实现:

```java
public abstract class Animal {
    public abstract void move();
    public abstract void eat();
}

public class Snack extends Animal {
    @Override
    public void move() {
        System.out.println("蛇是爬行动物");
    }

    @Override
    public void eat() {
        System.out.println("蛇是吃老鼠的");
    }
}

public class Fish extends Animal {
    @Override
    public void move() {
        System.out.println("鱼在水里游");
    }

    @Override
    public void eat() {
        System.out.println("鱼吃蚯蚓");
    }
}

public class Test {

    public static void main(String []args){

        Snack snack = new Snack();
        Fish fish = new Fish();
        snack.move();
        snack.eat();
        fish.move();
        fish.eat();
    }

}

```

> 蛇是爬行动物
> 蛇是吃老鼠的
> 鱼在水里游
> 鱼吃蚯蚓

抽象类除了不能实例化对象，类的其他功能仍然存在（可以有构造器，但不一定要有抽象方法），所有抽象类必须要被继承后才可以使用

抽象方法：如果想要设计一个类，包含一个特别的成员方法，该方法的具体实现由他的子类确定，可以在父类中声明为抽象方法（抽象方法是没有方法体的），那么这个类也就是抽象类了（不写abstract会报错），任何继承了这个抽象类的子类都必须重写这个抽象方法，或者声明自己是一个抽象类，其中构造方法和静态方法是不能被声明为抽象的。

**为什么抽象类也要有一个构造器呢？**

1.首先要知道JAVA中的构造器，不是用来实例化对象的，而是用来初始化的。

2.抽象类中并不都是抽象方法，里面有成员变量，构造器是用来给这些成员变量赋值的。

【补充】：abstract只能用来修饰类和方法，不能用来修饰属性。

3.要知道抽象类是用来继承的，为什么用来继承，一般是用来实现多态！那么我们知道，在new一个子类对象的时候，默认调用了super();

那如果没有构造器，这个super怎么调用啊。

### 四.老父亲--Object类相关介绍，泛型数组数组，包装和自动装箱



**Object类**：如果一个类继承了其他父类，就不会直接继承Object，但是它的父类如果再没有父类，就会继承Object，也就是说，Object是任何一个类的直接或间接的父类。因此，我们随意创建一个类，其实也不是空的，起码这个类中有Object类中的属性和方法。

```java
public class Example {

    public Example() {

    }

}

public class Test{
	pubic static void main(String [] args){
	
	Example example = new Example();
	System.out.println(example.equals(null));
        System.out.println(example.getClass());
        System.out.println(example.hashCode());
        System.out.println(example.toString());

	}

}
```

> false
> class extend.Example
> 603742814
> extend.Example@23fc625e

**简单介绍一下Object里的几个方法**：

1. equals()方法：

   ```java
   public boolean equals(Object obj) {
           return (this == obj);
       }
   ```

   

   2. 由方法定义可知，Object类中的equals方法，用来比较两个引用的虚地址。当且仅当两个引用在物理上是同一个对象时，返回值为true，否则将返回false。

   3. **注意：**当此方法被重写时，通常有必要重写 hashCode 方法，以维护 hashCode 方法的常规协定，该协定声明相等对象必须具有相等的哈希码。

2. hashCode()方法：

   ```java
   public native int hashCode();
   ```

   native含义native方法称为本地方法。在java源程序中以关键字“native”声明，不提供函数体。
   其实现使用C/C++语言在另外的文件中编写，编写的规则遵循Java本地接口的规范(简称JNI)。
   简而言就是Java中声明的可调用的使用C/C++实现的方法。

   

   3. 总之不管怎么生成的散列码，在Object 类里，equals返回为true，则hashCode是相同的

   4. 书上讲了一下String类复写的hashCode（）方法：

      1. > int hash = 0;
         >
         > for(int i = 0; i < length(); i++){
         >
         > ​	hash = 31 * hash + charAt(i);
         >
         > }

      2. 所以对于String类的对象来说来说，只要内容是一样的，则他们散列码就是一样的，而书中的StringBuilder类，没有定义hashCode()方法，所以必须要是一个对象或者一个对象的引用才会返回一样的散列码。

3. getClass()方法：

   1. 返回运行时包含对象信息的类对象,后面反射再说

4. toString()方法：

   1. 返回表示对象值的字符串，没有重写的话，就是一个类名接着一个字符串

      > extend.Example@23fc625e

   2. 一般会重写这个方法打印一个对象的基本信息

**泛型数组列表**：ArrayList<类型名>(数量)；

1. 泛型就是广泛的类型，这里的泛型数组就是可以存放某个特定类型的对象的数组，比如

   > ArrayList< Employee > staff = new ArrayList<>(100);
   >
   > 这里就规定这个泛型数组存放Employee对象，且容量设置为100，注意泛型数组的容量可以不设置，不像c语言一样，声明就必须规定大小，并且泛型数组会自动扩容，当这里超过100个元素就会自动扩容。

2. 可以用add()方法和get()方法添加和访问数组元素，方法都比较简单。

**对象包装器和自动装箱**：

对象包装器类是不可变的，一旦构造了包装器，由于他的值被final修饰，就无法改变包装在其中的值，同时类被final修饰，不能被继承

| 基本数据类型 | 包装类    |
| :----------: | --------- |
|     byte     | Byte      |
|    short     | Short     |
|     int      | Integer   |
|     long     | Long      |
|    float     | Float     |
|    double    | Double    |
|     char     | Character |
|   boolean    | Boolean   |

​      Integer里的value和构造器：

```java
  private final int value;
		public Integer(int value) {
        this.value = value;
    }
```

之前介绍的泛型数组当要储存基本类型数据时，由于<>里必须是类型参数，所以只能填类型名，而不能填基本数据类型，只能写成` ArrayList<Integer>()`这样的

简单的装箱和拆箱：

泛型数组里给我提供了这样一个便利：

```java
List<Integer> list = new ArrayList<>();
list.add(3);
int n = list.get(i);
```

其实根据ArrayList的定义，里面存放的是一个个对象，但是为什么往里面可以直接add一基本数据类型呢，从数组里拿出来的为什么是int，而不是integer对象呢？

其实这就是java的自动装箱和拆箱，表面上看，编译器执行的是list.add(3),其实由于自动装箱，编译器执行到这里的时候自动把3转化为Integer.valueOf(3)这是一个静态的构造方法，会返回一个integer对象,打包成一个Integer对象，送进了泛型数组里。同里取出的时候，也会把取出的Integer对象通过intValue()方法转成一个int数据。

### 五.反射

Class类对象：

` Person p = new Person();`执行这样一行代码，机器里的操作就是，在文件夹里寻找Person.java,然后javac编译为.class的字节码文件，再通过jvm的类加载器将其加载到堆中，创建一个Class类对象，这个对象不是一般的对象，它所包含的事是类的所有信息，然后通过调用里面的方法new 出一个普通的Person对象，反射主要就是通过获得这样一个Class类对象，可以在运行时检查类、接口、方法和变量等信息，无需知道类的名字，方法名等。还可以在运行时实例化新对象，调用方法以及设置和获取变量值。

java中提供3种方法获取内存中的Class对象

1. Class.forName()方式适用于运行时动态获取Class对象，只需将类名作为forName方法的参数：
2. 类名.class
3. 对象.getClass();

```java
	Person p = new Person();
	Class class1 = p.getClass();
        Class class2 = Person.class;
        Class class3 = Class.forName("extend.Person");
//			获得类对象
        System.out.println(class1);
        System.out.println(class2);
        System.out.println(class3);
```

> class extend.Person
> class extend.Person
> class extend.Person

```java
	Class class1 = p.getClass();
        Class class2 = Person.class;
        Class class3 = Class.forName("extend.Person");
//		获得接口
        Class[] interfaces= class1.getInterfaces();
//		获得构造器
        Constructor constructor = class1.getConstructor();
//		获得方法
        Method[] methods = class1.getMethods();
        for(Method m: methods){
            System.out.println(m);
        }
        System.out.println(interfaces.length);
        System.out.println(class1.getPackageName());
        System.out.println(constructor);
```


> public void extend.Person.eat(extend.Food)
> public final void java.lang.Object.wait(long,int) throws java.lang.InterruptedException
> public final void java.lang.Object.wait() throws java.lang.InterruptedException
> public final native void java.lang.Object.wait(long) throws java.lang.InterruptedException
> public boolean java.lang.Object.equals(java.lang.Object)
> public java.lang.String java.lang.Object.toString()
> public native int java.lang.Object.hashCode()
> public final native java.lang.Class java.lang.Object.getClass()
> public final native void java.lang.Object.notify()
> public final native void java.lang.Object.notifyAll()
> 0
> extend
> public extend.Person()
> class extend.Person
> class extend.Person
> class extend.Person

除了获得信息，可以看到包括方法，构造器，甚至接口都被我们保存到相应的实例里，我们可以通过调用这些实例来使用方法，构造器等等，反射是很有意思的东西，大家感兴趣可以玩一玩。

## 第六章 接口，lambda表达式与内部类

### 六.接口

一个类通过实现接口的方式，来实现接口中的各种方法，接口中的默认声明为public abstract（公共抽象的），接口和抽象类一样是不能被实例化的，接口的使用方式就是让其他的类去实现它，或者别的接口继承，其实最后也是要被类实现，进而实现里面的抽象方法

* 一个实现接口的类，必须实现接口的所有抽象方法，否则就要声明为抽象类
* 接口类型可以用来声明一个变量，他们可以成为一个空指针，或者被绑定为一个实现了此接口的对象
* 接口中可以定义常量，默认都是（static final），但是不能有实例域
* 可以用default修饰一个接口的方法，使得这个接口有一个默认实现

实现一个接口，就要重写其中的抽象方法，继承一个抽象类，也要实现其中的抽象方法，如果做不到，就要自己声明为抽象类，我们可以简单比较一下抽象类和接口：

> 区别一: 抽象类只能单继承,接口能多实现
> 区别二: 抽象类是一个类,可以被任意权限修饰符修饰,静态和非静态属性，final和非final属性，可
> 以有抽象方法和非抽象方法；
> 接口只能被public,final修饰,只能有静态方法,即使没有显示的声明，而且是不可修改的；
> 区别三: 抽象的事物不同:
> 抽象类是对整个类抽象,包括属性,行为；接口是对类的行为(局部)进行抽象；
> 区别四: 定义的时候,定义抽象类和接口的思想不同；
> 设计抽象类是自下而上的过程,我子类需要,所以我定义抽象类； 
> 设计接口是自上而下的过程,我接口规范某一行为,我某类需要这个行为,我实现某接口

**接口和回调**：

回调：

书上的例子也挺好，就是用一个计时器，每隔一定时间，发送一事件，然后再通过实现的接口的自定义方法去执行一些操作，书上的例子包装的很好。

类A调用类B的b方法 类B的b方法执行完毕主动调用类A的callback()方法

```java
public interface CallBack {
    void wakeUp();
}


//		这里的Customer我们实现了一个接口
public class Customer implements CallBack {

  //这个方法就是我们要去执行时，会回来调用callback接口的方法
    public void orderWakeUp(Hotel hotel){
        System.out.println("顾客订购叫醒服务！");
        hotel.goToWakeUp(this);
    }

    @Override
    public void wakeUp() {
        System.out.println("顾客被叫醒了！");
    }
}

public class Hotel {
  //通过调用这个方法，回调回之前的对象的方法，核心是这里的参数用接口的声明
    public void goToWakeUp(CallBack callBack) {
        try {
            // 模拟时间
            Thread.sleep(3000);
            System.out.println("过了一会……");
            callBack.wakeUp();
        } catch (InterruptedException e) {

            e.printStackTrace();
        }
    }
}

public class CallBackTest {
    public static void main(String []args){
        Hotel hotel = new Hotel();
        Customer customer = new Customer();
        customer.orderWakeUp(hotel);
    }
}
```

结果：

> 顾客订购叫醒服务！
> 过了一会……
> 顾客被叫醒了！

### 七.lambda表达式

形式：

```java
( params ) -> expression;//返回表达式的值
( params ) -> statement; //返回void
( params ) -> { statements };
//如果代码要完成的计算无法放在一个表达式里面，就可以像写方法一样写在一个{}里，并要包含显示的return语句。
```

* 如果方法没有参数，也要像方法一样留一个空括号
* 当lambda表达式只在一些分支返回值，而其他时候不返回，是会报错的

自己的理解：lambda表达式的就像是一个包装的方法，没有说明方法的名称和返回类型，但是规定了参数的传入，以及返回的值，整个lambda更像是一个一次性的方法，最后的结果就是箭头后的返回值。

**函数式接口**：对于只有一个抽象方法的接口，需要这种接口的对象时，我们可以提供一个lambda 表达式,lambda表达式可以转为接口

```java
public class Test {

    public static void main(String []args){

        Person p = new Person();
        p.eat(new Apple());
        p.eat(new Banana());
        p.eat(new Peach());
        p.eat(new Food() {
            @Override
            public String getName() {
                return "unknown";
            }
        });
      	p.eat(()->"lambda...");
    }
```

输出：

> I'm eating Apple
> I'm eating Banana
> I'm eating Peach
> I'm eating unknown
> I'm eating lambda...

这里的eat本来是传入一个Food的实例的，但是Food作为一个只有一个抽象方法的接口，刚好可以用lambda书写，只需要注意其抽象方法返回的是String，这样的lambda表达式刚好从一个类似于方法的东西在这里转型为接口使用。

### 八. 内部类

主要分为四类：

1. 成员内部类（成员内部类不允许存在任何static修饰的变量和方法，正如成员方法中不能有任何静态属性一样（成员方法与对象有关，静态变量与类有关），成员内部类与外部类的实例相联系，可以访问外部类的所有成员数据，正因为成员内部类与外部类的实例联系，因此它不允许在 方法区（Method Area）中申请内存，只能在 堆（Heap）中 申请，所以不能在其内部定义static成员变量，可以理解成内存上的一种包含关系。

```java
      public class Outter {
          String s;
          class Inner{
          }
      }
```

​      

2. 静态内部类

   1. 被关键字static修饰的成员内部类就是静态内部类

   2. 静态内部类的创建不依赖于外部类，而可以直接创建

   3. 静态内部类不可以使用任何外部类的非static类（包括属性和方法—），但可以自己存在自己的成员变量

```java
      public class Outter {
          String s;
          static class Inner{
          }
      }
```

​      

3. 方法（局部）内部类

   1. 定义在方法内部的一个类，方法内部类不允许使用访问修饰符（别的地方也没有理由可以使用它），因为方法内部类对外部完全隐藏，除了创建这个类的方法可以访问到他，其他都不可以

   2. 方法内部类如果要使用形参，该形参就要修饰为final

      1. 原因：局部变量和局部内部类的生命周期是不一致的，当局部变量所处的函数执行完毕后，就会被销毁，但局部内部类的对象还可能一直存在（若该对象仍然被引用），而当对象要访问该局部变量时，访问的就是不存在的东西了，用final修饰就会将这个局部变量复制一份，并一直保存了这个拷贝每次访问时就能找到这个拷贝




```java
public class Outter {
      public Inner getInner2(){
          class Inner2 extends Inner{
              String s = "Inner2";
          }
          return new Inner2();
      }
  }
```

4. 匿名内部类

   1. 用得最多的内部类，一个没有名字的方法内部类，

   2. 匿名内部类继承一个抽象类，或者实现一个接口

   3. 匿名内部类没有类名，因此也没有构造方法，不能有静态方法和属性

   ```java
      p.eat(new Food() {
                  @Override
                  public String getName() {
                      return "unknown";
                  }
              });
   ```

      