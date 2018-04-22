---
title: 'React Native基础之props，state，style'
date: 2017-06-05 23:42:40
tags: [React Native]
categories: [Android,React Native]
---

### 基础
搭建开发环境就不说啦。

### 创建项目
搭建好开发环境之后，找个目录，创建项目，然后运行项目，创建出来的项目的gradle的版本如果本地没有的话，建议改成本地有的，不然要下载好久。
```
react-native init 项目名
cd 项目目录
react-native run-android
```
*提示:*你可以使用--version参数创建指定版本的项目。
例如react-native init MyApp --version 0.39.2。注意版本号必须精确到两个小数点。

还有一点，RN的菜单需要按菜单键才能出来，但是现在很多手机都没有菜单键啦，RN肯定早就考虑到啦这一点，摇一摇手机就出来啦。

### Hello World
```Javascript
import React, { Component } from 'react';
import { AppRegistry, Text } from 'react-native';

class HelloWorldApp extends Component {
  render() {
    return (
      <Text>Hello world!</Text>
    );
  }
}
// 注意，这里用引号括起来的'HelloWorldApp'必须和你init创建的项目名一致
AppRegistry.registerComponent('HelloWorldApp', () => HelloWorldApp);
```
RN的Hello World程序如上面所示的。
上面的示例代码中的import、from、class、extends、以及() =>箭头函数等新语法都是ES2015（也叫作ES6）中的特性。如果你不熟悉ES2015的话，可以看看[阮一峰老师的书](http://es6.ruanyifeng.com/)，还有论坛的这篇[总结](http://bbs.reactnative.cn/topic/15)。
*提示:*顺便说一句，阮一峰老师的博客写的真的很好，很多东西都讲的非常透彻。

上面的代码定义了一个名为`HelloWorldApp`的新的组件（Component），并且使用了名为`AppRegistry`的内置模块进行了“注册”操作。你在编写`React Native`应用时，肯定会写出很多新的组件。而一个App的最终界面，其实也就是各式各样的组件的组合。组件本身结构可以非常简单——唯一必须的就是在`render`方法中返回一些用于渲染结构的`JSX`语句。

`AppRegistry`模块则是用来告知`React Native`哪一个组件被注册为整个应用的根容器。你无需在此深究，因为一般在整个应用里`AppRegistry.registerComponent`这个方法只会调用一次。上面的代码里已经包含了具体的用法，你只需整个复制到index.ios.js或是index.android.js文件中即可运行。
*提示:*上面的话出自[编写Hello World](https://reactnative.cn/docs/0.44/tutorial.html#content)。

<!-- more -->

### props
大多数组件在创建时就可以使用各种参数来进行定制。用于定制的这些参数就称为`props`（属性）。所谓`props`，就是属性传递，而且是单向传递的。属性多的时候，可以传递一个对象，这是es6中的语法。
![图片文字组件](/images/pic_text_zj.png)
例如，我现在要定制一个这样的组件，上面显示图片，下面显示文字，当然知道官方的组件`Image`,`Text`已经足已完成这样的功能，我们现在为了展示props的用途这里来学习一下。
编写的代码大概是这样，当然其实也是利用官方的组件做的啦。
```Javascript
class DescText extends Component {
    render() {
        return (
            <Text>Desc: {this.props.txt}!</Text>
        );
    }
}

class BannerImg extends Component {
    render() {
        return (
            <Image source={this.props.img} style={{width: 300, height: 200}}/>
        );
    }
}

class BannerSample extends Component {
    render() {
        let va = {
            text: 'qingyong',
            uri: 'http://pic62.nipic.com/file/20150319/12632424_132215178296_2.jpg'
        };
        return (
            <View style={{alignItems: 'center'}}>
                <BannerImg img={va}/>
                <DescText txt={va.text}/>
            </View>
        );
    }
}
```
我们定制啦两个组件`DescText`,`BannerImg`分别来展示文字和图片，这里在两个组件里面分别定义了两个属性`txt`,`img`,然后在`BannerSample`里面去使用了这两个组件以及我们定义好的属性。属性是不是很简单:*定义，然后使用即可*。
**其实按我的理解:**
属性最好的用途是封装，我们可以通过把一些基础组件封装成一个`Component`，然后写好自定义的属性，封装成自己的基础组件，这样就算以后RN的api改变了，也不影响我们的使用，我们只需要修改组件即可，不用大肆修改项目。

比如上面的图片中的展示，我们虽然可以使用上面的方式制定两个组件，分别引用，或者直接使用RN的基础组件，不用封装，但是呢，如果比较多的地方都需要这种展示，我们就可以封装成一个上面显示图片，下面显示文字的组件。
示例如下:只是在刚才的基础上代码合并了一下而已。
```Javascript
class Banner extends Component {
    render() {
        return (
            <View style={{alignItems: 'center'}}>
                <Image source={this.props.img} style={{width: 300, height: 200}}/>
                <Text>Desc: {this.props.txt}!</Text>
            </View>
        );
    }
}

class BannerSample extends Component {
    render() {
        let va = {
            text: 'qingyong',
            uri: 'http://pic62.nipic.com/file/20150319/12632424_132215178296_2.jpg'
        };
        return (
            <View>
                <Banner img={va} txt={va.text}/>
            </View>
        );
    }
}
```
是不是感受到了封装的好处啦。

### state
React靠一个`state`来维护状态，当`state`发生变化则更新DOM。控制一个组件，一般有两种数据类型，一种是`props`，一种是`state`。`props`是在父组件中设置，一旦指定，它的生命周期是不可以改变的。对于组件中数据的变化，我们是通过`state`来控制的。
一般来说，你需要在`constructor`中初始化`state`（译注：这是ES6的写法，早期的很多ES5的例子使用的是`getInitialState`方法来初始化`state`，这一做法会逐渐被淘汰），然后在需要修改时调用`setState`方法。
这里还是展示官方的示例吧，但是其实我感觉官方的这个示例并不是特别好，没有把`state`的功能完全展示出来。
```Javascript
class HiddenText extends Component {
    constructor(props) {
        super(props);
        this.state = {
            showTxt: true
        };
        setInterval(
            () => {
                this.setState({showTxt: !this.state.showTxt});
            }
            , 1000);
    }

    render() {
        let display = this.state.showTxt ? this.props.text : '';
        return (
            <Text>{display}</Text>
        );
    }
}

class HiddenSample extends Component {
    render() {
        return (
            <View style={{alignItems: 'center'}}>
                <HiddenText text='one'/>
                <HiddenText text='two'/>
                <HiddenText text='three'/>
            </View>
        )
    };
}
```
首先，它是自定义了一个HiddenText的组件，在构造函数中初始化了state，然后写了一个定时器，每个1秒改变一次状态，然后setState,然后在渲染render()方法中，判断状态的变化，如果是true，显示文字，false显示空。这样一闪一闪的效果就出来了。

官方的这个示例也是简单的介绍了`state`怎么使用的，并不是特别详细的。比如在外面怎么主动调用`setState`的。

### style
在React Native中我们不需要使用什么特殊的语言或者语法去定义样式，我们还是使用JavaScript来写样式。所有的核心组件都接受名为`style`的属性。唯一的不同就是属性样式的命名使用了驼峰命名法，例如将`background-color`改为`backgroundColor`。

感觉这个对我这个css样式不熟悉的人来说，还是比较蛋疼的。

`style`属性可以是一个普通的JavaScript对象。这是最简单的用法，因而在示例代码中很常见。你还可以传入一个数组——在数组中位置居后的样式对象比居前的优先级更高，这样你可以间接实现样式的继承。

实际开发中组件的样式会越来越复杂，我们建议使用StyleSheet.create来集中定义组件的样式。
```Javascript
class LotsOfStyles extends Component {
    render() {
        return (
            <View>
                <Text style={styles.red}>just red</Text>
                <Text style={styles.bigblue}>just bigblue</Text>
                <Text style={[styles.bigblue, styles.red]}>bigblue, then red</Text>
                <Text style={[styles.red, styles.bigblue]}>red, then bigblue</Text>
            </View>
        );
    }
}

const styles = StyleSheet.create({
    bigblue: {
        color: 'blue',
        fontWeight: 'bold',
        fontSize: 30,
    },
    red: {
        color: 'red',
    },
});
```
![效果.png](/images/react_native_text_style.png)
可以很明显的看到后面的style会覆盖前面的。
做项目写代码的时候，还是要多看文档，熟悉了就会比较快啦。