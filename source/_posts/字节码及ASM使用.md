---
title: 字节码及ASM使用
date: 2017-03-24 12:47:19
categories: 
- 编程
- java
tags:
- ASM
- JVM
- java
---
## 什么是字节码？
* 机器码
机器码(machine code)是CPU可直接解读的指令。机器码与硬件等有关，不同的CPU架构支持的硬件码也不相同。

* 字节码
字节码（bytecode）是一种包含执行程序、由一序列 op 代码/数据对 组成的二进制文件。字节码是一种中间码，它比机器码更抽象，需要直译器转译后才能成为机器码的中间代码。通常情况下它是已经经过编译，但与特定机器码无关。字节码主要为了实现特定软件运行和软件环境、与硬件环境无关。
字节码的实现方式是通过编译器和虚拟机器。编译器将源码编译成字节码，特定平台上的虚拟机器将字节码转译为可以直接执行的指令。
例如:C# IL,Java bytecode
    * JAVA代码编译和执行
    Java 代码编译是由 Java 源码编译器来完成，流程图如下所示：
    {% asset_img img1.gif %}
    Java 字节码的执行是由 JVM 执行引擎来完成，流程图如下所示：
    {% asset_img img2.gif %}

<!-- more -->
## JVM字节码执行

### JVM桢栈结构
以下内容可以参照示例阅读，理解会不一样
>方法调用在JVM中转换成的是字节码执行，字节码指令执行的数据结构就是栈帧（stack frame）。也就是在虚拟机栈中的栈元素。虚拟机会为每个方法分配一个栈帧，因为虚拟机栈是LIFO(后进先出)的，所以当前线程正在活动的栈帧，也就是栈顶的栈帧，JVM规范中称之为“CurrentFrame”,这个当前栈帧对应的方法就是“CurrentMethod”。字节码的执行操作，指的就是对当前栈帧数据结构进行的操作。
>  JVM的运行时数据区的结构如下图，本文主要讲桢栈结构。
>{% asset_img img3.png %}
>
>运行时数据区
>
>  栈帧的数据结构主要分为四个部分：局部变量表、操作数栈、动态链接以及方法返回地址（包括正常调用和异常调用的完成结果）。下面就一一介绍下这四种数据结构。
>
>局部变量表（local variables）
>
>  当方法被调用时，参数会传递到从0开始的连续的局部变量表的索引位置上。栈帧中局部变量表的长度存储在类或接口的二进制表示中。阅读Class文件会找到Code属性，所以能知道local variables的最大长度是在编译期间决定的。<b><font color=FFA500>一个局部变量表的占用了32位的存储空间（一个存储单位称之为slot，槽），所以可以存储一个boolean、byte、char、short、float、int、refrence和returnAdress数据，long和double需要2个连续的局部变量表来保存，通过较小位置的索引来获取。如果被调用的是实例方法，那么第0个位置存储“this”关键字代表当前实例对象的引用。</font></b>这个可以通过javap 工具查看实例方法和静态方法对比字节码指令的0位置。例子也可以参考JVM 字节码指令对于栈帧数据操作举例
>
>操作数栈（operand stack）
>
>  操作数栈同局部变量表一样，也是编译期间就能决定了其存储空间（最大的单位长度），通过 Code属性存储在类或接口的字节流中。操作数栈也是个LIFO栈。 
>  操作数栈是在JVM字节码执行一些指令（第二部分会介绍一些指令集）时创建的，主要是把局部变量表中的变量压入操作数栈，在操作数栈中进行字节码指令的操作，再将变量出操作数栈，结果入操作数栈。<b><font color=FFA500>同局部变量表,除了long和double,其他类型数据都只占用一个栈的单位深度。</font></b>
>
>动态链接
>
>  每个栈帧指向运行时常量池中该栈帧所属的方法的引用，也就是字节码的发放调用的引用。动态链接就是将符号引用所表示的方法，转换成方法的直接引用。加载阶段或第一次使用时转化为直接引用的（将变量的访问转化为访问这些变量的存储结构所在的运行时内存位置）就叫做静态解析。JVM的动态链接还支持运行期转化为直接引用。也可以叫做Late Binding,晚期绑定。动态链接是java灵活OO的基础结构。可以参考一个例子来加深理解从字节码指令看重写在JVM中的实现
>
>方法返回地址
>
>  <b><font color=FFA500>方法正常退出，JVM执行引擎会恢复上层方法局部变量表操作数栈并把返回值压入调用者的栈帧的操作数栈，PC计数器的值就会调整到方法调用指令后面的一条指令。</font></b>这样使得当前的栈帧能够和调用者连接起来，并且让调用者的栈帧的操作数栈继续往下执行。 
>  方法的异常调用完成，主要是JVM抛出的异常，如果异常没有被捕获住，或者遇到athrow字节码指令显示抛出，那么就没有返回值给调用者。

<b><font color=FFA500>
注:

* 操作long double类型数据，一定要分配两个slot，如果分配了一个slot不会报错，但会导致数据丢失;
* 实例方法的第0个位置存储“this”关键字代表当前实例对象的引用;
* 引用自[JVM字节码执行模型及字节码指令集](https://www.atatech.org/articles/42062#)，建议阅读《深入了解JVM》-- 虚拟机执行子系统;

</font></b>

### 字节码指令集
以下的字节码指令集比较枯燥，可参照ASM Opcodes
>加载和存储指令
>加载和存储指令用于将数据从栈帧的局部变量表和操作数栈之间来回传输。
>        1)将一个局部变量加载到操作数栈的指令包括：iload,iload_<n\>，lload、lload_<n\>、float、 fload_<n\>、dload、dload_<n\>，aload、aload_<n\>。
>        2)将一个数值从操作数栈存储到局部变量表的指令：istore,istore_<n\>,lstore,lstore_<n\>,fstore,fstore_<n\>,dstore,dstore_<n\>,astore,astore_<n\>
>        3)将常量加载到操作数栈的指令：bipush,sipush,ldc,ldc_w,ldc2_w,aconst_null,iconst_ml,iconst_<i\>,lconst_<l\>,fconst_<f\>,dconst_<d\>
>        4)局部变量表的访问索引指令:wide
>一部分以尖括号结尾的指令代表了一组指令，如iload_<i\>，代表了iload_0,iload_1等，这几组指令都是带有一个操作数的通用指令。
>
>运算指令
>算术指令用于对两个操作数栈上的值进行某种特定运算，并把结果重新存入到操作栈顶。
>        1)加法指令:iadd,ladd,fadd,dadd
>        2)减法指令:isub,lsub,fsub,dsub
>        3)乘法指令:imul,lmul,fmul,dmul
>        4)除法指令:idiv,ldiv,fdiv,ddiv
>        5)求余指令:irem,lrem,frem,drem
>        6)取反指令:ineg,leng,fneg,dneg
>        7)位移指令:ishl,ishr,iushr,lshl,lshr,lushr
>        8)按位或指令:ior,lor
>        9)按位与指令:iand,land
>        10)按位异或指令:ixor,lxor
>        11)局部变量自增指令:iinc
>        12)比较指令:dcmpg,dcmpl,fcmpg,fcmpl,lcmp
>Java虚拟机没有明确规定整型数据溢出的情况，但规定了处理整型数据时，只有除法和求余指令出现除数为0时会导致虚拟机抛出异常。
>Java虚拟机要求在浮点数运算的时候，所有结果否必须舍入到适当的精度，如果有两种可表示的形式与该值一样，会优先选择最低有效位为零的。称之为最接近数舍入模式。
>浮点数向整数转换的时候，Java虚拟机使用IEEE 754标准中的向零舍入模式，这种模式舍入的结果会导致数字被截断，所有小数部分的有效字节会被丢掉。
>
>类型转换指令
>类型转换指令将两种Java虚拟机数值类型相互转换，这些操作一般用于实现用户代码的显式类型转换操作。
>JVM直接就支持宽化类型转换(小范围类型向大范围类型转换)：
>        1)int类型到long,float,double类型
>        2)long类型到float,double类型
>        3)float到double类型
>但在处理窄化类型转换时，必须显式使用转换指令来完成，这些指令包括：i2b、i2c、i2s、l2i、f2i、f2l、d2i、d2l和 d2f。
>将int 或 long 窄化为整型T的时候，仅仅简单的把除了低位的N个字节以外的内容丢弃，N是T的长度。这有可能导致转换结果与输入值有不同的正负号。
>在将一个浮点值窄化为整数类型T（仅限于 int 和 long 类型），将遵循以下转换规则：
>        1）如果浮点值是NaN ， 呐转换结果就是int 或 long 类型的0
>        2）如果浮点值不是无穷大，浮点值使用IEEE 754 的向零舍入模式取整，获得整数v， 如果v在T表示范围之内，那就过就是v
>        3）否则，根据v的符号， 转换为T 所能表示的最大或者最小正数
>
>对象创建与访问指令
>虽然类实例和数组都是对象，Java虚拟机对类实例和数组的创建与操作使用了不同的字节码指令。
>        1)创建实例的指令:new
>        2)创建数组的指令:newarray,anewarray,multianewarray
>        3)访问字段指令:getfield,putfield,getstatic,putstatic
>        4)把数组元素加载到操作数栈指令:baload,caload,saload,iaload,laload,faload,daload,aaload
>        5)将操作数栈的数值存储到数组元素中执行:bastore,castore,castore,sastore,iastore,fastore,dastore,aastore
>        6)取数组长度指令:arraylength JVM支持方法级同步和方法内部一段指令序列同步，这两种都是通过moniter实现的。
>        7)检查实例类型指令:instanceof,checkcast
>
>操作数栈管理指令
>如同操作一个普通数据结构中的堆栈那样，Java 虚拟机提供了一些用于直接操作操作数栈的指令，包括：
>        1）将操作数栈的栈顶一个或两个元素出栈：pop、pop2
>        2）复制栈顶一个或两个数值并将复制值或双份的复制值重新压入栈顶：dup、dup2、dup_x1、dup2_x1、dup_x2、dup2_x2。
>        3）将栈最顶端的两个数值互换：swap
>
>控制转移指令
>让JVM有条件或无条件从指定指令而不是控制转移指令的下一条指令继续执行程序。控制转移指令包括：
>        1)条件分支:ifeq,iflt,ifle,ifne,ifgt,ifge,ifnull,ifnotnull,if_cmpeq,if_icmpne,if_icmlt,if_icmpgt等
>        2)复合条件分支:tableswitch,lookupswitch
>        3)无条件分支:goto,goto_w,jsr,jsr_w,ret
>
>JVM中有专门的指令集处理int和reference类型的条件分支比较操作，为了可以无明显标示一个实体值是否是null,有专门的指令检测null 值。boolean类型和byte类型,char类型和short类型的条件分支比较操作，都使用int类型的比较指令完成，而 long,float,double条件分支比较操作，由相应类型的比较运算指令，运算指令会返回一个整型值到操作数栈中，随后再执行int类型的条件比较操作完成整个分支跳转。各种类型的比较都最终会转化为int类型的比较操作。
>
>方法调用和返回指令
>invokevirtual指令:调用对象的实例方法，根据对象的实际类型进行分派(虚拟机分派)。
>invokeinterface指令:调用接口方法，在运行时搜索一个实现这个接口方法的对象，找出合适的方法进行调用。
>invokespecial:调用需要特殊处理的实例方法，包括实例初始化方法，私有方法和父类方法
>invokestatic:调用类方法(static)
>方法返回指令是根据返回值的类型区分的，包括ireturn(返回值是boolean,byte,char,short和 int),lreturn,freturn,drturn和areturn，另外一个return供void方法，实例初始化方法，类和接口的类初始化i方法使用。
>
>异常处理指令
>在Java程序中显式抛出异常的操作（throw语句）都有athrow 指令来实现，除了用throw 语句显示抛出异常情况外，Java虚拟机规范还规定了许多运行时异常会在其他Java虚拟机指令检测到异常状况时自动抛出。
>在Java虚拟机中，处理异常不是由字节码指令来实现的，而是采用异常表来完成的。
>
>同步指令
>方法级的同步是隐式的，无需通过字节码指令来控制，它实现在方法调用和返回操作中。虚拟机从方法常量池中的方法标结构中的 ACC_SYNCHRONIZED标志区分是否是同步方法。方法调用时，调用指令会检查该标志是否被设置，若设置，执行线程持有moniter，然后执行方法，最后完成方法时释放moniter。
>同步一段指令集序列，通常由synchronized块标示，JVM指令集中有monitorenter和monitorexit来支持synchronized语义。
结构化锁定是指方法调用期间每一个monitor退出都与前面monitor进入相匹配的情形。JVM通过以下两条规则来保证结结构化锁成立(T代表一线程，M代表一个monitor)：
>        1)T在方法执行时持有M的次数必须与T在方法完成时释放的M次数相等
>        2)任何时刻都不会出现T释放M的次数比T持有M的次数多的情况
        
<b><font color=FFA500>
注：

* 大多数的指令有前缀和（或）后缀来表明其操作数的类型。如下表

    | 前/后缀 | 操作数类型 |
    | ------| ------ |
    | i | 整数 | 
    | l | 长整数 | 
    | s | 短整数 | 
    | b | 字节 | 
    | c | 字符 | 
    | f | 单精度浮点数 | 
    | d | 双精度浮点数 | 
    | z | 布尔值 | 
    | a | 引用 | 

* 引用自[深入理解java虚拟机 字节码指令简介](http://blog.csdn.net/zq602316498/article/details/38847935)
</font></b>

### 示例:
JAVA源码

``` java
package com.taobao.film;

/**
 * @author - zhupin(kaiqiang.gkq@alibaba-inc.com)
 */
public class Demo {
    private static final String HELLO_CONST = "Hello";
    private static String CONST = null;

    static {
        CONST = HELLO_CONST + "%s!";
    }

    public static void main(String[] args) {
        if (args != null && args.length == 1) {
            System.out.println(String.format(CONST, args[0]));
        }
    }
}
```
对应字节码
简记 ms:操作数栈最大深度 ml:局部变量表最大容量 s:操作数栈 l:局部变量表

``` java
// class version 49.0 (49)
// access flags 0x21
public class com/taobao/film/Demo {

  // compiled from: Demo.java

  // access flags 0x1A 常量,类被装载时分配空间 private static final String HELLO_CONST = "Hello";
  private final static Ljava/lang/String; HELLO_CONST = "Hello"

  // access flags 0xA 静态属性,类被装载时分配空间 private static String CONST ;
  private static Ljava/lang/String; CONST

  // access flags 0x1 没有重载构造函数，默认构造函数<init>()V
  public <init>()V
   L0 // 标签表示方法的字节码中的位置。标签用于跳转，goto和切换指令，以及用于尝试catch块。标签指定刚刚之后的指令。注意，在标签和它指定的指令（例如其他标签，堆栈映射帧，行号等）之间可以有其他元素。
    LINENUMBER 6 L0 //异常的栈信息中对应的line number，可删除不影响运行
    ALOAD 0 //this
    INVOKESPECIAL java/lang/Object.<init> ()V //this.super()
    RETURN
   L1
    LOCALVARIABLE this Lcom/taobao/film/Demo; L0 L1 0
    MAXSTACK = 1 //最大栈深度1,压栈局部变量this
    MAXLOCALS = 1 //局部变量数1,局部变量this

  // access flags 0x9 简记 ms:操作数栈最大深度 ml:局部变量表最大容量 s:操作数栈 l:局部变量表
  public static main([Ljava/lang/String;)V
   L0
    LINENUMBER 15 L0
    ALOAD 0 //非静态方法,局部变量0即方法入参第一个参数args reference; ms=1,ml=1,s=[args ref],l=[args ref]
    IFNULL L1  //if(args ref==null)跳转 L1 return
    ALOAD 0 //load args ms=1,ml=1,s=[args ref],l=[args ref]
    ARRAYLENGTH//计算args.length将结果入操作数栈 ms=1,ml=1,s=[args ref],l=[args ref] 
    ICONST_1 //数字常量1 ms=2,ml=1,s=[1,args ref],l=[args ref] 
    IF_ICMPNE L1 //if(args.length == 1) 跳转 L1 return 
   L2 //ms=2,ml=1,s=[],l=[args ref] 
    LINENUMBER 16 L2
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;//访问System.out，入操作数栈 ms=2,ml=1,s=[System.out],l=[args ref]  
    GETSTATIC com/taobao/film/Demo.CONST : Ljava/lang/String;//访问CONST，入操作数栈 ms=2,ml=1,s=[CONST,System.out],l=[args ref]  
    {//这里一段，实现的是new Object[]{args[0]},String.format(String format, Object... args) 
    ICONST_1//1  ms=3,ml=1,s=[1,CONST,System.out],l=[args ref]  
    ANEWARRAY java/lang/Object// new Object[1]; ms=3,ml=1,s=[objects ref,CONST,System.out],l=[args ref]  
    DUP // ms=4,ml=1,s=[objects ref,objects ref,CONST,System.out],l=[args ref]  
    ICONST_0//0 ms=5,ml=1,s=[0,objects ref,objects ref,CONST,System.out],l=[args ref]  
    ALOAD 0 //args reference入操作数栈; ms=6,ml=1,s=[args ref,0,objects ref,objects ref,CONST,System.out],l=[args ref]  
    ICONST_0 //0 ms=7,ml=1,s=[0,args ref,0,objects ref,objects ref,CONST,System.out],l=[args ref] 
    AALOAD //args[0] ms=7,ml=1,s=[args[0],0,objects ref,objects ref,CONST,System.out],l=[args ref]
    AASTORE //objects[0]=args[0]; ms=7,ml=1,s=[objects ref,CONST,System.out],l=[args ref]
    }
    INVOKESTATIC java/lang/String.format (Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;//String.format(CONST,objects) ms=7,ml=1,s=[objects ref,CONST,System.out],l=[args ref]
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V //System.out.println(...); ms=7,ml=1,s=[],l=[args ref]
   L1
    LINENUMBER 18 L1
    RETURN
   L3
    LOCALVARIABLE args [Ljava/lang/String; L0 L3 0
    MAXSTACK = 7
    MAXLOCALS = 1

  // access flags 0x8 <clinit>V classload init，静态代码初始块，类被装在时init
  static <clinit>()V
   L0
    LINENUMBER 8 L0 
    ACONST_NULL //压栈常量NULL
    PUTSTATIC com/taobao/film/Demo.CONST : Ljava/lang/String; //静态属性赋值，CONST=null
   L1
    LINENUMBER 11 L1
    LDC "Hello%s!" //加载常量Hello%s! 编译优化，hello+%s编译优化为常量Hello%s!
    PUTSTATIC com/taobao/film/Demo.CONST : Ljava/lang/String;//赋值Hello%s!
   L2
    LINENUMBER 12 L2
    RETURN
    MAXSTACK = 1 //只有赋值操作，操作数栈深度1
    MAXLOCALS = 0 //静态方法无局部变量
}
```

## 借助字节码能干什么
回答这个问题前我们首先得明白字节码是作用于运行期，所以一般用来在运行期改变类的行为，基本上都会结合代理模式使用。

### 常见字节码框架
{% asset_img img4.png %}

* [ASM](http://asm.ow2.org/)
* [BCEL](http://commons.apache.org/proper/commons-bcel/)
* [CGLIB](https://github.com/cglib/cglib)
* [Javassist](http://jboss-javassist.github.io/javassist/)
* [Byte Buddy](http://bytebuddy.net/#/)

本文来介绍下ASM框架的使用(了解一点底层字节码操作，对了解JVM执行过程以及我们常用框架的实现原理都会有新的认识)
{% asset_img img5.jpg %}
ClassReader用来读取原有的字节码，ClassWriter用于写入字节码，ClassVisitor、FieldVisitor、MethodVisitor、AnnotationVisitor访问修改对应组件。(一般不要通过ClassReader读取再通过Vistor修改类型为再ClassWriter回写，覆盖原有类行为，采用继承会更安全)

注: 

* [Diving into Bytecode Manipulation](https://blog.newrelic.com/2014/09/29/diving-bytecode-manipulation-creating-audit-log-asm-javassist)
* [字节码操纵技术探秘
](http://www.infoq.com/cn/articles/Living-Matrix-Bytecode-Manipulation)

### 使用示例
以下代码效果等同于上面的Demo示例

``` java
import java.io.FileOutputStream;
import java.lang.reflect.Method;

import org.objectweb.asm.AnnotationVisitor;
import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.FieldVisitor;
import org.objectweb.asm.Label;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Opcodes;

public class DemoDump implements Opcodes {

    public static byte[] dump() throws Exception {

        ClassWriter cw = new ClassWriter(0);
        FieldVisitor fv;
        MethodVisitor mv;
        AnnotationVisitor av0;

        cw.visit(V1_5, ACC_PUBLIC + ACC_SUPER, "com/taobao/film/Demo", null, "java/lang/Object", null);

        cw.visitSource("Demo.java", null);

        {
            fv = cw.visitField(ACC_PRIVATE + ACC_FINAL + ACC_STATIC, "HELLO_CONST", "Ljava/lang/String;", null,
                "Hello");
            fv.visitEnd();
        }
        {
            fv = cw.visitField(ACC_PRIVATE + ACC_STATIC, "CONST", "Ljava/lang/String;", null, null);
            fv.visitEnd();
        }
        {
            mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
            mv.visitCode();
            Label l0 = new Label();
            mv.visitLabel(l0);
            mv.visitLineNumber(6, l0);
            mv.visitVarInsn(ALOAD, 0);
            mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
            mv.visitInsn(RETURN);
            Label l1 = new Label();
            mv.visitLabel(l1);
            mv.visitLocalVariable("this", "Lcom/taobao/film/Demo;", null, l0, l1, 0);
            mv.visitMaxs(1, 1);
            mv.visitEnd();
        }
        {
            mv = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "main", "([Ljava/lang/String;)V", null, null);
            mv.visitCode();
            Label l0 = new Label();
            mv.visitLabel(l0);
            mv.visitLineNumber(15, l0);
            mv.visitVarInsn(ALOAD, 0);
            Label l1 = new Label();
            mv.visitJumpInsn(IFNULL, l1);
            mv.visitVarInsn(ALOAD, 0);
            mv.visitInsn(ARRAYLENGTH);
            mv.visitInsn(ICONST_1);
            mv.visitJumpInsn(IF_ICMPNE, l1);
            Label l2 = new Label();
            mv.visitLabel(l2);
            mv.visitLineNumber(16, l2);
            mv.visitFieldInsn(GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            mv.visitFieldInsn(GETSTATIC, "com/taobao/film/Demo", "CONST", "Ljava/lang/String;");
            mv.visitInsn(ICONST_1);
            mv.visitTypeInsn(ANEWARRAY, "java/lang/Object");
            mv.visitInsn(DUP);
            mv.visitInsn(ICONST_0);
            mv.visitVarInsn(ALOAD, 0);
            mv.visitInsn(ICONST_0);
            mv.visitInsn(AALOAD);
            mv.visitInsn(AASTORE);
            mv.visitMethodInsn(INVOKESTATIC, "java/lang/String", "format",
                "(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/String;", false);
            mv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            mv.visitLabel(l1);
            mv.visitLineNumber(18, l1);
            mv.visitInsn(RETURN);
            Label l3 = new Label();
            mv.visitLabel(l3);
            mv.visitLocalVariable("args", "[Ljava/lang/String;", null, l0, l3, 0);
            mv.visitMaxs(7, 1);
            mv.visitEnd();
        }
        {
            mv = cw.visitMethod(ACC_STATIC, "<clinit>", "()V", null, null);
            mv.visitCode();
            Label l0 = new Label();
            mv.visitLabel(l0);
            mv.visitLineNumber(8, l0);
            mv.visitInsn(ACONST_NULL);
            mv.visitFieldInsn(PUTSTATIC, "com/taobao/film/Demo", "CONST", "Ljava/lang/String;");
            Label l1 = new Label();
            mv.visitLabel(l1);
            mv.visitLineNumber(11, l1);
            mv.visitLdcInsn("Hello%s!");
            mv.visitFieldInsn(PUTSTATIC, "com/taobao/film/Demo", "CONST", "Ljava/lang/String;");
            Label l2 = new Label();
            mv.visitLabel(l2);
            mv.visitLineNumber(12, l2);
            mv.visitInsn(RETURN);
            mv.visitMaxs(1, 0);
            mv.visitEnd();
        }
        cw.visitEnd();

        return cw.toByteArray();
    }

    public static class MyClassLoader extends ClassLoader {
        public MyClassLoader() {
            super();
        }

        public MyClassLoader(ClassLoader cl) {
            super(cl);
        }

        public Class<?> defineClass(String name, byte[] b) {
            return defineClass(name, b, 0, b.length);
        }
    }

    public static void main(String[] args) throws Exception {
        byte[] bytes = dump();
        FileOutputStream fileOutputStream = new FileOutputStream("D:\\Demo.class");
        fileOutputStream.write(bytes);
        fileOutputStream.close();
        MyClassLoader classLoader = new MyClassLoader(Thread.currentThread().getContextClassLoader());
        Class<?> testClass = classLoader.defineClass("com.taobao.film.Demo",
            bytes);
        // invoke static
        Method staticMain = testClass.getMethod("main", String[].class);
        staticMain.invoke(null, new Object[] {new String[] {"zhupin"}});
    }
}
```

[XML快速序列化](https://github.com/pocrd/net.pocrd.core/blob/master/src/main/java/net/pocrd/util/POJOSerializerProvider.java)

[为无状态逻辑类的指定函数产生一个代理，代理接口接受字符串数组，转换后调用原函数](https://github.com/TimGuan/net.pocrd.core/blob/master/src/main/java/net/pocrd/util/HttpApiProvider.java)

### 知名框架中的使用
* fastjson hotcode asm
* dubbo hsf javaassist
* hibernate cglib javaassist
* spring aop cglib (通过设置-Dcglib.debugLocation=D://tmp 开启classdump)
* mockito cglib

##建议
最早接触字节码编程还是在使用IL(C# Emit)，也在大型项目中使用过ASM获得了不错的效果。不过个人觉得操作字节码编程还是要保持谨慎态度。下面是我的一点小建议：
* 操作字节码作用于运行期，对于开发人员是完全透明的。
* 字节码编程的可阅读可维护性比较差，不要滥用字节码编程。
    * 能通过设计模式实现的场景尽量通过设计模式实现；
    * 字节码编程中复杂的逻辑也尽量使用java实现，在字节码中调用；
    * 使用字节码解决一些框架性的问题，不要用于处理易变逻辑；
* 字节码编程从逻辑块着手，优先明确程序跳转Label，再补充逻辑执行
* 借助工具
推荐使用intellj idea插件ASM Bytecode Outline，目前生成的字节码对应到 ASM 5.x。
使用decompile工具校验生成代码是否正确。
