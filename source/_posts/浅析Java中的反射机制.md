---
title: 浅析Java中的反射机制
date: 2019-08-27 20:27:05
tags: [Java,反射]
---
## 作用
反射机制属于Java中的常用的高级机制之一，他的功能非常强大，反射机制可以用来
- 在运行时分析类的能力
- 在运行时查看对象，如编写一个toString方法供所有类使用
- 实现通用的数组操作代码
- 利用Method对象，这个对象很像C++中的函数指针

<!-- more -->
## Class类
在运行期间，Java始终为所有对象维护一个被称为**运行时**的类型标识，这个信息跟踪着每个对象所属的类。虚拟机利用运行时类型信息选择相应的方法执行。然而我们可以使用专门的Java类访问这些信息。保存这些信息的类就被称为Class。关于如何获得这个Class对象有如下方法

### getClass方法和它的getName方法
Object类中的getClass方法就会返回一个Class类型的实例
```java
import java.util.ArrayList;

public class Main {
    public static void main(String[] args) {
        ArrayList<String> strings = new ArrayList<>();
        System.out.println(strings.getClass());
        System.out.println(strings.getClass().getName());
    }
}

>>> class java.util.ArrayList
>>> java.util.ArrayList
```
### 静态方法forName
Class类的静态方法forName可以获得对应的Class对象
```java
public class Main {
    public static void main(String[] args) {
        String string = "java.util.Random";
        try {
            Class<?> aClass = Class.forName(string);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

### 直接使用.class
```java
public class Main {
    public static void main(String[] args) {
        System.out.println(String.class.getName());
    }
}

>>> java.lang.String
```

### 注意
其实一个Class对象实际上表示的是一个类型，而这个类型未必是一个类，例如int不是类，但是int.class是一个Class类型的对象

### ==实现类对象的比较
```java
public class Main {
    public static void main(String[] args) {
        String string = "hello";
        System.out.println(string.getClass() == String.class);
    }
}
```

### 使用newInstance()动态创建一个类的实例
- 使用Class中的newInstance创建一个无参实例
- 使用java.lang.reflect.Constructor中的newInstance创建一个有参实例
```java
public class Main {
    public static void main(String[] args) {
        String string = "java.lang.String";
        try {
            Object object = Class.forName(string).newInstance();
            if (object.getClass() == String.class) {
                String string2 = (String) object;
                System.out.println(string2.isEmpty());
            }
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

>>> true
```

## 使用反射分析类的能力
反射机制最重要的内容就是，检查类的结构，在java.lang.reflect包下，有三个类，Field,Method,Constructor，可以用来查看类的结构，如下一段程序使用这个特性完成了打印一个类的Field，Method,Constructor的功能
```java
import java.lang.reflect.*;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        String name;
        if (args.length != 0) {
            name = args[0];
        } else {
            Scanner scanner = new Scanner(System.in);
            name = scanner.nextLine();
        }
        try {
            Class<?> aClass = Class.forName(name);
            Class<?> superclass = aClass.getSuperclass();
            String string = Modifier.toString(aClass.getModifiers());
            if(string.length() > 0) System.out.print(string+" ");
            System.out.print("class "+ name);
            if (superclass != null && superclass != Object.class) System.out.println(" extends "+superclass.getName());
            System.out.print("\n{\n");
            printConstructors(aClass);
            System.out.println();
            printMethod(aClass);
            System.out.println();
            printField(aClass);
            System.out.println();
            System.out.println("}");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void printField(Class<?> aClass) {
        // getDeclareField获得这个类的所有域，getField获得这个类和他的超类的所有域
        Field[] fields = aClass.getDeclaredFields();
        for (Field field : fields) {
            String string = Modifier.toString(field.getModifiers());
            System.out.print(string + " ");
            System.out.print(field.getType());
            System.out.print(" "+field.getName());
            System.out.println();
        }
    }

    private static void printMethod(Class<?> aClass) {
        Method[] methods = aClass.getDeclaredMethods();
        for (Method method : methods) {
            String string = Modifier.toString(method.getModifiers());
            System.out.print(string + " ");
            System.out.print(method.getReturnType()+ " ");
            System.out.print(method.getName());
            System.out.print("(");
            Class<?>[] parameterTypes = method.getParameterTypes();
            for (int i = 0; i < parameterTypes.length; i++) {
                if (i > 0) {
                    System.out.print(",");
                }
                System.out.print(parameterTypes[i].getName());
            }
            System.out.println(");");
        }
    }

    public static void printConstructors(Class c) {
        Constructor[] declaredConstructors = c.getDeclaredConstructors();
        for (Constructor declaredConstructor : declaredConstructors) {
            String name = c.getName();
            System.out.println(" ");
            String string = Modifier.toString(c.getModifiers());
            if(string.length() > 0) System.out.print(string+" ");
            System.out.print(name + "(");

            // 打印参数类型
            Class[] parameterTypes = declaredConstructor.getParameterTypes();
            for (int i = 0; i < parameterTypes.length; i++) {
                if (i>0) {
                    System.out.print(" ,");
                }
                System.out.print(parameterTypes[i].getName());
            }
            System.out.print(");");
        }
    }
}
```

## 在运行时使用反射分析对象
上面的代码主要用于这个类还没有运行的时候对于类的分析，下面就是使用反射对一个已经运行的实例的分析与修改。下面的代码对一个Employee实例，进行了修改
- getDeclaredFields获取域
- 使用setAccessible将其设置为true,否则当遇到Modifier为private的属性，将抛出IllegalAccessException
- 使用Field.get查看某个实例对应域的值
- 使用field.set(object, value)
- 关于getComponentType,如果是Array类型,他会返回数组中的内容类型，否则会返回null
- 你可以通过这种形式加上递归调用实现一个通用的toString方法
```java
import java.lang.reflect.Field;

public class Main {
    public static void main(String[] args) {
        Employee ayang919 = new Employee("ayang919", "homework");
        Class<? extends Employee> aClass = ayang919.getClass();
        try {
            Field[] fields = aClass.getDeclaredFields();
            for (Field field : fields) {
                field.setAccessible(true);
                System.out.println(field.get(ayang919));
                field.set(ayang919, "new work");
            }
            System.out.println();
            System.out.println(ayang919.task);
            System.out.println(ayang919.getName());
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }

    }
}

class Employee {
    private String name;
    public String task;

    public Employee(String name, String task) {
        this.name = name;
        this.task = task;
    }

    public String getName() {
        return name;
    }
}
```
```java
import java.lang.reflect.AccessibleObject;
import java.lang.reflect.Array;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.util.ArrayList;
import java.util.List;

/**
 * @author rambo.pan
 * 2016年10月10日
 */
public class ObjectAnalyzer {
    private static final String NULL_STRING = "null";
    private static final String VISITED_STRING = "...";
    //该数组用来记录已解析过的对象类型
    private List<Object> visited = new ArrayList<Object>();

    /**
     * 递归方法，object是个数组，则遍历其元素，递归调用该方法；否则是个类实例，则遍历其所有字段
     * @param obj
     * @return
     */
    public String toString(Object obj){
        //如果对象为空，则返回null
        if (obj == null) {
            return NULL_STRING;
        }
      //如果对象已被处理过，则返回，针对于 A、B对象相互组合的情况，否则会循环递归，导致StackOverFlow
        if (visited.contains(obj)) {
            return VISITED_STRING;
        }
        visited.add(obj);

        //获取对象的Class实例，同一个类的对象，共有一个相应的Class实例，在类被加载到虚拟机时就存储在方法区中了
        Class<?> cl = obj.getClass();
        //如果对象正好是String对象，则强转后直接返回
        if (cl == String.class) {
            return (String)obj;
        }
        //如果object是个数组，则遍历其元素，对元素进行toString(Object)递归调用
        if (cl.isArray()) {
            //获取数组的元素类型
            String r = cl.getComponentType() + "[]{";
            for (int i = 0; i < Array.getLength(obj); i++) {
                if (i > 0) {
                    r += ",";
                }
                //获取指定下标的数组元素
                Object val = Array.get(obj, i);
                //如果元素是原始/基本类型(Java共8种基本类型),则直接输出
                if (cl.getComponentType().isPrimitive()) {
                    r += val;
                }else {//否则递归调用，直到其为原始类型
                    r += toString(val);
                }
            }
            //对数组元素遍历完成，返回。
            return r + "}";
        }
        //object不是数组类型，首先获取其类型Name
        String r = cl.getName();
        //遍历object的所有字段，然后获取字段的Name和Value，之后再遍历其父类，直到遍历到父类为Object
        do{
            r += "[";
            //获取object的所有字段，无论访问限制符是public还是private，但不包括其从父类继承来的字段
            Field[] fields = cl.getDeclaredFields();
            //设置字段的访问权限为可访问
            AccessibleObject.setAccessible(fields, true);
            for (Field field : fields) {
                //忽略static属性
                if (Modifier.isStatic(field.getModifiers())) {
                    continue;
                }

                if (!r.endsWith("[")) {
                    r += ",";
                }
                r += field.getName() + "=";
                try{
                    //获取字段类型和字段的值
                    Class<?> t = field.getType();
                    Object val = field.get(obj);
                    //如果字段类型是原始类型，则直接输出；否则递归调用toString(Object)
                    if (t.isPrimitive()) {
                        r += val;
                    }else {
                        r += toString(val);
                    }
                }catch (Exception e) {
                    e.printStackTrace();
                }

            }
            //一个类型的字段遍历结束
            r += "]";
            //获取该类型的父类Class实例，然后继续该循环
            cl = cl.getSuperclass();
        }while(cl != null);

        //类型遍历完成，返回object的字符串结果
        return r;
    }
}
```
