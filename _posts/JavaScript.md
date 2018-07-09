---
title: 'JavaScript基础'
date: 2018-06-30 18:13:00
tags: [JavaScript]
categories: [JavaScript]
---
### 1.数据类型

JavaScript 语言的每一个值，都属于某一种数据类型。JavaScript 的数据类型，共有六种。(ES6 又新增了第七种 Symbol 类型的值)。
数值（number）：整数和小数（比如1和3.14）
字符串（string）：文本（比如Hello World）。
布尔值（boolean）：表示真伪的两个特殊值，即true（真）和false（假）
undefined：表示“未定义”或不存在，即由于目前没有定义，所以此处暂时没有任何值
null：表示空值，即此处的值为空。
对象（object）：各种值组成的集合。

### 2.对象
对象是最复杂的数据类型，又可以分为三个子类型
狭义的对象（object）
数组（array）
函数（function）
狭义的对象和数组是两种不同的数据组合方式。函数其实是处理数据的方法，JavaScript 把它当成一种数据类型，可以赋值给变量，这为编程带来了很大的灵活性，也为 JavaScript 的“函数式编程”奠定了基础。

#### 2.1.定义
什么是对象？简单说，对象就是一组“键值对”（key-value）的集合，是一种无序的复合数据集合。

对象的所有键名都是字符串（ES6 又引入了 Symbol 值也可以作为键名），所以加不加引号都可以。对象的每一个键名又称为“属性”（property），它的“键值”可以是任何数据类型。如果一个属性的值为函数，通常把这个属性称为“方法”，它可以像函数那样调用。对象的属性之间用逗号分隔，最后一个属性后面可以加逗号（trailing comma），也可以不加。

```javascript
var obj = {
    "name": '张三',
    "age": 15,
    "sayHello": function (name) {
        console.info("Hello my name is:" + name)
    }
};

var obj2 = {
    name: '张三',
    age: 15,
    sayHello: function (name) {
        console.info("Hello my name is:" + name)
    }
};
```

<!-- more -->

#### 2.2.对象还是语句
对象采用大括号表示，这导致了一个问题：如果行首是一个大括号，它到底是表达式还是语句？
```javascript
{p:123}
```
JavaScript 引擎读到上面这行代码，会发现可能有两种含义。第一种可能是，这是一个表达式，表示一个包含p属性的对象；第二种可能是，这是一个语句，表示一个代码区块，里面有一个标签p，指向表达式123。

为了避免这种歧义，V8 引擎规定，如果行首是大括号，一律解释为对象。不过，为了避免歧义，最好还是在大括号前加上圆括号。
```javascript
({p:123})
```

#### 2.3.属性的读取
读取对象的属性，有两种方法，一种是使用点运算符，还有一种是使用方括号运算符。请注意，如果使用方括号运算符，键名必须放在引号里面，否则会被当作变量处理。

查看一个对象本身的所有属性，可以使用Object.keys方法。

delete命令用于删除对象的属性，删除成功后返回true。

in运算符用于检查对象是否包含某个属性（注意，检查的是键名，不是键值），如果包含就返回true，否则返回false。

in运算符的一个问题是，它不能识别哪些属性是对象自身的，哪些属性是继承的。

### 3.数组
数组（array）是按次序排列的一组值。每个值的位置都有编号（从0开始），整个数组用方括号表示。任何类型的数据，都可以放入数组。

本质上，数组属于一种特殊的对象。typeof运算符会返回数组的类型是object。数组的特殊性体现在，它的键名是按次序排列的一组整数（0，1，2…）。

由于数组成员的键名是固定的（默认总是0、1、2…），因此数组不用为每个元素指定键名，而对象的每个成员都必须指定键名。JavaScript 语言规定，对象的键名一律为字符串，所以，数组的键名其实也是字符串。之所以可以用数值读取，是因为非字符串的键名会被转为字符串。

对象有两种读取成员的方法：点结构（object.key）和方括号结构（object[key]）。但是，对于数值的键名，不能使用点结构。arr.0的写法不合法，因为单独的数值不能作为标识符（identifier）。所以，数组成员只能用方括号arr[0]表示（方括号是运算符，可以接受数值）。

清空数组的一个有效方法，就是将length属性设为0。

### 4.函数
函数是一段可以反复调用的代码块。函数还能接受输入的参数，不同的参数会返回不同的值。

#### 4.1.函数声明
JavaScript 有三种声明函数的方法。
(1)function命令
function命令声明的代码区块，就是一个函数。function命令后面是函数名，函数名后面是一对圆括号，里面是传入函数的参数。函数体放在大括号里面。
```JavaScript
function print(s) {
  console.log(s);
}
```

(2)函数表达式
除了用function命令声明函数，还可以采用变量赋值的写法。这种写法将一个匿名函数赋值给变量。这时，这个匿名函数又称函数表达式（Function Expression），因为赋值语句的等号右侧只能放表达式。
```JavaScript
var print = function(s) {
  console.log(s);
};
```
采用函数表达式声明函数时，function命令后面不带有函数名。如果加上函数名，该函数名只在函数体内部有效，在函数体外部无效。

```JavaScript
var print = function x(){
  console.log(typeof x);
};
x
// ReferenceError: x is not defined
print()
// function
```
上面代码在函数表达式中，加入了函数名x。这个x只在函数体内部可用，指代函数表达式本身，其他地方都不可用。这种写法的用处有两个，一是可以在函数体内部调用自身，二是方便除错（除错工具显示函数调用栈时，将显示函数名，而不再显示这里是一个匿名函数）。因此，下面的形式声明函数也非常常见。
```JavaScript
var f = function f() {
};
```
需要注意的是，函数的表达式需要在语句的结尾加上分号，表示语句结束。而函数的声明在结尾的大括号后面不用加分号。总的来说，这两种声明函数的方式，差别很细微，可以近似认为是等价的。

(3)Function 构造函数
```JavaScript
var add = new Function('x', 'y', 'return x+y');
console.info(add(3, 2));
```
你可以传递任意数量的参数给Function构造函数，只有最后一个参数会被当做函数体，如果只有一个参数，该参数就是函数体。
总的来说，这种声明函数的方式非常不直观，几乎无人使用。

#### 4.2.圆括号运算符，return 语句和递归
调用函数要使用()运算符。()中可以加入函数的参数。
return语句不是必需的，如果没有的话，该函数就不返回任何值，或者说返回undefined。
函数可以调用自身，这就是递归（recursion）。

#### 4.3.第一等公民
JavaScript 语言将函数看作一种值，与其它值（数值、字符串、布尔值等等）地位相同。凡是可以使用值的地方，就能使用函数。比如，可以把函数赋值给变量和对象的属性，也可以当作参数传入其他函数，或者作为函数的结果返回。函数只是一个可以执行的值，此外并无特殊之处。

由于函数与其他数据类型地位平等，所以在 JavaScript 语言中又称函数为第一等公民。
```JavaScript
function add(x, y) {
    return x + y;
}
var addop = add;
function o(op) {
    return op;
}
console.info(o(addop(2,3)));
```

#### 4.4.函数名的提升
JavaScript 引擎将函数名视同变量名，所以采用function命令声明函数时，整个函数会像变量声明一样，被提升到代码头部。所以，下面的代码不会报错。
```JavaScript
f();
function f() {}
```

#### 4.5.不能在条件语句中声明函数
由于存在函数名的提升，所以在条件语句中声明函数，可能是无效的，这是非常容易出错的地方。要达到在条件语句中定义函数的目的，只有使用函数表达式。
```JavaScript
if (false) {
  var f = function () {};
}
f() // undefined
```

#### 4.6.函数内部的变量提升
作用域（scope）指的是变量存在的范围。在 ES5 的规范中，Javascript 只有两种作用域：一种是全局作用域，变量在整个程序中一直存在，所有地方都可以读取；另一种是函数作用域，变量只在函数内部存在。
注意，对于var命令来说，局部变量只能在函数内部声明，在其他区块中声明，一律都是全局变量。
与全局作用域一样，函数作用域内部也会产生“变量提升”现象。var命令声明的变量，不管在什么位置，变量声明都会被提升到函数体的头部。(只是变量声明提升到头部，而不是变量的赋值等操作提升到头部)
```JavaScript
function foo(x) {
  if (x > 100) {
    var tmp = x - 100;
  }
}
// 等同于
function foo(x) {
  var tmp;
  if (x > 100) {
    tmp = x - 100;
  };
}
```

#### 4.7.函数的作用域
函数本身也是一个值，也有自己的作用域。它的作用域与变量一样，就是其声明时所在的作用域，与其运行时所在的作用域无关。同样的，函数体内部声明的函数，作用域绑定函数体内部。

arguments对象包含了函数运行时的所有参数，arguments[0]就是第一个参数，arguments[1]就是第二个参数，以此类推。这个对象只有在函数体内部，才可以使用。

正常模式下，arguments对象可以在运行时修改。严格模式下，arguments对象是一个只读对象，修改它是无效的，但不会报错。('use strict'; // 开启严格模式)。函数的length属性返回函数预期传入的参数个数，即函数定义之中的参数个数。通过arguments对象的length属性，可以判断函数调用时到底带几个参数。

#### 4.8.闭包
理解闭包，首先必须理解变量作用域。前面提到，JavaScript 有两种作用域：全局作用域和函数作用域。函数内部可以直接读取全局变量。
如果出于种种原因，需要得到函数内的局部变量。正常情况下，这是办不到的，只有通过变通方法才能实现。那就是在函数的内部，再定义一个函数。
```JavaScript
function f1() {
  var n = 999;
  function f2() {
    console.log(n);
  }
  return f2;
}
var result = f1();
result(); // 999
```
上面代码中，函数f1的返回值就是函数f2，由于f2可以读取f1的内部变量，所以就可以在外部获得f1的内部变量了。

闭包就是函数f2，即能够读取其他函数内部变量的函数。由于在 JavaScript 语言中，只有函数内部的子函数才能读取内部变量，因此可以把闭包简单理解成“定义在一个函数内部的函数”。闭包最大的特点，就是它可以“记住”诞生的环境，比如f2记住了它诞生的环境f1，所以从f2可以得到f1的内部变量。在本质上，闭包就是将函数内部和函数外部连接起来的一座桥梁。

闭包的最大用处有两个，一个是可以读取函数内部的变量，另一个就是让这些变量始终保持在内存中，即闭包可以使得它诞生环境一直存在。请看下面的例子，闭包使得内部变量记住上一次调用时的运算结果。
```JavaScript
function createIncrementor(start) {
  return function () {
    return start++;
  };
}
var inc = createIncrementor(5);
inc() // 5
inc() // 6
inc() // 7
```
上面代码中，start是函数createIncrementor的内部变量。通过闭包，start的状态被保留了，每一次调用都是在上一次调用的基础上进行计算。从中可以看到，闭包inc使得函数createIncrementor的内部环境，一直存在。所以，闭包可以看作是函数内部作用域的一个接口。

为什么会这样呢？原因就在于inc始终在内存中，而inc的存在依赖于createIncrementor，因此也始终在内存中，不会在调用结束后，被垃圾回收机制回收。

闭包的另一个用处，是封装对象的私有属性和私有方法。
```JavaScript
function Person(name) {
    var _age;
    function setAge(n) {
        _age = n;
    }
    function getAge() {
        return _age;
    }
    return ({
        name: name,
        getAge: getAge,
        setAge: setAge
    });
}
var p = new Person("嘻嘻");
p.setAge(15);
console.info(p.getAge());
```

#### 4.9.立即调用的函数表达式
在 Javascript 中，圆括号()是一种运算符，跟在函数名之后，表示调用该函数。比如，print()就表示调用print函数。
有时，我们需要在定义函数之后，立即调用该函数。这时，你不能在函数的定义之后加上圆括号，这会产生语法错误。

产生这个错误的原因是，function这个关键字即可以当作语句，也可以当作表达式。为了避免解析上的歧义，JavaScript 引擎规定，如果function关键字出现在行首，一律解释成语句。因此，JavaScript引擎看到行首是function关键字之后，认为这一段都是函数的定义，不应该以圆括号结尾，所以就报错了。

解决方法就是不要让function出现在行首，让引擎将其理解成一个表达式。最简单的处理，就是将其放在一个圆括号里面。
```JavaScript
(function(){ /* code */ }());
// 或者
(function(){ /* code */ })();
```
当然不使用()也行，比如使用!。
上面两种写法都是以圆括号开头，引擎就会认为后面跟的是一个表示式，而不是函数定义语句，所以就避免了错误。这就叫做“立即调用的函数表达式”（Immediately-Invoked Function Expression），简称 IIFE。
注意，上面两种写法最后的分号都是必须的。如果省略分号，遇到连着两个 IIFE，可能就会报错。

#### 5.0.eval命令(执行下载的js)
eval命令的作用是，将字符串当作语句执行。
```JavaScript
eval('var a = 1;');
a // 1
```
此外，eval的命令字符串不会得到 JavaScript 引擎的优化，运行速度较慢。这也是一个不应该使用它的理由。
与eval作用类似的还有Function构造函数。利用它生成一个函数，然后调用该函数，也能将字符串当作命令执行。

### 5.错误
Error实例对象是最一般的错误类型，在它的基础上，JavaScript 还定义了其他6种错误对象。也就是说，存在Error的6个派生对象。
#### 5.1.SyntaxError
SyntaxError对象是解析代码时发生的语法错误。

#### 5.2.ReferenceError
ReferenceError对象是引用一个不存在的变量时发生的错误。另一种触发场景是，将一个值分配给无法分配的对象，比如对函数的运行结果或者this赋值。

#### 5.3.RangeError
RangeError对象是一个值超出有效范围时发生的错误。主要有几种情况，一是数组长度为负数，二是Number对象的方法参数超出范围，以及函数堆栈超过最大值。

#### 5.4.TypeError
TypeError对象是变量或参数不是预期类型时发生的错误。比如，对字符串、布尔值、数值等原始类型的值使用new命令，就会抛出这种错误，因为new命令的参数应该是一个构造函数。

#### 5.5.URIError
URIError对象是 URI 相关函数的参数不正确时抛出的错误，主要涉及encodeURI()、decodeURI()、encodeURIComponent()、decodeURIComponent()、escape()和unescape()这六个函数。

#### 5.6.EvalError
eval函数没有被正确执行时，会抛出EvalError错误。该错误类型已经不再使用了，只是为了保证与以前代码兼容，才继续保留。

#### 5.7.总结
以上这6种派生错误，连同原始的Error对象，都是构造函数。开发者可以使用它们，手动生成错误对象的实例。这些构造函数都接受一个函数，代表错误提示信息（message）。
```JavaScript
var err1 = new Error('出错了！');
var err2 = new RangeError('出错了，变量超出有效范围！');
var err3 = new TypeError('出错了，变量类型无效！');
err1.message // "出错了！"
err2.message // "出错了，变量超出有效范围！"
err3.message // "出错了，变量类型无效！"
```

#### 5.8.自定义错误
```JavaScript
function UserError(message) {
  this.message = message || '默认信息';
  this.name = 'UserError';
}
UserError.prototype = new Error();
UserError.prototype.constructor = UserError;
```

#### 5.9.其他
throw 语句，throw语句的作用是手动中断程序执行，抛出一个错误。
try…catch 结构，一旦发生错误，程序就中止执行了。JavaScript 提供了try...catch结构，允许对错误进行处理，选择是否往下执行。
finally 代码块，try...catch结构允许在最后添加一个finally代码块，表示不管是否出现错误，都必需在最后运行的语句。

### 6.编程风格
#### 6.1.区块
区块不要省略大括号，比如if语句的区块，等等。
区块大括号的位置不要另起一行写。因为 JavaScript 会自动添加句末的分号，导致一些难以察觉的错误。
```JavaScript
return
{
  key: value
};
// 相当于
return;
{
  key: value

};
```

#### 6.2.圆括号()
圆括号（parentheses）在 JavaScript 中有两种作用，一种表示函数的调用，另一种表示表达式的组合（grouping）。