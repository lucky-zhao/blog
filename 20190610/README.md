# 设计模式
* 设计模式和开发语言无关，它是软件开发的编程思想。

设计模式总共有23种，可以分为三大类：
1. 创建型(工厂模式，抽象工厂模式，单例模式，建造者模式，原型模式)
2. 结构型(适配器模式，桥接模式，过滤器模式，组合模式，装饰器模式，外观模式，享元模式，代理模式)
3. 行为型(责任链模式，命令模式，解释器模式，迭代器模式，中介者模式，备忘录模式，观察者模式，状态模式，空对象模式，策略模式，模版模式，访问者模式)

设计模式的六大原则：
1. 开闭原则：
对扩展开放，对修改关闭。在程序需要扩展的时候，不修改原有的代码，实现一个可插拔的效果。使程序扩展性更好，易于维护和升级。
2. 依赖倒转原则：
程序要依赖于抽象接口，不要依赖具体的实现。
例：张三是个司机，他现在开的是宝马车
```
public class Bmw {
    public void run(){
        System.out.println("宝马车行驶....");
    }
}

public class Driver {
    //司机驾驶汽车
    public void drive(Bmw bmw) {
        bmw.run();
    }
}

public class Client {
    public static void main(String[] args) {
        Driver zhangsan = new Driver();
        Bmw bmw = new Bmw();
        //张三开宝马
        zhangsan.drive(bmw);
    }
}
```
张三如果想换个奔驰车开，此时代码要怎么办？新建一个奔驰类，它也有个run方法，然后...这样肯定是合适的。
没有使用依赖倒置原则，类之间的耦合性太高，一旦需求改动，需要改动很多地方。使用依赖倒置原则：司机都会开车，可以建一个司机的接口，汽车都能被驾驶，建一个汽车的接口
```
public interface ICar {
    //汽车都能跑
    void run();
}

public interface IDriver {
    //司机都会驾驶汽车,入参是汽车的接口，传给我开什么车，我就开什么车
    void drive(ICar iCar);
}

public class Client {
    public static void main(String[] args) {
        IDriver zhangsan = new Driver();
        ICar car = new Bmw();//需要开什么车，就创建这个车的实例，其他代码不需要变动
        zhangsan.drive(car);
    }
}
```
简单来说，依赖倒置原则就是通过抽象使各个类之间相对独立，降低耦合度。

3. 里氏替换原则
任何基类可以出现的地方，子类一定可以出现。(子类可以扩展父类的功能，但不能改变父类原有的功能)

* 子类不能重写父类中已经实现的方法，父类中已经实现的方法其实是一种已经约定好的规范，如果我们随意修改它，会带来意想不到的错误。例：
```
public class A {
    public void fun(int a, int b) {
        System.out.println(a+b);
    }
}


public class B extends A {
    @Override
    public int fun(int a, int b) {
        return a - b;
    }
}

public class TTest {
    public static void main(String[] args) {
        A a=new A();
        System.out.println(a.fun(1,2));

        //按照里氏替换原则，父类出现的地方，子类可以替换
        A b=new B();
        System.out.println(b.fun(1,2));
    }
}
```
运行此程序会发现，由于子类重写了父类的方法，替换成子类实现，结果并不是想要的。

4. 接口隔离原则
这个意思很简单：保证接口的单一性，降低耦合度，便于维护

5. 最少知道原则
一个类应该对自己需要耦合或调用的类知道得最少。调用的类内部不管如何复杂都和我没有关系。类与类之间关系越密切，耦合度越大，一个类改变，对另外的类影响也很大。
具体实现方式：
*   从依赖者角度来说，只依赖需要依赖的对象
*   从被依赖(被调用)者的角度来说，只需要提供应该提供的方法
*   在类的划分上，应该创建弱耦合的类。可以用内部类实现
*   尽量降低类成员的访问权限
*   不暴露类的属性，应该提供get set方法

6. 组合复用原则

尽量使用对象组合/聚合, 而不是继承关系达到软件复用的目的。组合复用本质上是`has-a`的特征，即有一个什么属性。而继承是`is-a`，即是一个什么类型。`has-a`更加灵活，如果有需要定义一个其他类型的成员变量即可。这就是组合的含义。
继承的缺点：
*   破坏了类的封装性，因为继承会把父类的实现暴露给子类
*   子类和父类耦合度高，父类的实现任何改变都会导致子类发生变化

设计模式
* 创建型模式
1. 工厂模式：定义了一个创建对象的接口, 但由子类决定要实例化的类是哪一个。
适用场景：
* 类不知道要创建哪个对象时
* 类用他的子类来指定创建对象时
简单例子：
```
public interface Car {
    //汽车品牌
    String getName();
}

public class Ben implements Car {
    @Override
    public String getName() {
        return "Benz";
    }
}

public class Bmw implements Car {
    @Override
    public String getName() {
        return "bmw";
    }
}

public class CarFactory {

    public Car getCar(String name) {
        if (name.equals("BMW")) {
            return new Bmw();
        } else if (name.equals("Benz")) {
            return new Ben();
        } else {
            System.out.println("生产不了这个品牌的汽车：" + name);
            return null;
        }
    }
}
//测试
public class TTest {
    public static void main(String[] args) {
        CarFactory carFactory=new CarFactory();
        Car bmw = carFactory.getCar("Benz");
        System.out.println(bmw.getName());
    }
}

```
简单实现了一个工厂模式，想要生产什么品牌的车就调用`getCar()`即可，传入对应的参数。

2. 抽象工厂模式： 提供一个接口, 用于创建相关或依赖对象的工厂, 而不需要指定具体类。


