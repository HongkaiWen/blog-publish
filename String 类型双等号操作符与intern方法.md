---
title: String 类型双等号操作符与intern方法
date: 2015-08-06 08:53:50
tags:
---

等号 （ == ）操作符常常用来和equal方法比较，对于引用类型，== 操作符相当于比较内存地址，同一个类型的两个实例，用==判断结果一定是false；equal方法不同对象实现不同。
然，对于String类型做如下测试代码：

```java
String a = "abc";  
String b = "abc";  
System.out.println(a == b);  
```

变量a、b是String类型的两个变量，String非基础类型，双等号判断内存地址，结果应该是false，但这段代码输出的是true。
why？
 
针对String类型双等号，equal，intren 我设计了几个case，测试报告如下：

```log
case 1 [a-> "abc", b-> "abc"]  
a==b, result is [true]  
a.equals(b), result is [true]  
  
case 2 [a-> "abc", b-> "ab" + "c"]  
a==b, result is [true]  
a.equals(b), result is [true]  
  
case 3 [c-> "c", a-> "abc", b-> "ab" + c]  
a==b, result is [false]  
a.equals(b), result is [true]  
action [String.intern]  
a=b, result is [true]  
  
case 4 [a-> "abc", b-> new String("abc")]  
a==b, result is [false]  
a.equals(b), result is [true]  
action [String.intern]  
a=b, result is [true]  
```

这个测试结果是比较令我感到诧异的，有很多和我想象中不一样的地方。

- case1 和 case2的双等号判断，都是true，这本是不同的两个对象做双等号判断，怎么会是true呢？
- 既然case1 和case2的双等号判断是true，为什么case3 和 case4 的双等号判断又是false了呢？
- 为什么调用了intern方法之后，原来是false的判断就编程了true了呢？

根据第三点可以猜测一下，intern方法应该是会把引用互相equal的却不是同一个对象的字符串对象的变量合并成指向同一个字符串对象（此句好绕口， 用英语试试？ intern will make 2 different reference who refer to 2 different String object refer to 1 same string object）， 因为String类型是不可变对象，所以这种处理即可节省内存空间，又不会有什么问题，**因为String是不可变对象**。

java.lang.String#intern方法的非猜测解释：
这是一个native的方法，虚拟机在运行时有一块内存区域叫做“运行时常量池”，常量池用来存放编译期生成的各种**字面量**和**符号引用**。
intern方法的作用是，如果字符串常量池中包含一个等于此String对象的字符串，则返回代表池中这个字符串的对象，否则，将此String对象好汉的字符串添加到常量池中，并返回此String对象的引用。
这样不难解释为什么原本false的结果调用过intern之后就变成了true。

那么现在解释一下1 和 2 两个疑问：
前面所说，常量池用来存放编译期生成的各**种字面量**和**符号引用**， case1 和 case2 正好都是字面量，所以他们会被加入到常量池中，常量池中互相equal的字符串肯定是不允许存在的，所以相当于两个变量指向到了同一个对象。
（什么是字面量，自行百度，其实我也是看的百度，但有一个情况需要说一下：
case 2 中 a-> "abc", b-> "ab" + "c" ， b由两个字面量拼接而成，b还是字面量；
case 3 中 c-> "c", a-> "abc", b-> "ab" + c，b由一个字面量和变量c 拼接而成，b不是字面量
）

参考资料：

1. 《深入理解JAVA虚拟机》 第二版 第二章
2. http://www.cnblogs.com/zdwillie/archive/2013/10/23/3384766.html

附录：（生成测试报告的代码）

```java
public class TestString {  
    public static void main(String args[]){  
        Map<String, String> cases = new LinkedHashMap<String, String>();  
        init(cases);  
        for(Map.Entry<String, String> case_ : cases.entrySet()){  
            System.out.println(String.format("case [%s]", case_.getKey()));  
            test("abc", case_.getValue());  
            System.out.println();  
        }  
    }  
  
    private static void test(String a, String b){  
        boolean result = (a==b);  
        System.out.println(String.format("a==b, result is [%s]", result));  
        System.out.println(String.format("a.equals(b), result is [%s]",a.equals(b)));  
        if(!result){  
            System.out.println("action [String.intern]");  
            b = b.intern();  
            System.out.println(String.format("a=b, result is [%s]", a==b));  
        }  
    }  
  
    private static void init(Map<String, String> cases){  
        cases.put("a-> \"abc\", b-> \"abc\"", "abc");  
        cases.put("a-> \"abc\", b-> \"ab\" + \"c\"", "ab" + "c");  
        String c = "c";  
        cases.put("c-> \"c\", a-> \"abc\", b-> \"ab\" + c", "ab" + c);  
        cases.put("a-> \"abc\", b-> new String(\"abc\")", new String("abc"));  
    }  
}  
```
          

                  
                  
