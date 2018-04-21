---
title: 'Jenkins自动打包配置'
date: 2016-08-12 23:43:40
tags: [Jenkins]
categories: [Android,Jenkins]
---

当时也是花费了不少时间来配置Jenkins自动打包的问题，觉得还是需要记录一下。
1.安装Jenkins，这个很简单，不需要多说。
2.下载Git Plugin，Gradle Plugin，Android Emulator Plugin（这个可以配置SDK路径，觉得这个插件挺好），
Email Extension Plugin 邮件提醒插件，自带的邮件提醒插件确实太弱。
3.配置，SDK路径，JDK路径，Git路径，Gradle路径。

配置git:
git config --global user.name "name"
git config --global user.email email
查看:
cat /root/.gitconfig
root是指当前的用户

生成公钥和私钥:
ssh-keygen -t rsa -C "email"

邮件配置:
Default Subject
```
构建通知:$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!
```

Default Content
```html
<hr/>

(本邮件是程序自动下发的，请勿回复！)<br/><hr/>

项目名称：$PROJECT_NAME<br/><hr/>

构建编号：$BUILD_NUMBER<br/><hr/>

构建状态：$BUILD_STATUS<br/><hr/>

触发原因：${CAUSE}<br/><hr/>

构建日志地址：<a href="${BUILD_URL}console">${BUILD_URL}console</a><br/><hr/>

构建地址：<a href="$BUILD_URL">$BUILD_URL</a><br/><hr/>

变更集:${JELLY_SCRIPT,template="html"}<br/><hr/>
```

4.项目配置
4.1构建一个自由风格的软件项目
4.2git地址配置，我们是专门创建一个用户用来拉取项目打包，如果使用ssh记得配置私钥和公钥（我们是使用gitblit，需要在gitblit上面配置公钥，Jenkins里面配置私钥）

分支配置

Additional Behaviours
高级配置，clone时间等

4.3Poll SCM，触发器配置
H/30 23 * * 1-5 表示星期一到星期五每天晚上23:30分构建一次。
第一个参数代表的是分钟 minute，取值 0~59；
第二个参数代表的是小时 hour，取值 0~23；
第三个参数代表的是天 day，取值 1~31；
第四个参数代表的是月 month，取值 1~12；
第五个参数代表的是星期 week，取值 0~7，0 和 7 都是表示星期天。
如H/5 表示的就是每5分钟检查一次源码变化。

4.4配置gradle task
由于每个项目，每个修改的用户都不一样，所以我们的gradle.properties是配置在电脑的user/.gradle/gradle.properties这里的，这个文件里面配置了我们的私有maven库的地址，keystore的位置，所以呢，我们项目里面的gradle.properties是空的，会单独写个config.gradle文件来重新生成这个文件。这个文件执行可以防止setting.gradle里面，因为这个是最先执行的，但是这样的话，会有干扰，因为只有Jenkins打包的时候才会需要重新修改gradle.properties文件，所以呢，我们这个文件是独立的。

配置gradle命令执行config.gradle文件里面的task
-q -b config.gradle taskname 大概是这样

配置clean等命令
clean build

执行完成之后可以在写个gradle文件去执行把打包好的apk文件拷贝到指定的地点。

4.5邮件配置
主要是高级设置里面，需要配置，Triggers，触发什么时候需要发邮件。

哈哈，当然，我这篇文章写的简短，但是其实过程中遇到很多问题，大概折腾两三天，Win上面还好，Linux上面会有很多依赖库的问题，我是分别在Linux和Windows上面都调试好了滴。

遇到问题可以看[http://www.jianshu.com/p/c1b1b2817d90](http://www.jianshu.com/p/c1b1b2817d90)这篇文章，写的很好。