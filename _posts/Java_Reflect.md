---
title: 'Java反射与优化'
date: 2017-08-14 21:35:40
tags: [插件化]
categories: [Android,插件化]
---

还是插件化相关的内容，不过这次说的是反射相关的。插件化的两个基础，动态代理与反射，上次说了动态代理，这次就说反射了。
先说一下Java的内存模型，也就是java虚拟机在运行时的内存。运行时的内存分为线程私有和线程共享两块。
线程私有的有程序计数器，虚拟机栈，本地方法栈，线程共享的有方法区(包含运行时常量池)，java堆。
![内存模型](/images/jvm_neicun.png)

我们平时说的java内存分为堆和栈，分别对应的是上面的堆和虚拟机栈。
**程序计数器:** java允许多个线程同时执行指令，如果是有多个线程同时执行指令，那么每个线程都有一个程序计数器，在任意时刻，一个线程只允许执行一个方法的代码，每当执行到一条java方法的代码时，程序计数器保存当前执行字节码的地址，若执行的为native方法，则PC的值为undefined。
**虚拟机栈:** 描述了java方法执行的内存模型，每个方法在执行的时候都会创建出一个帧栈，用于存储局部变量表，操作数栈，动态链接，方法出口等信息，每个方法的从调用到完成，都对应着一个帧栈从入栈到出栈的过程。
**本地方法栈:** 为虚拟机使用到的Native方法提供内存空间，本地方法栈使用传统的C Stack来支持native方法。

**java堆:** 提供线程共享时的内存区域，是java虚拟机管理的最大的一块内存区域，也是gc的主要区域，几乎所有的对象实例和数组实例都要在java堆上分配。java堆的大小可以是固定的，也可以随着需要来扩展，并且在用不到的时候自动收缩。
**方法区:** 存放已被虚拟机加载的类信息，常量，静态变量，编译器编译后的代码等数据。
**运行时常量池:** 存放编译器生成的字面量和符号引用。

### 1.反射是什么
反射是java语言的特性之一，它允许运行中的程序获取自身的信息，并且可以操作类和对象的内部属性。java反射框架主要提供以下功能:
- 1.在运行时判断任意对象所属的类；
- 2.在运行时构造任意一个类的对象；
- 3.在运行时判断任意一个类所具有的成员变量和方法(通过反射甚至可以调用private方法)；
- 4.在运行时调用任意一个对象的方法;

### 2.反射的用途
- 1.我们在使用ide，输入一个对象，并想调用它的属性和方法的时候，一按点号，编译器就会自动列出它的属性和方法，这里就会用到反射。
- 2.通用框架，很多框架都是配置化的(比如Spring通过xml配置Bean或者Action)， 为了保证框架的通用性，可能需要根据不同的配置文件加载不同的对象或者类，调用不同的方法，这个时候就需要反射，运行时动态加载需要加载的对象。

### 3.反射的基本运用
上面提到了提供的一些功能，获取类，调用类的属性或者方法。
#### 3.1.获取类(Class)对象
方法有三种:
- 使用Class的静态方法
```java
try {
    Class.forName("");
} catch (ClassNotFoundException e) {
    e.printStackTrace();
}
```

- 直接获取一个对象的Class
```java
public class Reflection {

    private void getClazz() {
        Class<Reflection> c = Reflection.class;
        Class<String> s = String.class;
        Class<Integer> i = int.class;
    }

}
```

- 调用某个对象的getClass方法
```java
ArrayList list = new ArrayList();
Class<?> l = list.getClass();
```

#### 3.2.判断是否为某个类的实例
一般我们使用`instanceof`，也可以使用`Class.isInstance(obj)`。
```java
StringBuilder sb = new StringBuilder();
Class<?> c = sb.getClass();
System.out.println(c.isInstance(sb));
```

#### 3.3.创建实例
用反射来生成对象的方式主要有两种。
- 使用`Class.newInstance`方法
这个方法最终调用的是无参数的构造函数，所以如果对象没有无参数的构造函数就会报错了。使用`newInstance`必须要保证：1、这个 类已经加载；2、这个类已经连接了。newInstance()实际上是把new这个方式分解为两步，即首先调用Class加载方法加载某个类，然后实例化。当然构造方法不能是私有的。
```java
Class<Reflection> c = Reflection.class;
try {
    Reflection r = c.newInstance();
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
}
```

- 先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例。这种方法可以用指定的构造器构造类的实例。当然构造方法不能是私有的。
```java
Class<String> s = String.class;
try {
    Constructor constructor = s.getConstructor(String.class);
    try {
        Object o = constructor.newInstance("378");
        System.out.println(o);
    } catch (InstantiationException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    }
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}
```

#### 3.4.获取方法（Method）
获取某个Class对象的方法集合，主要有以下几种方法。
```java
public Method[] getDeclaredMethods() throws SecurityException
```
可以获取当前类的公有，保护，默认，私有的方法，即可以拿到当前类的所有的方法，但是必须是当前类里面存在的。
```java
public Method[] getMethods() throws SecurityException
```
只能拿public的，但是这个情况复杂一些。
1.当前类里面定义的public方法
2.实现的接口的public方法（妈蛋，实现的接口的方法肯定是public的）
3.父类里面的public方法
4.父类里面的protected的方法，但是当前类public方式覆写
总结就是获取当前类里面存在的或者父类里面存在的public类型的方法。
```java
public Method getDeclaredMethod(String name, Class<?>... parameterTypes) throws NoSuchMethodException, SecurityException
```
可以获取特定的自身的公有，保护，默认，私有的方法，但是不包括继承实现的方法。
```java
public Method getMethod(String name, Class<?>... parameterTypes) throws NoSuchMethodException, SecurityException
```
可以获取特定的公有和继承实现的方法。

#### 3.5.获取构造器信息（Constructor）
通过Class对象的`getConstructor`方法。
```java
Class<String> s = String.class;
try {
    Constructor constructor = s.getConstructor(String.class);
} catch (NoSuchMethodException e) {
    e.printStackTrace();
}
```

#### 3.6.获取成员变量信息（Field）
主要是这几个方法，在此不再赘述：
getFiled: 访问公有的成员变量
getDeclaredField：所有已声明的成员变量。但不能得到其父类的成员变量
getFileds和getDeclaredFields用法同上（参照Method）

#### 3.7.调用方法（invoke）
当我们从类中获取了一个方法后，我们就可以用invoke()方法来调用这个方法。
```java
public class Reflection {

    private String mm;

    public Reflection(String v) {
        mm = v;
    }

    public static void main(String[] ps) {
        runMethod();
    }

    private void in() {
        System.out.println(mm);
    }

    private static void runMethod() {
        Class<Reflection> c = Reflection.class;
        try {
            Constructor constructor = c.getConstructor(String.class);
            try {
                Object o = constructor.newInstance("378");
                Method method = c.getDeclaredMethod("in", (Class<?>[]) null);
                method.setAccessible(true);
                method.invoke(o, (Object[]) null);
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }

}
```
invoke方法用来在运行时动态地调用某个实例的方法。invoke方法会首先检查AccessibleObject的override属性的值。AccessibleObject 类是 Field、Method 和 Constructor 对象的基类。它提供了将反射的对象标记为在使用时取消默认 Java 语言访问控制检查的能力。override的值默认是false,表示需要权限调用规则，调用方法时需要检查权限;我们也可以用setAccessible方法设置为true,若override的值为true，表示忽略权限规则，调用方法时无需检查权限（也就是说可以调用任意的private方法，违反了封装）。

#### 3.8.利用反射创建数组
数组在Java里是比较特殊的一种类型，它可以赋值给一个Object Reference。
```java
public static void createArray() throws ClassNotFoundException {
    Class<?> cls = Class.forName("java.lang.String");
    Object array = Array.newInstance(cls, 3); // 等价于 new String[3];
    //往数组里添加内容
    Array.set(array, 0, "OK");
    Array.set(array, 1, "HOW ARE YOU");
    Array.set(array, 2, "Fine");
    //获取某一项的内容
    System.out.println(Array.get(array, 2)); // 等价于array[2]
}
```

#### 3.9.泛型的处理
Java 5中引入了泛型的概念之后，Java反射API也做了相应的修改，以提供对泛型的支持。由于类型擦除机制的存在，泛型类中的类型参数等信息，在运行时刻是不存在的。JVM看到的都是原始类型。
```java
private List<String> genericTypeValue = new ArrayList<String>();
private List nullGenericType;

public void testGenericType() throws SecurityException, NoSuchFieldException, InstantiationException, IllegalAccessException{
        //如果类属性的类型带有类型参数，如List<T>
        //那么想获取类型T时用field.getGenericType();方法,然后转型为参数化类型[ParameterizedType]
        Field genericTypeField1 = clazz.getDeclaredField("genericTypeValue");
        Field genericTypeField2 = clazz.getDeclaredField("nullGenericType");

        ParameterizedType genericType1 = (ParameterizedType)genericTypeField1.getGenericType();
//        nullGenericType并没有参数类型，强制转换为(ParameterizedType)会抛异常！
//        只能转换为(Class<?>)或通过getType()获得类型
//        ParameterizedType genericType2 = (ParameterizedType)genericTypeField2.getGenericType();
        Class<?> type1 = genericTypeField1.getType();//type1为List<String>的类型！
        Class<?> Type2 = (Class<?>)genericTypeField2.getGenericType();
        Class<?> Type2_1 = genericTypeField2.getType();

        //通过参数化类型[ParameterizedType]获得声明的参数类型的数组
        Type[] types1 = genericType1.getActualTypeArguments();
        Class<?> typeValue1 = (Class<?>) types1[0];
        System.out.println("typeValue1:"+typeValue1);//class test.String
        System.out.println("typeValue2:"+Type2);//interface java.util.List
        System.out.println("typeValue2_1:"+Type2_1); //interface java.util.List

        if(typeValue1.equals(String.class))    //true
            System.out.println("typeValue1.equals(String.class)?"+typeValue1.equals(String.class));
        if(Type2.equals(List.class))    //true
            System.out.println("Type2.equals(List.class)?"+Type2.equals(List.class));

        //创建包含参数类型的类型的对象[异常！类型声明为接口List，而却要创建ArrayList]
//        ArrayList<String> newInstance = (ArrayList<String>) type1.newInstance();
//        newInstance.add("123");

}
```

### 4.反射的优化
#### 4.1.善用API
比如，尽量不要getMethods()后再遍历筛选，而直接用getMethod(methodName)来根据方法名获取方法。

#### 4.2.缓存大法好
比如，需要多次动态创建一个类的实例的时候，有缓存的写法会比没有缓存要快很多。还有将反射得到的method/field/constructor对象做缓存。
```java
// 1. 没有缓存
void createInstance(String className){
    return Class.forName(className).newInstance();
}

// 2. 缓存forName的结果
void createInstance(String className){
    cachedClass = cache.get(className);
    if (cachedClass == null){
        cachedClass = Class.forName(className);
        cache.set(className, cachedClass);
    }
    return cachedClass.newInstance();
}
```
为什么？当然是因为forName太耗时了。Cache请自行实现。

#### 4.3.尽量使用高版本JDK

#### 4.4.使用反射框架
例如[joor](https://github.com/jOOQ/jOOR)，或者Apach Commons BeanUtils，JAVAASSIST。

#### 4.5.[ReflectASM](https://github.com/EsotericSoftware/reflectasm)通过字节码生成的方式加快反射速度
ASM 是一个 Java 字节码操控框架。它能被用来动态生成类或者增强既有类的功能。ASM 可以直接产生二进制 class 文件，也可以在类被加载入 Java 虚拟机之前动态改变类行为。Java class 被存储在严格格式定义的 .class 文件里，这些类文件拥有足够的元数据来解析类中的所有元素：类名称、方法、属性以及 Java 字节码（指令）。ASM 从类文件中读入信息后，能够改变类行为，分析类信息，甚至能够根据用户要求生成新类。
java的源码在源代码和编译后的类中表现是不一样的。
下面列出java类型对应的类型描述符:
|    Java类型    |    Type Descriptor    |    说明    |
|     ----------    |    ------------------    |    -----    |
|    boolean    |    Z    |   B被byte占用了    |
|   char    |   C    |    说明    |
|   byte    |   B    |    说明    |
|   short    |    S    |    说明    |
|    int    |    I    |    说明    |
|   long    |    J    |    不用L是L被对象的类型描述符占用了    |
|   float    |    F    |    说明    |
|    double    |    D    |    说明    |
|   void    |    V    |    说明    |
|  数组    |    [    |    以[开头，配合其他的特殊字符，表示对应数据类型的数组，几个[表示几维数组    |
|   引用类型    |    L全类名;    |   以L开头、;结尾，中间是引用类型的全类名    |
|   方法    |    (参数类型参数类型)返回类型    |   方法的描述是括号，括号里面是参数，然后括号右边是返回类型    |

字段描述符示例
|    描述符   |    字段声明    |
|     ----------    |    ------------------    |
|     I    |    int i    |
|     [[J    |    long[][] xi    |
|     [Ljava/lang/Object;    |    Object[] obj    |
|     Ljava/util/Hashtable;    |    Hashtable tab    |
|     [[[Z    |    boolean[][][] re    |

方法描述符示例
|    描述符   |    方法声明    |
|     ----------    |    ------------------    |
|     ()I    |    int getCount()    |
|     ()Ljava/lang/String;    |    String getDesc()    |
|     ([Ljava/lang/String;)V    |    void sp(String[] s)    |
|     (J)Ljava/lang/String;    |    String ltostr(long t)    |
|     (JI)V    |    void wait(long t,int count)    |
|     ([BJI)I    |    int wit(byte[] t,long l,int i)    |
|     (Z[Ljava/lang/String;II)Z    |    boolean should(boolean ig,String s,int i,int j)    |

执行一下 javap -s java.lang.String 来看看 java.lang.String 的所有方法签名
![方法签名](/images/java_method_sign.png)

实例:
```java
public class Asmain {

    public static void main(String[] args) {

        ClassVisitor visitor = new ClassVisitor(Opcodes.ASM5) {
            @Override
            public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
                super.visit(version, access, name, signature, superName, interfaces);
                //打印出父类name和本类name
                System.out.println(superName + " " + name);
            }

            @Override
            public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
                //打印出方法名和类型签名
                System.out.println(name + " " + desc);
                return super.visitMethod(access, name, desc, signature, exceptions);
            }
        };
        //读取静态内部类
        ClassReader cr = null;
        try {
            cr = new ClassReader("com.yong.reflection.asm.Asmain$Sam");
            cr.accept(visitor, 0);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    static class Sam {

        private String name;

        public Sam(String name) {
            this.name = name;
        }

        private long getAge() {
            return 25;
        }

        private void Say() {
            System.out.println("你是不是傻...");
        }

    }

}
```
输出:
![方法签名](/images/java_code_method_sign.png)
然后我们可以往Sam类里面新增方法:
```java
public void addedMethod(String str) {
}
```
使用ClassWriter
```java
        try {
            ClassReader classReader = new ClassReader(Sam.class.getName());
            ClassWriter classWriter = new ClassWriter(classReader, ClassWriter.COMPUTE_MAXS);
            classReader.accept(classWriter, Opcodes.ASM5);
            MethodVisitor mv = classWriter.visitMethod(ACC_PUBLIC, "addedMethod", "(Ljava/lang/String;)V", null, null);
            mv.visitInsn(Opcodes.RETURN);
            mv.visitEnd();
            // 获取生成的class文件对应的二进制流
            byte[] code = classWriter.toByteArray();
            //将二进制流写到目录下
            FileOutputStream fos = new FileOutputStream("./javareflection/Sm.class");
            fos.write(code);
            fos.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
```
看生成的代码:
![生成代码](/images/java_asm_gen_code.png)
