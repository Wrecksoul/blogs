---
title: java传值还是传引用
categories:
- java基础
---

## 序

**经典名言：**

> O'Reilly's Java in a Nutshell by David Flanagan puts it best: 
> "Java manipulates objects 'by reference,' but it passes object references to methods 'by value.'"

*翻译：O'Reilly出版的Java in a Nutshell的作者David Flanagan说：java通过引用操作对象，但是在java中，向方法传递引用时传递的是（引用的）值*

这个老生常谈的问题到了2017年其实没必要提出来了，但是竟然有人说这种说法是**故弄玄虚**。今天就分析一下到底是不是故弄玄虚。
<!--more-->
## 直接上图

### 基础类型
![](http://ozhp30d7a.bkt.clouddn.com/image/java%E4%BC%A0%E5%80%BC%E8%BF%98%E6%98%AF%E4%BC%A0%E5%BC%95%E7%94%A8/%E5%9F%BA%E7%A1%80%E7%B1%BB%E5%9E%8B%E7%9A%84%E4%BC%A0%E9%80%92.png)
如上图所示，基础类型在函数传参时是值传递的，`i`的值是1，方法获取的参数是形参`i'`改动`i'`，不会对`i`造成任何影响。

虽然基础类型的传递没什么疑问（一准是值传递），为了完整起见还是提一下。

### 可变对象引用
![](http://ozhp30d7a.bkt.clouddn.com/image/java%E4%BC%A0%E5%80%BC%E8%BF%98%E6%98%AF%E4%BC%A0%E5%BC%95%E7%94%A8/%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%E7%9A%84%E4%BC%A0%E9%80%92.png)
如上图所示，引用`o`传入方法后，形参是`o'`。

对于第一种改动：
对象`obj11`在方法指向期间就拥有了两个入口都可以对它操作，并且两个引用的改动是互相可见互相影响的。

对于第二种改动：
引用`o'`只要不是final类型，就可以改动引用的指向。改动了o'的指向后，方法运行期间，用户可以通过o'获取`obj12`的内容，而不再是`obj11`，但是这种改动不会影响到`obj11`。

### 不可变对象引用
![](http://ozhp30d7a.bkt.clouddn.com/image/java%E4%BC%A0%E5%80%BC%E8%BF%98%E6%98%AF%E4%BC%A0%E5%BC%95%E7%94%A8/%E4%B8%8D%E5%8F%AF%E5%8F%98%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%BC%95%E7%94%A8%E7%B1%BB%E5%9E%8B%E4%BC%A0%E9%80%92.png)

如上图所示，引用`o`传入方法后，形参是`o'`。

对不可变对象作出修改时，类似于`可变对象引用`中的第二种改动，用户需要改动不可变对象时，由于对象不可变，改动后的对象其实是一个新的对象，就像图上话的那样，想把`obj21`的内容变化成2222，`obj21`不会改变，而是生成了新的、一个内容是2222的`obj22`，并且引用`o'`指向了新的对象。这种改动也不会影响到原始的引用`o`。

## 上代码
```java
public class StringBufferDemo {
    public static void main(String[] args) {
        String s1 = "hello";
        String s2 = "world";
        System.out.println(s1 + "---" + s2);// hello---world
        change(s1, s2);
        System.out.println(s1 + "---" + s2);// hello---world

        StringBuffer sb1 = new StringBuffer("hello");
        StringBuffer sb2 = new StringBuffer("world");
        System.out.println(sb1 + "---" + sb2);// hello---world
        change(sb1, sb2);
        System.out.println(sb1 + "---" + sb2);// hello---worldworld

    } 

    public static void change(StringBuffer sb1, StringBuffer sb2) {
        sb1 = sb2;
        sb2.append(sb1);
    }

    public static void change(String s1, String s2) {
        s1 = s2;
        s2 = s1 + s2;
    }
}
```

上面的例子有StringBuffer和String两种，先看StringBuffer，这是一个普通的可变对象，显然在`change`方法改变了sb1的指向，让它指向了一个sb2指向的对象，但是并没有对之前的对象造成任何影响。而`sb2.append(sb1)`改变了sb2的内容。
其实只看这一个例子就能看出来，为什么sb1没有变化呢？如果按照传引用的话，引用的指向被改动了，那么在函数调用之前的应用也应该被改动了指向才对，然而并没有。也就是说，引用在被传递的时候，也是把引用的值传过去了。
```
O'Reilly's Java in a Nutshell by David Flanagan (see Resources) puts it best: "Java manipulates objects 'by reference,' but it passes object references to methods 'by value.'"
```

再看String的那个例子，`s2 = s1 + s2;`这个赋值语句其实是改动了s2的指向，s1+s2产生了一个新的对象赋值给s2这个引用，这个赋值也只是作用于函数内部的这个s2引用的复制而已。

## 容易混淆的一点

很多人看了一些文章之后理解了，但是后来在使用的时候又开始产生疑问。我的总结：没有弄清**“传值还是传引用”**这个问题提出时的上下文——**函数调用**。

```java
public class ImmuableTest {
	private Integer a = 10000;
	private Object o = null;
	
    public void setA(){
		a = 9999;
	}
	public void setO(){
		o = new Object();
	}
}
```
上面这种情况不需要考虑传值还是传引用，因为`setA()`和`setO()`使用的就是原始引用，根本没有引用的传递！！

是的，在java中，如果你想让一个方法拿到原始的引用，而不是一个原始引用的复制品的话，那你就只能通过这种使用类的成员变量的方式，而不能通过函数传值。

## 结论

1. java函数传参——只会传值，即使你想传递的是引用，那也是传递的引用的值。

2. 这种说法不是矫情，如果不这么理解的话，对于不可变对象的理解会出问题。

## 太多弯弯绕？

总结：
~~传基础类型和不可变对象时，方法内部的改动不会改变源~~
~~传递可变对象引用时，方法内部的改动会改变源~~
虽然说法是对的但是很不推荐这种记忆方式。

况且不可变对象也并非完全不可变，比如可以通过反射的方式修改String类型的对象内部保存的数组这种达到改变所谓的不可变的对象。

常见的产生不可变对象的类有：
String，Number的子类Integer、Long、Double、Float、Boolean、Short、Byte、BigDecimal、BigInteger。