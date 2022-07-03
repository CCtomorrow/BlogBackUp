---
title: '使用Compose编写ui界面'
date: 2022-02-20 17:00:00
tags: [Compose]
categories: [Android,Compose]
---

Jetpack Compose 1.0.0正式版于[2021年7.28号](https://mvnrepository.com/artifact/androidx.compose.ui/ui/1.0.0)发布，现在最新版本已经是[1.2.0-alpha03](https://mvnrepository.com/artifact/androidx.compose.ui/ui)。

文档:https://developer.android.com/jetpack/compose/why-adopt?hl=zh-cn
官方课程:https://developer.android.com/courses/pathways/compose?hl=zh-cn

### [Compose 编程思想](https://developer.android.com/jetpack/compose/mental-model?hl=zh-cn)
Jetpack Compose 是一个适用于 Android 的新式声明性界面工具包。首先说一下什么是声明式，一般来说，声明式是相对于命令式来说的。命令式就是根据指令来改变状态，拿安卓来说从`findViewById()`找到组件，然后通过`textView.setText(String)`设置数据手动改变ui的状态和显示。
声明式就是组件不以对象的方式提供，主要通过描述我们需要的需要展示的状态，状态改变的时候使用之前的描述重新更新ui，自动订阅数据的改变，不需要手动更新，当然现在的声明式的编程比如`SwiftUI`，`React`，`Vue`都是他们自身或者编译器提供了一些方式更新ui的时候只更新需要更新的部分，不会整个重新计算渲染。

关于改变界面的状态界面重新构建的更新，在`React`采用了VirtualDom的思想，自己实现了[Diffing](https://zh-hans.reactjs.org/docs/reconciliation.html)算法去实现局部刷新，以提高性能。

Compose 没有使用树形结构，而是在 Gap Buffer 这样线性结构上进行 diff。但本质上是相同的，可以将 Gap Buffer 理解为一个树形结构经 DFS 处理后的数组，数组单元通过 key 标记其在树上的位置信息。
Compose 在编译期为 Composable 生成带有位置信息的 key，存入到 Gap Buffer 数组的对应位置。运行时可以根据 key 来识别 Composable 节点是否发生了位置变化，以决定是否参与[重组](https://juejin.cn/post/6965671818598285325)。

databing和compose的区别，databing只能更新界面的值，compose可以更新界面的任意内容，包括界面的结构。

<!-- more -->

### [Compose 性能](https://juejin.cn/post/7008522702835154980)
![性能](/images/compose_xn.png)
目前从测试的结果来看，compose的性能并不能比上xml布局的，原因也是各有各的说法，有的是说compose没有AOT，目前跑的JIT形式，还有的是说compose这类高度依赖计算的方式，加之kotlin中函数还不是一等公民，初始化布局时，大量的对象创建和堆栈调用，慢也不奇怪。 

不过随着后面的发展，Google对compose的优化肯定是越来约好的，再一个，我们写普通的界面来说影响并不大，多列表的我们还是可以使用RecyclerView，因为compose可以方便在旧项目里面使用起来。


### [Compose 常用布局和组件](https://developer.android.com/codelabs/jetpack-compose-layouts)
compose里面有三大基础容器布局组件
![基础布局](/images/compose_base_lay.png)
Column，Row可以类比LinearLayout，Box类比FrameLayout，LazyColumn和LazyRow类比RecyclerView，不过目前来说性能没有RecyclerView好。

#### Modifier修饰符
借助修饰符，以修饰可组合项，更改其行为、外观，添加无障碍功能标签等信息，处理用户输入，甚至添加高级交互（例如使某些元素可点击、可滚动、可拖动或可缩放）。
可以将它们分配给变量并重复使用，也可以将多个修饰符逐一串联起来，以组合这些修饰符。
串联修饰符时请务必小心，因为顺序很重要。由于修饰符会串联成一个参数，所以顺序将影响最终结果。

如下面一个简单的示例:
![基础布局示例](/images/compose_base_lay_sample.png)


### [Compose 中的 State](https://developer.android.com/codelabs/jetpack-compose-state)
![](/images/compose_state_flow.png)
先来看看在之前的view体系里面，我们一般更新一个ui是怎么操作的。
首先来看直接使用api的方式的更新
```kotlin
class HelloCodelabActivity : AppCompatActivity() {

   private lateinit var binding: ActivityHelloCodelabBinding
   private var name = ""

   override fun onCreate(savedInstanceState: Bundle?) {
       binding.textInput.doAfterTextChanged {text ->
           name = text.toString()
           updateHello()
       }
   }

   private fun updateHello() {
       binding.helloText.text = "Hello, $name"
   }
}
```

后面推出了ViewModel，再来看使用ViewModel的方式的更新
```kotlin
class HelloCodelabViewModel: ViewModel() {

   // LiveData holds state which is observed by the UI
   // (state flows down from ViewModel)
   private val _name = MutableLiveData("")
   val name: LiveData<String> = _name

   // onNameChanged is an event we're defining that the UI can invoke
   // (events flow up from UI)
   fun onNameChanged(newName: String) {
       _name.value = newName
   }
}

class HelloCodeLabActivityWithViewModel : AppCompatActivity() {
   private val helloViewModel by viewModels<HelloCodelabViewModel>()

   override fun onCreate(savedInstanceState: Bundle?) {
       /* ... */

       binding.textInput.doAfterTextChanged {
           helloViewModel.onNameChanged(it.toString())
       }

       helloViewModel.name.observe(this) { name ->
           binding.helloText.text = "Hello, $name"
       }
   }
}
```

Compose版本
```kotlin
@Composable
private fun HelloScreen(helloViewModel: HelloViewModel = viewModel()) {
    val name: String by helloViewModel.name.observeAsState("")
    HelloInput(name = name, onNameChange = { 
        helloViewModel.onNameChanged(it) 
        }
    )
}

@Composable
private fun HelloInput(name: String, onNameChange: (String) -> Unit) {
    Column {
        Text(name)
        TextField(
            value = name,
            onValueChange = onNameChange,
            label = { Text("Name") }
        )
    }
}
```

Compose 应用程序通过调用可组合函数将数据转换为 UI。如果您的数据发生更改，Compose 会使用新数据重新执行这些函数，创建更新的 UI — 这称为重新组合。Compose 还会查看单个可组合项需要哪些数据，以便它只需要重组数据已更改的组件，并跳过重组未受影响的组件。

对于上面的代码来说，意思就是如果用户输入的内容变了，会再次的调用`HelloInput`方法。这样的话，例如如下的代码，就算我们在Text里面输入了文本，界面也不会有任何的变化。
```kotlin
@Composable
fun HelloContent() {
   Column(modifier = Modifier.padding(16.dp)) {
       Text(
           text = "Hello!",
           modifier = Modifier.padding(bottom = 8.dp),
           style = MaterialTheme.typography.h5
       )
       OutlinedTextField(
           value = "",
           onValueChange = { },
           label = { Text("Name") }
       )
   }
}
```
如上面的代码，OutlinedTextField是一个文本输入框，上面的代码，我们在输入框里面输入任何的内容，界面也不会有变化，这是因为，TextField 不会自行更新，但会在其 value 参数更改时更新。这是因 Compose 中组合和重组的工作原理造成的。
想要有变化，我们需要在onValueChange回调里面把我们输入的值赋值给`OutlinedTextField`的`value`，跟上面Compose版本的示例那样。

如上面所说`如果您的数据发生更改，Compose 会使用新数据重新执行这些函数，创建更新的 UI`，这样的话，我们假如有变量想在`Composable`修饰的函数里面使用，那么我们应该怎么做呢？
举一个简单的例子，假如我们有一个TextView，一个Input输入框，我们想要在输入框没有文字的时候隐藏TextView，有内容的时候TextView显示输入框的内容，我们会想到写出如下的代码:
```kotlin
@Composable
fun HelloContent() {
   Column(modifier = Modifier.padding(16.dp)) {
       var name = ""
       if (name.isNotEmpty()) {
           Text(
               text = "Hello, $name!",
               modifier = Modifier.padding(bottom = 8.dp),
               style = MaterialTheme.typography.h5
           )
       }
       OutlinedTextField(
           value = name,
           onValueChange = { name = it },
           label = { Text("Name") }
       )
   }
}
```
按照compose的原理，每次界面改变，都会重新执行HelloContent方法，按照上面的代码，就算我们输入了内容，给name赋值了，OutlinedTextField里面的value变了，要展示界面的话又会重新执行HelloContent这样，name又变成了空字符串。

所以想要在Composable函数里面记住变量需要使用特别的方式。不过compose已经为我们提供好了对应的方式。

可组合函数(Composable函数)可以使用 remember 可组合项记住单个对象。系统会在初始组合期间将由 remember 计算的值存储在组合中，并在重组期间返回存储的值。remember 既可用于存储可变对象，又可用于存储不可变对象。

mutableStateOf 会创建可观察的 MutableState<T>，后者是与 Compose 运行时集成的可观察类型。value 如有任何更改，系统会安排重组读取 value 的所有可组合函数。

在可组合项中声明 MutableState 对象的方法有三种：
```kotlin
val mutableState = remember { mutableStateOf(default) }
var value by remember { mutableStateOf(default) }
val (value, setValue) = remember { mutableStateOf(default) }
```
这些声明是等效的，以语法糖的形式针对状态的不同用法提供。

上面示例里面的name换如下形式即可。
```kotlin
var name by remember { mutableStateOf("") }
```

虽然 remember 可帮助您在重组后保持状态，但不会帮助您在配置更改（页面旋转方向之类的）后保持状态。为此，您必须使用 rememberSaveable。rememberSaveable 会自动保存可保存在 Bundle 中的任何值。对于其他值，您可以将其传入自定义 Saver 对象。

Jetpack Compose 并不要求您使用 MutableState<T> 存储状态。Jetpack Compose 支持其他可观察类型。在 Jetpack Compose 中读取其他可观察类型之前，您必须将其转换为 State<T>，以便 Jetpack Compose 可以在状态发生变化时自动重组界面。

### [Compose 中的 实际应用]()
基于以上的知识点我们其实就可以开始开发一些简单的需求，像自定义layout，动画这些都是可以在用到的时候去学习的。
