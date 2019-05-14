# 静态代理和动态代理
代理也是设计模式里面的一种，主要作用：
1. 为其他对象提供一种代理以控制对这个对象的访问
2. 在某些情况下，一个对象不适合或者不能直接引用另外一个对象，而代理对象可以在客户端和目标对象之间起作用
3. 可以借助代理类增强功能，而不影响原有的代码
4. 解耦，当两个类需要通信时，引入第三个类，将两个类解耦

主要特征： 代理类与委托类实现了相同的接口

代理也分两种：静态代理和动态代理
静态代理：由程序员创建，在编译期就已经存在.class文件
动态代理：在程序运行时，利用反射机制动态创建

尝试用代码实现静态代理：
新建一个UserService接口，里面有一个添加用户的方法
```
public interface UserService {

    void addUser(Long id, String name);
}
```
实现类：
```
public class UserServiceImpl implements UserService {

    @Override
    public void addUser(Long id, String name) {
        System.out.println("添加用户");
    }
}
```
有一个日志类，专门负责记录日志：
```
public class Logger {

    public static void start(){
        System.out.println("start...");
    }

    public static void end(){
        System.out.println("end...");
    }
}
```
代理类：
```
public class UserProxy implements UserService {

    //目标对象
    private UserService userService;

    public UserProxy(UserService userService) {
        this.userService = userService;
    }


    @Override
    public void addUser(Long id, String name) {
        Logger.start();
        userService.addUser(id, name);
        Logger.end();
    }
}
```
测试：
```
public class TestUser {
    public static void main(String[] args){
        UserService userService = new UserProxy(new UserServiceImpl());
        userService.addUser(1L, "A");
    }
}
```
输出：
```
start...
添加用户
end...
```

静态代理好处：功能增强，不改变原有代码；解耦

缺点：
1. 代理类和委托类实现同一个接口，代理类实际上调用的还是委托类里面的方法实现。如果新增一个方法，除了实现类需要实现此方法，所有的代理类也会实现此方法，会有大量重复代码。
2. 代理对象只服务于一种类型的对象，如果要服务多类型的对象，那么要创建多个代理类。

动态代理：
静态代理的缺点，代理对象只服务于一种类型的对象，并且是这编译期就已经确定被代理的对象，而动态代理是在运行时，通过反射机制，生成能够代理各种类型的对象；
在java中要实现`java.lang.reflect.InvocationHandler`接口。
基于上面的`UserService`的例子，创建一个代理类：
```
public class LogHandler implements InvocationHandler {
    //代理对象
    private Object target;

    public Object bind(Object object) {
        this.target = object;
        return Proxy.newProxyInstance(
                this.target.getClass().getClassLoader(),//指定类加载器
                this.target.getClass().getInterfaces(), //目标对象实现的接口，因为需要根据接口动态生成对象
                this);//表明拦截到的方法需要执行哪个InvocationHandler的方法，这里就是指 LogHandler类里面的 invoke方法
    }

    /**
     * @param proxy  被代理的对象
     * @param method 原对象被调用的方法
     * @param args   方法的参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Logger.start();//添加额外的方法
        //调用目标方法
        Object result = method.invoke(this.target, args);
        Logger.end();
        return result;
    }
}

```
先说反射的作用：
1. 在程序运行时构造出任意一个类的对象
2. 在程序运行时，能获取类的方法和成员变量

动态代理好处：
1.
spring动态代理