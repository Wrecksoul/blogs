---
title: java8——Lambda和匿名内部类
categories:
- java基础
tags:
- java8
- 函数式编程
---

java8加入的函数式编程一大主力之一——Lambda表达式。

java经常被人们诟病的一个问题就是啰嗦，表达能力欠佳。而Lambda表达式的加入可以一定程度的弥补这一缺陷。
<!-- more -->

## java啰嗦的地方
1. java中的方法是不可以独立存在的，必须在类或者接口中定义，这就使得java比较啰嗦，当我们仅仅需要定义一个方法时必须先定义一个类。
2. java中不存在函数指针，所以函数无法被传递。

js中可以这样写。

```javascript
var al = function(i){
	return i<100;
}
var bl = function(i){
	return i==0;
}
function dealwithVal(val,judge){
	if(judge(val)){
		//dealwith..
	}
}
var val = 100;//get from somewhere
dealwithVal(val,cl);
dealwithVal(val,al);
dealwithVal(val,function(a){
	return a>8&&a<100;
});

```
这实现了把行为参数化， `判断` 这个行为函数，被当做一个参数传递进入另一个函数，而且判断行为可以在使用时再具体传入，甚至可以临时定义一个传递进去，这样大量的减少了代码量，语言的表达能力大大提高。

但是，java中真的不能把方法作为参数传递吗？答案是可以。

对于上面的需求我们可以给出一个java版本的实现,方法就是利用接口。通常我们这样实现：

```java
interface Judge<T>{
	boolean test(T t);
}
class JudgeWayA implements Judge<Integer>{
	@Override
	public boolean test(Integer t) {
		return t.compareTo(100)<0;//t<100
	}
}
class JudgeWayB implements Judge<Integer>{
	@Override
	public boolean test(Integer t) {
		return t.compareTo(8)>0;//t>8
	}
}
public class DealWith{
	private Judge judge;
	public DealWith(Judge judge){
		this.judge = judge;
	}
	public static void main(String[] args) {
		int val = 100; //get from somewhere
		new DealWith(new JudgeWayA()).deal(val);
		new DealWith(new JudgeWayB()).deal(val);
		new DealWith(new Judge<Integer>(){
			@Override
			public boolean test(Integer t) {
				return t.compareTo(8) > 0 && t.compareTo(100) < 0;
			}
		}).deal(val);
	}
	public void deal(int val){
		if(judge.test(val)){
			//dealwith..
		}
	}
}
```
## 行为参数化

这是我们平时在设计模式中成为 **策略模式** 的一种编码模式，可以实现 **行为** 的变化。将变化的行为一定程度的固定下来，但是留有可以变化的余地，通过传递不同的实现达到需求变更时最小程度代码变更的目的。

在java中，传递一个函数或者说传递一种行为，的方式是传递一个接口，面向接口编程其实是**面向变化的行为**编程，是将**行为参数化**的java实现方式。

但是这种实现有点太麻烦了。很明显，相比于JS语言我们需要额外的定义一个方法签名及对应接口类、实现方法签名并声明对应类。

虽然可以实现**行为的参数化**但是面向接口编程还是很啰嗦，而匿名内部类的使用进一步减少了接口的实现类的书写长度，但是仍然面临着不够简洁的问题。

行为参数化可以利用接口和匿名内部类，但是使用起来比较麻烦，java8的Lamda表达式则进一步简化了书写方式。

## 小结

1. java要实现行为参数化需要使用接口
2. 行为参数化最简洁的方式是匿名内部类
3. 匿名内部类还不够简洁

## 简写匿名内部类

### 是行为而不是声明

上面的`Judge`接口，它只有一个方法，而`DealWith`这个类需要的就是一个行为，这个行为可以通过`Judge`接口里面的方法实现定义。换句话说，我们其实不关心在实现`Judge`这个接口时关于接口类型、方法返回值、方法参数等等声明，这些声明早已在接口中定义过了，我们只关心接口方法的实现的 **逻辑** 是什么，只要知道了方法逻辑，我们就可以让DealWith执行下去了，而接口的定义以及声明格式完全是为了让jvm认得我们书写的逻辑是属于哪个接口的，是jvm要求我们这样去写才带来的这许多麻烦。要是它能认出、推断出我们的代码就是实现的哪一个接口的，那我们就不要写那么多**声明**了。

根据上面的代码逻辑，我们可以知道：
1. **判断** 这样一个行为，需要的是返回一个boolean返回值。
2. 判断的具体逻辑是什么并不确定。
3. 虽然具体逻辑不确定，但是判断时参数是确定的，就是一个整型值。

其实上面这些问题，正是接口定义时定义的，既然接口已经定义了，我们还需要重复定义吗？比如：
```java
public class JudgeWayA implements Judge{
	@override
	boolean test(int i){
		return i<100;
	}
}
public class JudgeWayB implements Judge{
	@override
	boolean test(int i){
		return i==0;
	}
}
```
这属于重复定义，看看上面的代码，有效的只有`return i==0;`和`return i<100;`这两句，其他的代码跟`Judge`里面的代码重复了。

而匿名内部类省略了一小部分书写的代码：
```java
Judge j= new Judge(){
	@override
	boolean test(int a){
		return a>8 && a<100;
	}
}
new DealWith(j).deal();
```
但是还可以更简洁。

### 类型推断

类似于菱形符号的加入：`List<String> l = new ArrayList<>();`java7引入了这种书写方式，这里使用了类型的自动推断。那么匿名内部类不也可以自动推断吗。

当我们书写new DealWith(xxxx)的时候，编译器就应该知道，这里就是一个Judge类型的对象，因为我们已经定义了DealWith的单参构造方法：`public DealWith(Judge judge)`。

所以，Lambda表达式完全可以看成是匿名内部类的一种类型推断。
```java
Judge j = (int a)-> a>8 && a<100;
new DealWith(j).deal();
```
更进一步，因为参数的类型我们也定义过了干脆不写：
```
Judge j = (a)-> a>8 && a<100;
new DealWith(j).deal();
```
或者直接写成：`new DealWith((a)-> a>8 && a<100).deal()`

### 突出本质

Lambda表达式的真谛就是突出本质，其实Lambda并没有那么特别，还是需要定义一个接口，接口限制为只能有一个方法，然后就可以将匿名函数改写成Lambda表达式了。

特务接头的时候，双方一般需要互相确认，只有确认完毕后，才进行情报交换，比如：“地振高冈，一派青山千古秀。”这是让对方知道自己是天地会的，“门朝大海,三河合水万年流”，这是确认对方是天地会的。类比到java中就是接口声明了要使用某种参数，这是第一步，使用接口时再次声明参数的类型符合接口的要求，这是第二部。

对于韦小宝来说，一旦对方说出前一句，那就可以了，根本不需要说后一句，效率高也省得麻烦。当然了这是有一定风险的，但是带来的好处是效率高。

再比如，TCP连接是需要确认双方都可以收到消息的，相对来说比较慢，而UDP不需要确认，给个端口就发送数据，效率高。但是带来一定的丢失消息的风险。

### 风险和回报

就java来说，此处的风险是什么呢？就是两个函数式接口定义的方法、以及方法的调用处写了两种都写了，导致无法推断出到底是哪个接口的表达式。

比如我们上面的`Judge`接口定义的跟java8提供的`Predicate`抽象方法签名一样，那我们的方法就会有点问题了。

```java
@FunctionalInterface
public interface Predicate<T> {
	boolean test(T t);
	//...还有很多default方法..省略
}
```
如上图我们的定义时一样的。如果
```java
public class DealWith{
	private Judge<Integer> judge;
	private Predicate<Integer> predicate;
	public Func(Judge<Integer> judge){
		this.judge = judge;
	}
	public Func(Predicate<Integer> predicate){
		this.predicate = predicate;
	}
	//...省略
}
```
那么此时，`new DealWith((a)-> a>8 && a<100).deal()`就会报错，因为此时产生了歧义——`The constructor Func(Judge<Integer>) is ambiguous`

**但是**，这又有什么关系呢？如果一个方法签名是一样的，那效果就会一模一样，我们只需要去掉一个就可以了。

可以说如此做法带来的回报远远高于风险。

### java8的改造

一直都存在于java中的接口们可以在java8中重整旗鼓再焕发一次了，比如Runnable接口只有一个抽象方法，符合函数式接口的要求。所以以前我们这样写：

```java
new Thread(new Runnable(){
	@override
	public void run() {
		System.out.println("hello");
	}
});
```

Lambda的表达方式是：
```java
new Thread( ()->System.out.println("hello") );
```

这样不仅实现了简洁，如果我们熟悉了这种写法之后，代码的可读性也会提高。

## Lambda不是匿名内部类的简写

看上去Lambda表达式就是匿名内部类的简写，可以替代匿名内部类，其实不是：
1. Lambda只能代表重写一个接口方法的匿名内部类；
2. Lambda表达式的jvm执行方式与匿名内部类不同。

如果你使用匿名内部类，其实只是没有书写一个类的名字而已，与一般的内部类一样，编译时会生成一个额外的class文件，而Lambda表达式则不会产生额外的class文件。

```java
public class MainLambda {
	public static void main(String[] args) {
		new Thread(() -> System.out.println("Lambda Thread run()")).start();
	}
}
```

反编译之后我们发现Lambda表达式被封装成了主类的一个私有方法，并通过invokedynamic指令进行调用(参考：https://github.com/CarpenterLee/JavaLambdaInternals/blob/master/2-Lambda%20and%20Anonymous%20Classes(II).md)

```java
// javap -c -p MainLambda.class
public class MainLambda {
  ...
  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class java/lang/Thread
       3: dup
       4: invokedynamic #3,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable; /*使用invokedynamic指令调用*/
       9: invokespecial #4                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
      12: invokevirtual #5                  // Method java/lang/Thread.start:()V
      15: return

  private static void lambda$main$0();  /*Lambda表达式被封装成主类的私有方法*/
    Code:
       0: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #7                  // String Lambda Thread run()
       5: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

## this引用的意义
既然Lambda表达式不是内部类的简写，那么Lambda内部的this引用也就跟内部类对象没什么关系了。在Lambda表达式中this的意义跟在表达式外部完全一样。因此下列代码输，而不是两个引用地址。

```java
public class Main {
	public static void main(String[] args) {
		new Main().deal();
	}
	public void deal() {
		Runnable r1 = () -> { System.out.println(this); };
		Runnable r2 = new Runnable() {
			@Override
			public void run() {
				System.out.println(this);
			}
			@Override
			public String toString() {
				return "Hello Runnable";
			}
		};
		r1.run();
		r2.run();
	}
	public String toString() { return "Hello Lambda"; }
}
```
输出：
> Hello Lambda
> Hello Runnable

r1没有输出引用地址，因为它不是一个类，而属于Main的私有方法，所以this指向的是Main的实例，而Main的toString输出`Hello Lambda`

r2输出`Hello Runnable`，因为它自己是一个类，this指向匿名内部类的实例。

