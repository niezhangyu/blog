---
title: Kotlin初体验
date: 2018-09-28 21:17:58
tags:
---

​	新公司使用kotlin开发安卓项目，在学习使用两周后，做一个总结。本篇文章直接从开发的角度来讲述Kotlin，引入Kotlin的知识点。

​	对于安卓而言，最多的交互场景为Activity，因此我们需要继承BaseActivity或者AppCompatActivity。对于继承，Kotlin有其自己的冒号来表达继承关系：

```kotlin
class MainActivity : AppCompatActivity()
```

​	对于Activity而言，我们习惯内部用静态方法来调用启动，于Java同样不同，静态方法这样表达：

```kotlin
companion object{
    fun 方法名（变量名：变量属性){
        //TODO 
    }
}
```

​	作为一个Activity或者一个类，我们肯定要定义很多成员变量，Koltin主要关键字就val和var两种。我们可以简单的认为val修饰的类似于Java里面的final声明常量，而var声明变量。Kotlin编码思想中，尽可能的使用val。对于类似Java定义变量属性的String等等，Kotlin表达如下

```kotlin
Kotlin  private var num : Int
Java    private String i
```

​	我们要特别注意，Kotlin中的数字类型不会自动转型，使用的时候必须做明确的类型转换。但还有个问题，变量声明的时候要求要被初始化，像上面这样声明是会报错的。所以我们必须如下声明：

```kotlin
private var name0: String //报错
private var name1: String = "johnnie" //不报错
private var name2: String? = null //不报错
```

​	name2的问号符号等会说到可空性再讲，这里我们暂且认为我们可以赋予变量null值。看到name0这样定义会报错，但是我们肯定有很多变量对象无法直接赋值,比如我们写RecyclerView的adapter。这个时候，Kotlin给我们提供了lateinit关键字延迟加载：

```kotlin
private lateinit var adapter : RecycleAdapter //不报错
private lateinit var num : Int //报错
private lateinit var string : String //不报错
```

​	laternit var只能用来修饰类属性,不能修饰基本数据类型

​	既然是Activity我们肯定要实现Activiy的生命周期，Java里是用注解的方式来表现该方法为重写方法，在Kotlin中，直接在方法前面加上override即可，例如：

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {}
```

​	我们从onCreate()方法也可以看出一般方法的写法，对于我们设定要有返回值的方法而言，参考方法如下：

```kotlin
fun doSomething(参数：参数类型) : 返回值类型{}
```

​	在安卓里我们要定义很多点击事件，Kotlin的Lambdas多样化我还在学习理解中。对于点击事件而言，如下

```kotlin
view.setOnClickListener(Object : OnClickListner{
    override fun onClick(v:View){
        //TODO someThing
    }
})
//lambda表达式通过参数的形式被定义在箭头的左边（即被圆括号包围的内容），然后在箭头的右边返回结果值，对于设置点击事件如下转换
fun setOnClickListener(listener:(View) -> Unit)

//即
view.setOnClickListener({
    view ->//TODO something
})

//如果左边的参数view没有用到。我们还可以省略左边的参数
view.setOnClickListener({
    //TODO something
})

//如果这个函数的最后一个参数还是一个函数，我们可以把这个函数移动到圆括号外面
view.setOnClickListener(){}

//如果这个函数只有一个参数，我们可以省略这个圆括号，所以最终
view.setOnClickListener{
    //TODO something
}
```

继续写，最近因为开始改bug所以遇到了很多回调函数函数的问题，作为kolin可以这么写。

~~~kotlin
fun init(id:Int,ready:(Bean) -> Unit){
    //获取Bean
	ready.invoke(xxxx.getBean(id))
}

//调用的时候
init(id){it->
    //此时it即为Bean
}

~~~

我对这个写法的更多的方便性还需要进一步理解，由于项目同样还是谷歌的MVVM框架，所以当我在ViewModle层开线程查询数据库等操作获得结果返回给View层的时候，这种写法对我开子线程同时需要回调十分有益。