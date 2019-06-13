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

5. 原型模式：用于创建重复的对象，同时又能保证性能。java中的克隆技术，以某个对象为原型，复制出新对象。新对象类似于`new`，但是`new`出来的对象属性是默认值。而克隆出来的对象属性和原对象相同，并且对克隆出来的新对象修改不会影响原型对象。
对象的复制又有两种：浅拷贝和深拷贝
浅拷贝：创建一个新的对象，这个对象和原对象属性值相同，如果是基本数据类型，那么就是拷贝的基本数据类型的值，因为是两份不同的数据，所以对其中一个进行修改，不会影响其他的。如果是引用类型，那么拷贝的就是引用(内存地址)，指向的还是原来的对象。
引入java中值传递的两种不同：
* 基本数据类型值传递，简单的例子：
```
public class NumberTest {
    public static void main(String[] args) {
        int a=5;
        NumberTest.change(a);
        System.out.println(a);

    }
    private static void change(int a){
        a=10;
    }
}
```
输出还是5，不会因为调用了`change`方法被改变。方法里的基本数据类型都是存在于栈内存里，在调用`change`方法时，把5拷贝过去，并且在该方法栈里定义了一个变量来接受传递过来的值。此时栈内存里有2个变量，分别对应了5和10。在`change`方法中对变量修改，改变只是这个方法中变量的值，不会影响其他方法里的值。
* 引用类型的值传递，例子1：
```
public class User {
    private int age;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}

public class UserTest {
    public static void main(String[] args) {
        User user=new User();
        user.setAge(10);
        changeObj(user);
        System.out.println(user.getAge());
    }

    private static void changeObj(User user){
        user.setAge(20);
    }
}

```
输出结果是20，main方法中原来定义的age=10，变成了20。大概执行过程：
*   `main`方法定义了一个`User user=new User();`，变量user存在于栈内存，而实际new出来的对象存放在堆内存。
*   调用`changeObj`的时候把栈内存的user引用拷贝了一份传了过去，也就是说`changeObj`方法里的user变量指向的还是原来的堆内存中的那个对象。也就是2个引用指向同一个对象。这样的情况就很明显了，改来改去都是改的同一个对象，所以不管哪个引用获取属性值的时候，都是被修改后的新值。因为只有一个对象。

* 例子2:
```
public class StringTest {
    public static void main(String[] args) {
        String str="aaa";
        changeStr(str);
        System.out.println(str);
    }
    private static void changeStr(String str){
        str="bbb";
    }
}
```
这个输出什么？`String`是引用类型，是不是和上面的结果一样呢？但是实际上输出结果还是`aaa`。
执行过程：
*   `main`方法定义了一个`String str="aaa";` ，变量str存于栈内存中，变量值`aaa`会先到常量池中查找是否存在相同的字符串，如果存在，则直接把栈内存中的引用指向该字符串，如果不存在，则在常量池中生成一个字符串，在将引用指向它。
*   在常量池中没有`aaa`，所以在堆中开辟内存并存值`aaa`
*   调用`changeStr`的时候把栈内存中的str引用拷贝一份传了过去
*   拷贝出来的str引用进入`changeStr`方法，先在常量池中查找是否有`bbb`这个字符串，没有的话在堆中开辟内存并且存值`bbb`，此时就是2个引用，2个对象。
*   `changeStr`中是对它方法体中的str赋值，跟main方法中的str没有关系，所以输出还是aaa

浅拷贝的例子：
```
public class Student implements Cloneable {
    private int id;

    private String name;

    private InnerAge innerAge;

    @Override
    public String toString() {
        return "Student{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", innerAge=" + innerAge +
                '}';
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    public InnerAge getInnerAge() {
        return innerAge;
    }

    public void setInnerAge(InnerAge innerAge) {
        this.innerAge = innerAge;
    }

    //静态内部类，方便测试引用类型
    public static class InnerAge{
        private int age;

        @Override
        public String toString() {
            return "InnerAge{" +
                    "age=" + age +
                    '}';
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```
测试用例：
```
public class TTest {
    public static void main(String[] args) throws CloneNotSupportedException {
        Student student = new Student();
        Student.InnerAge innerAge = new Student.InnerAge();
        innerAge.setAge(10);
        student.setId(1);
        student.setName("a");
        student.setInnerAge(innerAge);
        System.out.println("原对象：" + student);
        //克隆出一个新的对象

        Student newObj = (Student) student.clone();
        //改变原来对象里的属性
        student.getInnerAge().setAge(20);
        student.setName("b");
        student.setId(2);
        System.out.println("改变原对象值后的克隆对象："+newObj);
    }
}
```
输出结果：
```
原对象：Student{id=1, name='a', innerAge=InnerAge{age=10}}
改变原对象值后的克隆对象：Student{id=1, name='a', innerAge=InnerAge{age=20}}
```
可以看到浅拷贝出来的对象，这个对象和原对象属性值相同，如果是基本数据类型，那么就是拷贝的基本数据类型的值，因为是两份不同的数据，所以对其中一个进行修改，不会影响其他的。如果是引用类型，那么拷贝的就是引用(内存地址)，指向的还是原来的对象。两个引用指向同一个对象，所以改来改去都是对同一个对象的修改。所以输出innerAge由以前的10变成20。

深拷贝，把某个对象里面所有的属性拷贝，并且需要把该对象里面的对象也要拷贝。这样拷贝出来的新对象就是全新的。
把刚才的例子改造一下`Student`里面不光clone自己的类对象，还要拷贝内部类`InnerAge`，并且`InnerAge`类也需要实现`Cloneable`接口，实现它的`clone`方法。
```
public class Student implements Cloneable {
    private int id;

    private String name;

    private InnerAge innerAge;

    @Override
    public String toString() {
        return "Student{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", innerAge=" + innerAge +
                '}';
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        Object o = super.clone();
        Student student = (Student) o;
        student.innerAge = (InnerAge) student.getInnerAge().clone();
        return o;
    }

    public InnerAge getInnerAge() {
        return innerAge;
    }

    public void setInnerAge(InnerAge innerAge) {
        this.innerAge = innerAge;
    }

    //静态内部类，方便测试引用类型
    public static class InnerAge implements Cloneable {
        private int age;

        @Override
        protected Object clone() throws CloneNotSupportedException {
            return super.clone();
        }

        @Override
        public String toString() {
            return "InnerAge{" +
                    "age=" + age +
                    '}';
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```
运行刚才的测试代码输出：
```
原对象：Student{id=1, name='a', innerAge=InnerAge{age=10}}
改变原对象值后的克隆对象：Student{id=1, name='a', innerAge=InnerAge{age=10}}
```
innerAge的值没有变，说明他们是2个完全独立的对象。这就是深拷贝。而原型模式，本质上就是拷贝对象。

6. 适配器模式：将一个类的接口转换成客户希望的另外一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。
生活中的例子：
* 耳机3.5mm接口，有些手机有这个接口，有些手机取消了耳机孔，只有一个typec接口，这样就需要一个转接头，这个转接头就是一个适配器。
* 家里的插头有2孔的，3孔的。如果全是2孔插座，那么3孔的插头就需要一个转接头，这个也是适配器。
例子：我的笔记本提供SD卡槽，可以直接插SD卡读取数据，如果我想读取手机卡TF卡里面的数据，这时就需要一个读卡器(适配器)。
我的电脑：
```
public interface Computer {
    //电脑可以直接读取SD卡
    String readSD(SDCard sdCard);
}
public class ThinkpadComputer implements Computer {
    @Override
    public String readSD(SDCard sdCard) {
        return sdCard.readSD();
    }
}
```
SD卡接口和实现类：
```
public interface SDCard {
    //读取SD卡
    String readSD();
}
public class SDCardImpl implements SDCard{
    @Override
    public String readSD() {
        return "读取SD卡的信息";
    }
}
```
测试代码：
```
        Computer computer=new ThinkpadComputer();
        SDCard sdCard=new SDCardImpl();
        System.out.println(computer.readSD(sdCard));
```
输出`读取SD卡的信息`。现在要做个读卡器，把TF卡插入读卡器，然后在把读卡器插入电脑，就可以实现读取TF卡的数据。
首先有个TF卡
```
public interface TFCard {
    //读取TF卡
    String readTFCard();
}
public class TFCardImpl implements TFCard{
    @Override
    public String readTFCard() {
        return "读取TF卡的信息";
    }
}
```
适配器
```
public class SDAdapterTF implements SDCard {
    private TFCard tfCard;

    public SDAdapterTF(TFCard tfCard) {
        this.tfCard = tfCard;
    }

    @Override
    public String readSD() {
        return tfCard.readTFCard();
    }
}
```
这个适配器的功能就是把读取SD卡的转换成读取TF卡，实现方式是：实现SD卡的接口，然后使用组合的方式在该适配器中定义了一个TF卡，然后在重写`readSD`方法，用TF卡实现具体细节。
测试代码：
```
public class TTest {
    public static void main(String[] args) {
        //拿出电脑
        Computer computer=new ThinkpadComputer();
        //拿出TF卡
        TFCard tfCard=new TFCardImpl();
        //把TF卡插到 adapter 适配器(读卡器)
        SDCard adapter=new SDAdapterTF(tfCard);
        //在把 adapter 读卡器插到电脑，电脑有读取SD卡的功能。而适配器实现了SDCard，所以可以直接把适配器插到电脑上
        System.out.println(computer.readSD(adapter));
    }
}
```
输出`读取TF卡的信息`。
适配器使用场景：
* 需要使用已有的类或接口，但是该类或接口不满足需求。
* 通过接口转换，把一个类插入到另外一个类中。

还有一种方式，如果一个接口中有很多方法，但是只需要实现其中某些方法就行了，其他方法用不到，这时候也可以用适配器。
```
public interface CommonService {
    void a();
    void b();
    void c();
    void d();
    void e();
    void f();
}
```
实际我只要a,b,c3个方法，但是实现这个接口，就必须把这些方法都重写。
```
public class CommonServiceImpl implements CommonService{
    @Override
    public void a() {
        //do something
    }
    @Override
    public void b() {
        //do something
    }
    @Override
    public void c() {
        //do something
    }
    @Override
    public void d() {

    }
    @Override
    public void e() {

    }
    @Override
    public void f() {

    }
}
```
虽然其他几个用不到的方法可以不实现，但是代码看起来乱，没用的代码放着干嘛？这时可以用一个抽象类来实现接口。
```
public abstract class AdapterService implements CommonService {
    @Override
    public void a() {
        //do something
    }
    @Override
    public void b() {
        //do something
    }
    @Override
    public void c() {
        //do something
    }
}
```
这样就可以实现你需要的方法。其他的方法不写也没事。`AdapterService`其实也是起到一个适配器的作用。

7. 桥接模式：

