---
title: 'JavaScript基础'
date: 2018-07-02 18:13:00
tags: [JavaScript]
categories: [JavaScript]
---
### 1.概念
典型的面向对象编程语言（比如 C++ 和 Java），都有“类”（class）这个概念。所谓“类”就是对象的模板，对象就是“类”的实例。但是，JavaScript 语言的对象体系，不是基于“类”的，而是基于构造函数（constructor）和原型链（prototype）。

JavaScript 语言使用构造函数（constructor）作为对象的模板。所谓”构造函数”，就是专门用来生成实例对象的函数。它就是对象的模板，描述实例对象的基本结构。一个构造函数，可以生成多个实例对象，这些实例对象都有相同的结构。
构造函数就是一个普通的函数，但是有自己的特征和用法。
```JavaScript
var Vehicle = function () {
  this.price = 1000;
};
```
上面代码中，Vehicle就是构造函数。为了与普通函数区别，构造函数名字的第一个字母通常大写。

<!-- more -->

构造函数的特点有两个:
1.函数体内部使用了this关键字，代表了所要生成的对象实例。
2.生成对象的时候，必须使用new命令。
为了防止调用者忘记使用new命令，可以使用下面两种方案:
1.构造函数使用严格模式
2.构造函数内部判断是否使用new命令
```JavaScript
function Fubar(foo, bar){
  //'use strict';
  if (!(this instanceof Fubar)) {
    return new Fubar(foo, bar);
  }
  this._foo = foo;
  this._bar = bar;
}
```

### 2.new命令
使用new命令时，它后面的函数依次执行下面的步骤。
1.创建一个空对象，作为将要返回的对象实例。
2.将这个空对象的原型，指向构造函数的prototype属性。
3.将这个空对象赋值给函数内部的this关键字。
4.开始执行构造函数内部的代码。

如果构造函数内部有return语句，而且return后面跟着一个对象，new命令会返回return语句指定的对象；否则，就会不管return语句，返回this对象。如果return语句返回的是一个跟this无关的新对象，new命令会返回这个新对象，而不是this对象。这一点需要特别引起注意。

如果对普通函数（内部没有this关键字的函数）使用new命令，则会返回一个空对象。

函数内部可以使用new.target属性。如果当前函数是new命令调用，new.target指向当前函数，否则为undefined。使用这个属性，可以判断函数调用的时候，是否使用new命令。
构造函数作为模板，可以生成实例对象。但是，有时拿不到构造函数，只能拿到一个现有的对象。我们希望以这个现有的对象作为模板，生成新的实例对象，这时就可以使用Object.create()方法。

### 3.this
简单说，this就是属性或方法“当前”所在的对象。
1.全局环境使用this，它指的就是顶层对象window。
2.构造函数中的this，指的是实例对象。
3.如果对象的方法里面包含this，this的指向就是方法运行时所在的对象。该方法赋值给另一个对象，就会改变this的指向。
4.由于this的指向是不确定的，所以切勿在函数中包含多层的this。嵌套情况指向全局对象window。
5.数组的map和foreach方法，允许提供一个函数作为参数。这个函数内部不应该使用this。
6.回调函数中的this往往会改变指向，最好避免使用。

### 3.1.绑定this的方法
1.Function.prototype.call()
函数实例的call方法，可以指定函数内部this的指向（即函数执行时所在的作用域），然后在所指定的作用域中，调用该函数。
call方法的参数，应该是一个对象。如果参数为空、null和undefined，则默认传入全局对象。
```JavaScript
var n = 123;
var obj = { n: 456 };
function a() {
  console.log(this.n);
}
a.call() // 123
a.call(null) // 123
a.call(undefined) // 123
a.call(window) // 123
a.call(obj) // 456
```
call方法的一个应用是调用对象的原生方法。
```JavaScript
var obj = {};
obj.hasOwnProperty('toString') // false
// 覆盖掉继承的 hasOwnProperty 方法
obj.hasOwnProperty = function () {
  return true;
};
obj.hasOwnProperty('toString') // true
Object.prototype.hasOwnProperty.call(obj, 'toString') // false
```
上面代码中，hasOwnProperty是obj对象继承的方法，如果这个方法一旦被覆盖，就不会得到正确结果。call方法可以解决这个问题，它将hasOwnProperty方法的原始定义放到obj对象上执行，这样无论obj上有没有同名方法，都不会影响结果。

2.Function.prototype.apply()
apply方法的作用与call方法类似，也是改变this指向，然后再调用该函数。唯一的区别就是，它接收一个数组作为函数执行时的参数，使用格式如下。
`func.apply(thisValue, [arg1, arg2, ...])`
apply方法的第一个参数也是this所要指向的那个对象，如果设为null或undefined，则等同于指定全局对象。第二个参数则是一个数组，该数组的所有成员依次作为参数，传入原函数。原函数的参数，在call方法中必须一个个添加，但是在apply方法中，必须以数组形式添加。
```JavaScript
function f(x, y){
  console.log(x + y);
}
f.call(null, 1, 1) // 2
f.apply(null, [1, 1]) // 2
```
3.Function.prototype.bind()
bind方法用于将函数体内的this绑定到某个对象，然后返回一个新函数。

### 3.prototype
大部分面向对象的编程语言，都是通过“类”（class）来实现对象的继承。JavaScript 语言的继承则是通过“原型对象”（prototype）。
#### 3.1.构造函数的缺点
JavaScript 通过构造函数生成新对象，因此构造函数可以视为对象的模板。实例对象的属性和方法，可以定义在构造函数内部。
通过构造函数为实例对象定义属性，虽然很方便，但是有一个缺点。同一个构造函数的多个实例之间，无法共享属性，从而造成对系统资源的浪费。

#### 3.2.prototype 属性的作用
JavaScript 继承机制的设计思想就是，原型对象的所有属性和方法，都能被实例对象共享。也就是说，如果属性和方法定义在原型上，那么所有实例对象就能共享，不仅节省了内存，还体现了实例对象之间的联系。
JavaScript 规定，每个函数都有一个prototype属性，指向一个对象。对于普通函数来说，该属性基本无用。但是，对于构造函数来说，生成实例的时候，该属性会自动成为实例对象的原型。
```JavaScript
function Animal(name) {
  this.name = name;
}
Animal.prototype.color = 'white';
var cat1 = new Animal('大毛');
var cat2 = new Animal('二毛');
cat1.color // 'white'
cat2.color // 'white'
```
原型对象的属性不是实例对象自身的属性。只要修改原型对象，变动就立刻会体现在所有实例对象上。当实例对象本身没有某个属性或方法的时候，它会到原型对象去寻找该属性或方法。这就是原型对象的特殊之处。如果实例对象自身就有某个属性或方法，它就不会再去原型对象寻找这个属性或方法。
总结:原型对象的作用，就是定义所有实例对象共享的属性和方法。这也是它被称为原型对象的原因，而实例对象可以视作从原型对象衍生出来的子对象。

#### 3.3.原型链
JavaScript 规定，所有对象都有自己的原型对象（prototype）。一方面，任何一个对象，都可以充当其他对象的原型；另一方面，由于原型对象也是对象，所以它也有自己的原型。因此，就会形成一个“原型链”（prototype chain）：对象到原型，再到原型的原型……
如果一层层地上溯，所有对象的原型最终都可以上溯到Object.prototype，即Object构造函数的prototype属性。也就是说，所有对象都继承了Object.prototype的属性。这就是所有对象都有valueOf和toString方法的原因，因为这是从Object.prototype继承的。
那么，Object.prototype对象有没有它的原型呢？回答是Object.prototype的原型是null。null没有任何属性和方法，也没有自己的原型。因此，原型链的尽头就是null。
```JavaScript
Object.getPrototypeOf(Object.prototype) //Object.getPrototypeOf方法返回参数对象的原型
// null
```

#### 3.4.constructor 属性
prototype对象有一个constructor属性，默认指向prototype对象所在的构造函数。
```JavaScript
function P() {}
var p = new P();
p.constructor === P // true
p.constructor === P.prototype.constructor // true
p.hasOwnProperty('constructor') // false
```
上面代码中，p是构造函数P的实例对象，但是p自身没有constructor属性，该属性其实是读取原型链上面的P.prototype.constructor属性。
constructor属性表示原型对象与构造函数之间的关联关系，如果修改了原型对象，一般会同时修改constructor属性，防止引用的时候出错。

#### 3.5.instanceof 运算符
instanceof运算符返回一个布尔值，表示对象是否为某个构造函数的实例。
instanceof运算符的左边是实例对象，右边是构造函数。它会检查右边构建函数的原型对象（prototype），是否在左边对象的原型链上。因此，下面两种写法是等价的。
```JavaScript
v instanceof Vehicle
// 等同于
Vehicle.prototype.isPrototypeOf(v)
```
注意，instanceof运算符只能用于对象，不适用原始类型的值。

#### 3.6.Object 对象的相关方法
##### 3.6.1.Object.getPrototypeOf方法返回参数对象的原型。这是获取原型对象的标准方法。
```JavaScript
var F = function () {};
var f = new F();
Object.getPrototypeOf(f) === F.prototype // true
// 空对象的原型是 Object.prototype
Object.getPrototypeOf({}) === Object.prototype // true
// Object.prototype 的原型是 null
Object.getPrototypeOf(Object.prototype) === null // true
// 函数的原型是 Function.prototype
function f() {}
Object.getPrototypeOf(f) === Function.prototype // true
```

##### 3.6.2.Object.setPrototypeOf方法为参数对象设置原型，返回该参数对象。它接受两个参数，第一个是现有对象，第二个是原型对象。
new命令可以使用Object.setPrototypeOf方法模拟。
```JavaScript
var F = function () {
  this.foo = 'bar';
};
var f = new F();
// 等同于
var f = Object.setPrototypeOf({}, F.prototype);
F.call(f);
```
上面代码中，new命令新建实例对象，其实可以分成两步。第一步，将一个空对象的原型设为构造函数的prototype属性（上例是F.prototype）；第二步，将构造函数内部的this绑定这个空对象，然后执行构造函数，使得定义在this上面的方法和属性（上例是this.foo），都转移到这个空对象上。

##### 3.6.3.Object.create()方法接受一个对象作为参数，然后以它为原型，返回一个实例对象。该实例完全继承原型对象的属性。

##### 3.6.4.Object.prototype.isPrototypeOf()用来判断该对象是否为参数对象的原型。
只要实例对象处在参数对象的原型链上，isPrototypeOf方法都返回true。

##### 3.6.5.Object.prototype.__proto__实例对象的__proto__属性（前后各两个下划线），返回该对象的原型。该属性可读写。
根据语言标准，__proto__属性只有浏览器才需要部署，其他环境可以没有这个属性。它前后的两根下划线，表明它本质是一个内部属性，不应该对使用者暴露。因此，应该尽量少用这个属性，而是用Object.getPrototypeof()和Object.setPrototypeOf()，进行原型对象的读写操作。

##### 3.6.6.获取原型对象方法
```JavaScript
obj.__proto__
obj.constructor.prototype
Object.getPrototypeOf(obj)
```
上面三种方法之中，前两种都不是很可靠。__proto__属性只有浏览器才需要部署，其他环境可以不部署。而obj.constructor.prototype在手动改变原型对象时，可能会失效。

##### 3.6.7.Object.getOwnPropertyNames()
Object.getOwnPropertyNames方法返回一个数组，成员是参数对象本身的所有属性的键名，不包含继承的属性键名。
对象本身的属性之中，有的是可以遍历的（enumerable），有的是不可以遍历的。Object.getOwnPropertyNames方法返回所有键名，不管是否可以遍历。只获取那些可以遍历的属性，使用Object.keys方法。

##### 3.6.8.Object.prototype.hasOwnProperty()
对象实例的hasOwnProperty方法返回一个布尔值，用于判断某个属性定义在对象自身，还是定义在原型链上。hasOwnProperty方法是 JavaScript 之中唯一一个处理对象属性时，不会遍历原型链的方法。

##### 3.6.9.in 运算符和 for…in 循环
in运算符返回一个布尔值，表示一个对象是否具有某个属性。它不区分该属性是对象自身的属性，还是继承的属性。in运算符常用于检查一个属性是否存在。获得对象的所有可遍历属性（不管是自身的还是继承的），可以使用for...in循环。
获得对象的所有属性（不管是自身的还是继承的，也不管是否可枚举），可以使用下面的函数。
```JavaScript
function inheritedPropertyNames(obj) {
  var props = {};
  while(obj) {
    Object.getOwnPropertyNames(obj).forEach(function(p) {
      props[p] = true;
    });
    obj = Object.getPrototypeOf(obj);
  }
  return Object.getOwnPropertyNames(props);
}
```

##### 3.6.10.对象的拷贝
1.确保拷贝后的对象，与原对象具有同样的原型。
2.确保拷贝后的对象，与原对象具有同样的实例属性。
```JavaScript
function copyObject(orig) {
  var copy = Object.create(Object.getPrototypeOf(orig));
  copyOwnPropertiesFrom(copy, orig);
  return copy;
}

function copyOwnPropertiesFrom(target, source) {
  Object
    .getOwnPropertyNames(source)
    .forEach(function (propKey) {
      var desc = Object.getOwnPropertyDescriptor(source, propKey);
      Object.defineProperty(target, propKey, desc);
    });
  return target;
}
```
另一种更简单的写法，是利用 ES2017 才引入标准的Object.getOwnPropertyDescriptors方法。
```JavaScript
function copyObject(orig) {
  return Object.create(
    Object.getPrototypeOf(orig),
    Object.getOwnPropertyDescriptors(orig)
  );
}
```