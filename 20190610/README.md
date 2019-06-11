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

缺点：
* 如果要生成大众汽车，那么必须定义一个类实现`Car`接口，然后需要修改`CarFactory`代码实现生产大众汽车。这个改动原有代码，违背了开闭原则。每加入一个新的品牌，都需要改动原有代码。这个问题在工厂方法模式中得到了一定的解决。
这种业务场景很常见。本来说好了，只生产宝马和奔驰两种车，现在又要加其他品牌的汽车。这样修改代码，代价很大。

2.抽象工厂模式： 提供一个接口, 用于创建相关或依赖对象的工厂, 而不需要指定具体类。

还是以生产汽车为例子：
建一个抽象工厂类，工厂可以生产产品，但是具体生产什么产品，由实现类去定义。
```
public interface Factory {
    Car produce();
}

```
建一个汽车抽象类，由具体实现类去实现方法。
```
public interface Car {
    void show();
}
public class Benz implements Car  {
    @Override
    public void show() {
        System.out.println("生产奔驰");
    }
}
public class Bmw implements Car {
    @Override
    public void show() {
        System.out.println("生产宝马");
    }
}
```
建两个实现类，分别生产奔驰汽车和宝马车
```
public class BenFactory implements Factory {
   @Override
    public Car produce() {
        return new Benz();
    }
}
public class BmwFactory implements Factory {
    @Override
    public Car produce() {
        return new Bmw();
    }
}

```
调用的时候：
```
public class Client {
    public static void main(String[] args) {
        //生产奔驰
        Factory benFactory = new BenFactory();
        benFactory.produce().show();

        //生产宝马
        Factory bmwFactory = new BmwFactory();
        bmwFactory.produce().show();
    }
}
```
如果需求变更了，需要生产别克汽车，那么不需要改动原有的代码，直接新建一个Buick类
```
public class Buick implements Car {
    @Override
    public void show() {
        System.out.println("生产别克汽车");
    }
}
```
并且新建一个生产别克汽车的工厂类即可。
```
public class BuickFactory implements Factory {
    @Override
    public Car produce() {
        return new Buick();
    }
}
```
好处：
* 更加符合开闭原则，需求变动，不需要改原有代码，只需要增加对应的产品类和该产品的工厂子类即可。
* 每个工厂只创建一种产品
缺点：
* 添加新产品时，除了增加产品类之外，还需要建他的工厂类。增加了系统的复杂度

简单小结一下：
* 简单工厂模式就是一个专门生产某个产品的类。上面的例子，汽车工厂类，专门生成汽车。传入`Benz`会生产奔驰车，传入`Bmw`会生产宝马车。
* 工厂模式是汽车工厂的父类或者接口`Factory`，有生产汽车的方法`produce()`。奔驰汽车工厂`BenFactory`和宝马汽车工厂`BmwFactory`实现它，可以分别生产奔驰车和宝马车。生产哪种车不像一开始由参数决定，而是创建汽车工厂时，由宝马/奔驰工厂决定。
```
 //生产奔驰
        Factory benFactory = new BenFactory();
        benFactory.produce().show();

        //生产宝马
        Factory bmwFactory = new BmwFactory();
        bmwFactory.produce().show();
```
* 抽象工厂模式不仅能生产汽车，还能生产汽车座椅，奔驰汽车工厂和宝马汽车工厂都实现它，就可以生产奔驰车+奔驰牌座椅，宝马车+宝马牌座椅.
新建这两个产品的抽象类，一个可以生产汽车，一个可以生产座椅。具体细节由实现类决定。
```
public interface Car {
    void show();
}
public interface Seat {
    void show();
}
```
汽车工厂实现类
```
public class BenzCar implements Car {
    @Override
    public void show() {
        System.out.println("生产奔驰车");
    }
}
public class BmwCar implements Car {
    @Override
    public void show() {
        System.out.println("生产宝马车");
    }
}
```
座椅工厂实现类
```
public class BenzSeat implements Seat {
    @Override
    public void show() {
        System.out.println("生产奔驰汽车座椅");
    }
}
public class BmwSeat implements Seat {
    @Override
    public void show() {
        System.out.println("生产宝马汽车座椅");
    }
}
```
奔驰工厂实现生产奔驰车和奔驰座椅
```
public class BenzFactory implements Factory {
    @Override
    public Car produceCar() {
        return new BenzCar();
    }

    @Override
    public Seat produceSeat() {
        return new BenzSeat();
    }
}
```
宝马工厂实现生产宝马车和宝马座椅
```
public class BmwFactory implements Factory {
    @Override
    public Car produceCar() {
        return new BmwCar();
    }

    @Override
    public Seat produceSeat() {
        return new BmwSeat();
    }
}
```
客户调用的时候：
```
public class Client {
    public static void main(String[] args) {
        //生产奔驰牌汽车和座椅就创建奔驰汽车工厂的实例对象
        Factory benzFactory = new BenzFactory();
        benzFactory.produceCar().show();
        benzFactory.produceSeat().show();
        System.out.println("----------");
        //生产宝马牌汽车和座椅
        Factory bmwFactory = new BmwFactory();
        bmwFactory.produceCar().show();
        bmwFactory.produceSeat().show();
    }
}

生成奔驰车
生产奔驰汽车座椅
----------
生产宝马车
生产宝马汽车座椅

```
实际场景中如果确定只要生产宝马车就行了，那么则不需要创建工厂。有了工厂模式简单来说，需要什么对象就到工厂里取，调用工厂里的方法即可，如果以后需要添加或者修改对象，那么直接到工厂里添加或者修改即可。所有调用这个方法的人都会拿到新对象。

3. 单例模式：保证一个类只有一个对象
* 最常见的写法,线程不安全，懒加载：
```
public class Single {

    private static Single instance;

    //私有构造器
    private Single() {
    }

    public static Single getInstance() {
        if (instance == null) {
            instance = new Single();
        }
        return instance;
    }
}
```
* 如果要保证线程安全可以在`getInstance`方法前加`synchronized `
* 饿汉式线程安全，没有懒加载,类在加载的时候就已经被实例化
```
public class Single {

    private static Single instance=new Single();

    //私有构造器
    private Single() {
    }

    public static Single getInstance() {
        return instance;
    }
}
```
还有其他实现方法，我就不试了。

4. 建造者模式：使用多个简单的对象构建成一个复杂的对象。引用《effective java》中的例子

java中的`StringBuilder`是个典型引用
```
 StringBuilder stringBuilder = new StringBuilder();
 stringBuilder.append("aa").append("bb").append("cc");
```
每次`append`都会返回一个`StringBuilder`对象，然后又可以继续`append`,一系列的`append`操作后，最终形成需要的对象。
建一个`User`类
```
public class User {
    private int id;
    
    private String name;
    
    private int age;
    
    private String account;
    
    private String pwd;
    
    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public User(int id, String name, int age, String account, String pwd) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.account = account;
        this.pwd = pwd;
    }
    ...其他构造器略...
 }
```
假设规定创建一个User对象，必须传入`id`,`name`，其他几个属性可赋值，可不赋值。那么会有很多个不同参数的构造器。这样会很麻烦。而且如果参数过多，容易搞错参数的位置。

如果用getter、setter方式试试呢，实体类里生成getter、setter，创建User对象的时候则可以：
```
        User user=new User(1，"lucky");
        //根据需要给属性设值
        user.setAccount("123456");
        user.setPwd("123456");
        user.setAge(20);
```
这样貌似可以实现想要的功能，但是根据书中所说`javabean`可能处于不一致状态<font color=red>(这个地方我不明白是什么意思，如果有人知道，请告诉我，万分感谢)</font>
用建造者模式：
```
public class User {
    private int id;

    private String name;

    private int age;

    private String account;

    private String pwd;

    private User(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.age = builder.age;
        this.account = builder.account;
        this.pwd = builder.pwd;
    }

    protected static class Builder {
        private int id;

        private String name;

        private int age;

        private String account;

        private String pwd;

        public Builder(int id, String name) {
            this.id = id;
            this.name = name;
        }

        protected User build() {
            return new User(this);
        }


        protected Builder age(int age) {
            this.age = age;
            return this;
        }

        protected Builder account(String account) {
            this.account = account;
            return this;
        }

        protected Builder pwd(String pwd) {
            this.pwd = pwd;
            return this;
        }
    }
}

```

User类里建了一个静态内部类，用来给User构建对象属性。在调用的时候则可以象`StringBuilder`一样链式调用，并且实现了必须传入`id`,`name`其他属性可选赋值的功能。
```
     User lucky = new User.Builder(1,"lucky")
                .account("123456")
                .pwd("123456")
                .age(20)
                .build();
```
相比之前的写法更加清晰，调用更加清楚。

5. 原型模式：用于创建重复的对象，同时又能保证性能。
