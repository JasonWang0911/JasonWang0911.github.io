---
title: java8新特性Optional深度解析
---
```
本文以jdk1.8.0_111源码为例
```
```
public final class Optional<T> {}
```
Optional是一个为了解决NullPointerException设计而生可以包含对象也可以包含空的容器对象。封装了很多对空处理的方法也增加了filter、map这样的检索利器，其中函数式编程会有种炫酷到爆的感觉。
基础测试用例对象：
```
public class Java8OptionalTest {
    List<String> stringList = null;
    ICar car = new WeiLaiCar();
}
public class WeiLaiCar implements ICar {
    Integer wheels = new Integer(4);
｝
```
##Api中提供的4种optional
最核心的当属Optional对象，泛型的引入支持了所有对象类型，又增加对常用场景下的double\int\long进行扩展。重点介绍一下Optional对象的方法其他三个类似。

. public final class Optional<T> {

. public final class OptionalDouble {

. public final class OptionalInt {

. public final class OptionalLong {

```
@FunctionalInterface
Predicate\Consumer\Supplier三个接口都是函数式接口
```
###静态方法of
```
private Optional() {
	this.value = null;
}
```
构造方法被private，不能new但提供了of这样的静态方法去初始化类；
```
public static <T> Optional<T> of(T value) {
    return new Optional<>(value);
}

public static <T> Optional<T> ofNullable(T value) {
    return value == null ? empty() : of(value);
}

public static<T> Optional<T> empty() {
    @SuppressWarnings("unchecked")
    Optional<T> t = (Optional<T>) EMPTY;
    return t;
}
```
1、empty支持你去创建一个空的optional类，这样的类直接get()会报错：
```
java.util.NoSuchElementException: No value present
```
2、of(x)传入的对象不能为null，而ofNullable(x)是支持传入null的对象，一般用这两个比较多。

####present 方法
isPresent是用来判断optional中对象是否为null，ifPresent的参数是当对象不为null时执行的lamdba表达式。

```angular2html
public boolean isPresent() {
    return value != null;
}
public void ifPresent(Consumer<? super T> consumer) {
    if (value != null)
        consumer.accept(value);
}
```
示例详解介绍了ifPresent特性：
```angular2html
Java8OptionalTest test = new Java8OptionalTest();
Optional<Java8OptionalTest> optional = Optional.of(test);

pringTest(optional.isPresent());
//true
optional.ifPresent( a -> pringTest(a.getCar().getClass().getName()));
//com.ts.util.optional.WeiLaiCar
optional.ifPresent( a -> Optional.ofNullable(a.getStringList()).ifPresent(b -> pringTest("StringList:" + (b == null))));
//第一级的ifPresent是存在test对象，所以执行了lambda表达式，而第二级的ifPresent的stringList是null，所以没有执行表达式
optional.ifPresent( a -> Optional.ofNullable(a.getCar()).ifPresent(b -> pringTest("car:" + (b == null))));
//car:false
//第二级ifPresent的car对象是存在的，所以第二级的表达式执行了
```
####map 方法
源码提供了两种map和flatMap。

. map方法的参数是个当包含的对象不为null时才执行的lambda表达式，返回该表达式执行结果的封装optional对象，同理支持链式调用，逐层深入和递归递进很像；

. flatMap区别在于lambda表达式的返回结果必须主动包裹Optinoal，否则报错

```angular2html
public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Optional.ofNullable(mapper.apply(value));
    }
}
public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
    Objects.requireNonNull(mapper);
    if (!isPresent())
        return empty();
    else {
        return Objects.requireNonNull(mapper.apply(value));
    }
}
```
测试示例：

```angular2html
Java8OptionalTest test = new Java8OptionalTest();
Optional<Java8OptionalTest> optional = Optional.of(test);

Optional opt1 = optional.map( a -> a.getCar());
pringTest(opt1.get());
//com.ts.util.optional.WeiLaiCar@5d6f64b1
int wheel = 0;//传统null判断写法
if(test != null){
    if(test.getCar() != null){//实际业务里面层级也许会超过3层
        wheel = test.getCar().getWheelCount();
    }
}
pringTest("传统:"+wheel);
//传统:4
Optional opt2 = optional.map( a -> a.getCar()).map(b -> b.getWheelCount());//Optional支持下的写法
pringTest("optinal:"+opt2.get());
//optinal:4
Optional opt3 = optional.map( a -> a.getStringList()).map(b -> b.size());
pringTest(opt3);
//Optional.empty

Optional opt4 = optional.flatMap(a -> Optional.of(a.getCar()));//主动包裹Optional对象
pringTest(opt4);
//Optional[com.ts.util.optional.WeiLaiCar@5d6f64b1]
Optional opt5 = optional.flatMap(a -> Optional.of(a.getCar())).flatMap(b -> Optional.ofNullable(b.getWheelCount()));
pringTest(opt5);
//Optional[4]
```
####filter 方法

源码如下

```angular2html
public Optional<T> filter(Predicate<? super T> predicate) {
    Objects.requireNonNull(predicate);
    if (!isPresent())
        return this;
    else
        return predicate.test(value) ? this : empty();
}
```
filter方法传入一个断言语句条件的lambda表达式，返回一个原对象的optional包装，所以支持链式调用；只要记住这三点你便掌握如何使用了。

看下面的例子：
```angular2html
Java8OptionalTest test = new Java8OptionalTest();

Optional<Java8OptionalTest> optional = Optional.of(test);

Optional result = optional.filter( a -> a.getCar() != null).filter( b -> b.getClass().getName() != null);
pringTest(result.isPresent()? result.get().getClass().getName(): result.isPresent());
//com.ts.util.Java8OptionalTest
Optional result1 = optional.filter( a -> a.getStringList() != null);
pringTest(result1.get());
//java.util.NoSuchElementException: No value present
```
####orElse 方法

Api提供了三个方法。

. orElse 当optional内对象为null就返回这个参数，比较像很多默认值设置；

. orElseGet 基本同orElse，区别是传入参数支持lambda表达式，返回的就是表达式执行结果；

. orElseThrow 也是传入lambda表达式，但是表达式是抛出异常

```angular2html
public T orElse(T other) {
    return value != null ? value : other;
}

public T orElseGet(Supplier<? extends T> other) {
    return value != null ? value : other.get();
}

public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
    if (value != null) {
        return value;
    } else {
        throw exceptionSupplier.get();
    }
}
```
测试用例如下：

```angular2html
Java8OptionalTest one = null;
Java8OptionalTest test = new Java8OptionalTest();
Optional<Java8OptionalTest> optional = Optional.ofNullable(one);
pringTest(optional);
//Optional.empty
pringTest(optional.orElse(test));
//com.ts.util.Java8OptionalTest@5197848c
pringTest(optional.orElseGet(() -> new Java8OptionalTest()));
//com.ts.util.Java8OptionalTest@5d6f64b1
pringTest(optional.orElseThrow(() -> new RuntimeException("orElseThrow")));
//java.lang.RuntimeException: orElseThrow
```
###总结
   官方推出Optional绝不会就是替大家判断一下null，filter\map\orElse这三种使用场景是比较容易想到的，很多业务场景需要慢慢摸索使用。多函数式的用法需要好好掌握，技术发展是非常快速的。
   
   后面会专门开一篇讲函数式和Lambda表达式用法，保持好奇心关注我的博客。