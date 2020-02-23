# JUC-AQS



juc包下提供了很多线程安全的类或者接口，可以直接提供给开发者使用，比如`ReentrantLock`、`CountDownLatch`等等，很多接口都是基础aqs实现的，所以在学习juc并发包下的源码前，有必要先了解一下什么是aqs。

AQS全名：`AbstractQueuedSynchronizer`抽象队列同步器，是`java.util.concurrent.locks`包下的一个抽象类。