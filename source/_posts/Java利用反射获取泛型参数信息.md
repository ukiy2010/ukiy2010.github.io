---
title: Java利用反射获取泛型参数信息
date: 2016-07-21 19:47:50
tags:
---
由于Java的类型擦除，对象（或者局部变量）本身不会存储任何泛型信息，但是泛型的信息会存储在**Class字节码中**，可以通过**反射**去获取到它们。比如下代码：

``` Java
class Cls<T> extends ArrayList<Integer> implements List<Integer>{
  List<String> field;
  <V> List<String> method(List<String> in){
    List<String> list = new ArrayList<>();
    return list;
  }
}
```

你不能直接从list中获取任何泛型有关的信息，也就是说运行时不可能获得list里面存储元素的类型。（除非取出list的的元素），因为类型擦除。

但是可以从**类（Class，Type）**，**方法（Method）**，**字段（Field）**中来获取泛型信息，还用上面的代码做例子：

``` Java
((ParameterizedType) Cls.class.getDeclaredField("field").getGenericType()).getActualTypeArguments()[0];
```

可以得到`java.lang.String`这个类

需要注意的是，获取到的这些泛型信息是在编译时就决定了的，不会在运行时再发生变化：

``` Java
Cls.class.getTypeParameters()[0];

new Cls<String>().getClass().getTypeParameters()[0];

new Cls<Integer>().getClass().getTypeParameters()[0];
```

都是得到`T`，仅仅是一个占位符而已。

但是可以用一个小技巧从对象中获得泛型的信息：

``` Java
(ParameterizedType) new Cls<String>() {}.getClass().getGenericSuperclass()).getActualTypeArguments()[0];
```

得到的就是`java.lang.String`了，这是因为这里声明了一个继承`Cls<String>`的匿名内部类，这样泛型信息就保存到这个对象的类中了，相当于：

``` Java
class Anonymous extends Cls<String>{}
```

这样`Anonymous`类就有`Cls<String>`的泛型信息了。大名鼎鼎的gson就利用了这个办法保存泛型信息，具体代码在`TypeToken`中。

---

持有参数化类型的类叫做`ParameterizedType`，比如`List<String>`，可以从中取出参数化的类型`String`

> `ParameterizedType`和`Class`都继承自Type，`class.getSuperclass()`和`class.getGenericSuperclass()`的区别是，前者返回Class，后者返回Type，举例来说`Cls.class.getSuperclass()`得到的是ArrayList，`Cls.class.getGenericSuperclass()`得到的是`ArrayList<Integer>`，强制转换成`ParameterizedType`后可以得到泛型参数的信息。

获取本类的类型参数，返回`T`：

``` Java
Cls.class.getTypeParameters()[0]
```

获取父类的类型参数，返回`Integer`：

``` Java
((ParameterizedType)Cls.class.getGenericSuperclass()).getActualTypeArguments()[0]
```

获取所继承接口（第i个）的类型参数，返回`Integer`：

``` Java
((ParameterizedType) Cls.class.getGenericInterfaces()[i]).getActualTypeArguments()[0]
```

获取方法参数（第i个）的类型参数，返回`String`：

``` Java
((ParameterizedType) getClass().getDeclaredMethod("method", List.class).getGenericParameterTypes()[i]).getActualTypeArguments()[0]
```

获取方法返回值的类型参数，返回`String`：

``` Java
((ParameterizedType) getClass().getDeclaredMethod("method", List.class).getGenericReturnType()).getActualTypeArguments()[0]
```

获取方法的类型参数，返回`V`：

``` Java
getClass().getDeclaredMethod("method", List.class).getTypeParameters()[0]
```

获取字段的类型参数，返回`String`：

``` Java
((ParameterizedType) Cls.class.getDeclaredField("field").getGenericType()).getActualTypeArguments()[0];
```

---

在获取方法的时候`method.getDeclaredMethod("method", List.class)`第二个参数，表示方法的参数类型列表，如果是基本类型则用`XX.TYPE`或者`xx.class`，例如：`int`对应`Integer.TYPE`或者`int.class`，`long`对应`Long.TYPE`或者`long.class`


