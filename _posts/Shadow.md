---
title: 'Shadow插件化框架使用'
date: 2023-08-03 20:18:00
tags: [插件化]
categories: [Android,插件化]
---

### 说明
最近项目想要做模块动态升级，所以了解了最近还在维护的插件化框架[Shadow](https://github.com/Tencent/Shadow/issues/245).

shadow框架的官网的顶置[issue](https://github.com/Tencent/Shadow/issues/245)，里面有非常多的关于框架的解析的文章，文字。想要了解此框架，这个必看。

这里还是截取一张项目代码图。
![项目代码图](/images/shadow_project_pic.png)

### 项目解读
shadow框架为了实现复杂的插件化框架本身也动态升级，做了很多复杂操作：
#### 宿主本身只跟plugin-manager插件交互

来说一下plugin-manager插件，依赖core-manager，dynamic-manager。
core-manager:
1、插件信息的存储
2、插件信息的管理
3、 so、dex管理
4、插件包zip释放

dynamic-manager:
1、只提供最基础的 dex、 res、so 的释放的基础API，这些 API 的组合调用需要自己实现
2、只负责加载 业务插件运行需要的 loader 和runtime 插件，业务插件的加载由 loader 插件实
现

宿主和manager插件交互，是直接通过构造`ApkClassLoader`，加载manager插件，构造插件里面的`PluginManagerImpl`对象。具体可以看`ManagerImplLoader`类。

在构造`PluginManagerImpl`对象的时候，是通过调用manager插件固定类里面的固定方法`com.tencent.shadow.dynamic.impl.ManagerFactoryImpl#buildManager`，然后这个`PluginManagerImpl`最终也是我们自己实现的。

我们需要实现`PluginManagerImpl`，然后根据不同的意图，比如打开activity，启动service，来调用不同的`core-manager`，或者`dynamic-manager`的方法，比如安装插件、打开插件activity之类的。

总体而言，自由度比较大，但是弊端也很明显，我们自己也要做很多的工作。

<!-- more -->

#### 调用插件类，需要通过manager插件和插件zip包里面的loader插件交互
对目前的shadow来说，宿主和manager插件在一个进程，插件和加载插件的loader插件在另一个进程。
所以目前调用插件类需要通过ipc的方式和loader插件交互。manager插件调用到loader插件之后，loader插件通过加载固定类的固定方法`com.tencent.shadow.dynamic.loader.impl.CoreLoaderFactoryImpl#build`，去构造`ShadowPluginLoader`插件加载逻辑类，我们需要在这里面去配置宿主占坑组件和插件组件的对应关系。

总体而言，自由度比较大，但是弊端也很明显，我们自己也要做很多的工作。这里例如`VirtualApk`框架，是根据解析插件的组件在manifest里面的配置，去自动寻找宿主合适的组件的，如果这个逻辑还得我们自己实现的话，也很麻烦。还有个问题在配置宿主占坑组件和插件里面的对应关系的时候，框架给的参数太少了，例如:
```
public ComponentName onBindContainerActivity(ComponentName pluginActivity) {
    switch (pluginActivity.getClassName()) {
        /**
          * 这里配置对应的对应关系
          */
    }
    return new ComponentName(context, DEFAULT_ACTIVITY);
}
```
就拿这个方法来说，插件调用只传递来了一个`ComponentName`对象，里面有用的信息只有`ClassName`，我怎么根据一个`ClassName`去知道这个插件activity应该使用宿主的哪个占坑activity去对应呢，一个个的if else写死嘛，起码我要知道这个插件activity的启动模式，配置的主题等等参数，才能决定，所以这里设计的很不合理。可能shadow的逻辑是插件更新了，loader插件也要更新，所以写if else也没问题。

#### 插件打包问题
shadow打包插件，对于manager插件来说就是一个单独的apk，打包之后加载即可，对于业务插件来说就麻烦了，业务插件想要加载需要有loader插件和runtime插件，难道我们每一个业务插件都需要带一个loader插件和runtime插件嘛，虽然loader插件和runtime的插件代码也确实比较小，每个业务插件有一个其实问题也不大，不过如果loader和runtime的代码都差不多的话，还是感觉不好，根据在issue里面找到的方案，shadow是使用UUID相同表示一组apk可以共用工作。这组apk里可以有一个runtime一个loader和多个插件apk。
基于此，如果我们有一些插件可以共用一组loader和runtime的话，可以只在某一个插件zip里面打包loader和runtime，其他的插件不打包，但是他们的uuid必须相同。
可以看这些issue:
https://github.com/Tencent/Shadow/issues/457
https://github.com/Tencent/Shadow/issues/743
具体配置如下:
```
//common插件里面包含了runtime和loader
shadow {
    transform {
        //useHostContext = ['abc']
    }
    packagePlugin {
        pluginTypes {
            debug {
                loaderApkConfig = new Tuple2('plugin_loader-debug.apk', ':plugin_loader:assembleDebug')
                runtimeApkConfig = new Tuple2('plugin_runtime-debug.apk', ':plugin_runtime:assembleDebug')
                pluginApks {
                    plugin_1 {
                        //businessName相同的插件，context获取的Dir是相同的。businessName留空，表示和宿主相同业务，直接使用宿主的Dir
                        businessName = ''
                        partKey = 'plugin_common'
                        buildTask = 'assemblePluginDebug'
                        apkPath = 'plugin_common_app/build/outputs/apk/plugin/debug/plugin_common_app-plugin-debug.apk'
                        hostWhiteList = ["com.blankj.utilcode.util",
                                         "com.blankj.utilcode.constant",
                        ]
                        //dependsOn = ['']
                    }
                }
            }

            release {
                loaderApkConfig = new Tuple2('plugin_loader-release.apk', ':plugin_loader:assembleRelease')
                runtimeApkConfig = new Tuple2('plugin_runtime-release.apk', ':plugin_runtime:assembleRelease')
                pluginApks {
                    plugin_1 {
                        businessName = ''
                        partKey = 'plugin_common'
                        buildTask = 'assemblePluginRelease'
                        apkPath = 'plugin_common_app/build/outputs/apk/plugin/debug/plugin_common_app-plugin-release.apk'
                        hostWhiteList = ["com.blankj.utilcode.util",
                                         "com.blankj.utilcode.constant",
                        ]
                        //dependsOn = ['']
                    }
                }
            }
        }

        uuid = "123567"
        loaderApkProjectPath = 'plugin_loader'
        runtimeApkProjectPath = 'plugin_runtime'

        archiveSuffix = System.getenv("PluginSuffix") ?: ""
        archivePrefix = 'plugin_common'
        destinationDir = "${getRootProject().getBuildDir()}"

        version = 1
        compactVersion = [1]
        uuidNickName = "1.0.0"

    }
}
```
然后插件A里面如下配置:
```
shadow {
    transform {
        //useHostContext = ['abc']
    }
    packagePlugin {
        pluginTypes {
            debug {
                //这里不配置，最终的zip包里面就不会有loader和runtime了
                //loaderApkConfig = new Tuple2('plugin_loader-debug.apk', ':plugin_loader:assembleDebug')
                //runtimeApkConfig = new Tuple2('plugin_runtime-debug.apk', ':plugin_runtime:assembleDebug')
                pluginApks {
                    plugin_a {
                        //businessName相同的插件，context获取的Dir是相同的。businessName留空，表示和宿主相同业务，直接使用宿主的Dir
                        businessName = ''
                        partKey = 'plugin_a'
                        buildTask = 'assemblePluginDebug'
                        apkPath = 'plugina/build/outputs/apk/plugin/debug/plugina-plugin-debug.apk'
                        hostWhiteList = ["com.blankj.utilcode.util",
                                         "com.blankj.utilcode.constant",
                        ]
                        dependsOn = ['plugin_common']
                    }
                }
            }

            release {
                //loaderApkConfig = new Tuple2('plugin_loader-release.apk', ':plugin_loader:assembleRelease')
                //runtimeApkConfig = new Tuple2('plugin_runtime-release.apk', ':plugin_runtime:assembleRelease')
                pluginApks {
                    plugin_a {
                        businessName = ''
                        partKey = 'plugin_a'
                        buildTask = 'assemblePluginRelease'
                        apkPath = 'plugina/build/outputs/apk/plugin/debug/plugina-plugin-release.apk'
                        hostWhiteList = ["com.blankj.utilcode.util",
                                         "com.blankj.utilcode.constant",
                        ]
                        dependsOn = ['plugin_common']
                    }
                }
            }
        }

        uuid = "123567"
        loaderApkProjectPath = 'plugin_loader'
        runtimeApkProjectPath = 'plugin_runtime'

        archiveSuffix = System.getenv("PluginSuffix") ?: ""
        archivePrefix = 'plugina'
        destinationDir = "${getRootProject().getBuildDir()}"

        version = 1
        compactVersion = [1]
        uuidNickName = "1.0.0"

    }
}
```

#### 插件依赖问题
shadow block里面的配置，可以通过hostWhiteList配置可以访问宿主的哪些类。但是还是有一些情况需要注意。
- 插件依赖通过参数dependsOn控制，可以是多个，内容填写插件的partKey
- 可以通过在参数hostWhiteList配置可以访问宿主的类，默认情况，插件不能访问宿主
- 插件A dependsOn 插件B，那么插件Shadow会将插件B的ClassLoader作为插件A的parent
- 插件A dependsOn 插件B，那么插件A配置的hostWhiteList就不起作用了，需要在插件B里面配置
- 插件A dependsOn 插件B，目前并不支持插件A访问插件B的资源
- 宿主要访问插件里面的类比较麻烦

具体官方这篇文章也有介绍[Shadow对插件包管理的设计](https://juejin.cn/post/6844903893130821640).

### 具体使用
综合上面的一些描述，我们其实是可以发现，shadow插件化框架是有不少问题的，官方自己的介绍文章里面也说了一些，总体要是直接使用起来其实是很不方便的。
使用shadow，我们最看中的是实现插件化还是没用什么反射。那我们可以按照自己要求进行二次定制。

#### nodynamic模式
官方Demo里面其实有nodynamic的[sample](https://github.com/Tencent/Shadow/blob/master/projects/test/none-dynamic/host/test-none-dynamic-host)的。所谓nodynamic就是插件化框架本身不需要升级，我们直接在宿主里面加载插件。对于shadow来说，就是不需要manager插件了，把loader和runtime插件打包到宿主里面。
我们封装一个sdk给宿主使用，sdk里面直接包含loader和runtime。

##### 首先引入依赖：
```
//把loader和runtime打包到宿主，不用插件框架自身的升级
//common
implementation "com.tencent.shadow.core:common:$shadow_version"
//包含core:runtime和core:load-parameters
implementation "com.tencent.shadow.core:loader:$shadow_version"
implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:1.5.32"
//承载插件的容器，runtime
implementation "com.tencent.shadow.core:activity-container:$shadow_version"
//数据库管理插件的
implementation "com.tencent.shadow.core:manager:$shadow_version"
```
这里之所以我们引入了manger，是因为后续封装过程使用到了manger里面的一些封装好的数据结构。

后续就是一些对shadow的loader sdk的一些封装了。这里就不展示代码了。

##### 对gradle插件进行修改
这一节的内容假定你已经会写gradle插件了，不会的话需要先了解这方面的知识。

由于我们把loader和runtime打入宿主了，不需要之前复杂的插件信息了。但是我们仍然需要知道当前加载的插件的插件信息，没有插件信息怎么去加载呢。我们最终最少只需要如下的插件信息即可。
```
shadow {
    pluginInfo {
        pluginKey = 'plugina'
        version = android.defaultConfig.versionCode
        hostWhiteList = [
                "com.blankj.utilcode.util",
                "com.blankj.utilcode.constant",
        ]
        dependsOn = [
                "plugin_common_app"
        ]
    }
}
```
然后我们需要修改shadow的gradle插件，在构建完成插件apk之后，随即生成插件信息的json。
```
class ShadowPlugin : Plugin<Project> {

    ...

    override fun apply(project: Project) {
        project.afterEvaluate {
            onEachPluginVariant(project) { pluginVariant ->
                checkAaptPackageIdConfig(pluginVariant)
                val appExtension: AppExtension = project.extensions.getByType(AppExtension::class.java)

                //这里是我们新增的代码，其他代码没改
                createPluginInfoTasks(project, shadowExtension, pluginVariant)

                createGeneratePluginManifestTasks(project, appExtension, pluginVariant)
            }
        }
    }

    /**
     * 创建根据用户的配置生成插件信息的task
     */
    private fun createPluginInfoTasks(
        project: Project, shadowExtension: ShadowExtension, pluginVariant: ApplicationVariant
    ) {
        val extension = shadowExtension.pluginInfo
        if (extension.pluginKey.isNotBlank()) {
            //System.err.println("${project.name} pluginInfo===>$extension")
            pluginVariant.outputs?.all { output ->
                //因为前面已经过滤过了，所有这里基本一定是ApkVariantOutputImpl
                if (output is ApkVariantOutputImpl) {
                    //NormalDebug
                    val full = pluginVariant.name.capitalize()
                    //Normal
                    val favor = pluginVariant.flavorName.capitalize()
                    //Debug
                    val type = pluginVariant.buildType.name.capitalize()
                    //System.err.println("name=$full output=${output.outputFile.absolutePath}")
                    //assembleNormalDebug
                    val assembleTask = project.tasks.getByName("assemble$full")
                    assembleTask.doFirst { task ->
                        //直接在doFirst里面操作即可
                        //System.err.println("${task.name} doFirst")
                        //{
                        //    "partKey": "",
                        //    "apkName": "",
                        //    "version": 100,
                        //    "dependsOn": ["",""],
                        //    "hostWhiteList": ["",""]
                        //}
                        //写入outputs的config.json
                        val config = JSONObject()
                        config["pluginKey"] = extension.pluginKey
                        config["apkName"] = output.outputFile.name
                        config["version"] = extension.version
                        if (extension.dependsOn.isNotEmpty()) {
                            val dependsOnJson = JSONArray()
                            for (k in extension.dependsOn) {
                                dependsOnJson.add(k)
                            }
                            config["dependsOn"] = dependsOnJson
                        }
                        if (extension.hostWhiteList.isNotEmpty()) {
                            val hostWhiteListJson = JSONArray()
                            for (k in extension.hostWhiteList) {
                                hostWhiteListJson.add(k)
                            }
                            config["hostWhiteList"] = hostWhiteListJson
                        }
                        val file = File(output.outputFile.parentFile, "config.json")
                        //System.err.println("config json file=" + file.absolutePath)
                        project.logger.info("config json file=" + file.absolutePath)
                        val bizWriter = BufferedWriter(FileWriter(file))
                        bizWriter.write(config.toJSONString())
                        bizWriter.flush()
                        bizWriter.close()
                    }
                }
            }
        }
    }

}
```
当然ShadowExtension我们需要修改
```
open class ShadowExtension {
    var transformConfig = TransformConfig()
    fun transform(action: Action<in TransformConfig>) {
        action.execute(transformConfig)
    }

    var pluginInfo = PluginInfoConfig()
    fun pluginInfo(action: Action<in PluginInfoConfig>) {
        action.execute(pluginInfo)
    }
}

//新增PluginInfoConfig类

open class PluginInfoConfig {
    /**
     * 插件我们认为key是唯一的
     */
    var pluginKey = ""
    var apkName = ""

    /**
     * 插件的版本每次如果升级的话，表示是一个新插件
     */
    var version = -1
    var dependsOn: Array<String> = emptyArray()
    var hostWhiteList: Array<String> = emptyArray()

    constructor() {
    }

}
```

这样我们即在assemblePluginRelease(Debug)的时候生成了插件信息json，路径和生成apk的路径在同一个位置/build/outputs/plugin/release(debug)/config.json。
```
{"apkName":"plugina-plugin-debug.apk","dependsOn":["plugin_common_app"],"pluginKey":"plugina","hostWhiteList":["com.blankj.utilcode.util","com.blankj.utilcode.constant"],"version":100}
```

##### 修改CreateResourceBloc支持插件依赖插件的时候也能依赖插件的资源。
修改CreateResourceBloc即可。
```
object CreateResourceBloc {

    /**
     * 现在插件不能
     */
    fun create(
        archiveFilePath: String,
        hostAppContext: Context,
        loadParameters: LoadParameters,
        pluginPartsMap: MutableMap<String, PluginParts>
    ): Resources {
        ...
        if (Build.VERSION.SDK_INT > MAX_API_FOR_MIX_RESOURCES) {
            fillApplicationInfoForNewerApi(
                applicationInfo,
                hostApplicationInfo,
                archiveFilePath,
                loadParameters,
                pluginPartsMap
            )
        } else {
            fillApplicationInfoForLowerApi(
                applicationInfo,
                hostApplicationInfo,
                archiveFilePath,
                loadParameters,
                pluginPartsMap
            )
        }
        ...
    }

    private fun fillApplicationInfoForNewerApi(
        applicationInfo: ApplicationInfo,
        hostApplicationInfo: ApplicationInfo,
        pluginApkPath: String,
        loadParameters: LoadParameters,
        pluginPartsMap: MutableMap<String, PluginParts>
    ) {
        ...
        // hostSharedLibraryFiles中可能有webview通过私有api注入的webview.apk
        val hostSharedLibraryFiles = hostApplicationInfo.sharedLibraryFiles
        val paths = arrayListOf<String>()
        val dependsOn = loadParameters.dependsOn
        if (dependsOn != null && dependsOn.isNotEmpty()) {
            dependsOn.forEach {
                pluginPartsMap[it]?.apply {
                    paths.add(pluginPackageManager.archiveFilePath)
                }
            }
        }
        val otherApksAddToResources =
            if (hostSharedLibraryFiles == null)
                arrayOf(
                    *paths.toTypedArray(),
                    pluginApkPath
                )
            else
                arrayOf(
                    *hostSharedLibraryFiles,
                    *paths.toTypedArray(),
                    pluginApkPath
                )

        applicationInfo.sharedLibraryFiles = otherApksAddToResources
    }

    /**
     * API 25及以下系统，单独构造插件资源
     */
    private fun fillApplicationInfoForLowerApi(
        applicationInfo: ApplicationInfo,
        hostApplicationInfo: ApplicationInfo,
        pluginApkPath: String,
        loadParameters: LoadParameters,
        pluginPartsMap: MutableMap<String, PluginParts>
    ) {
        applicationInfo.publicSourceDir = pluginApkPath
        applicationInfo.sourceDir = pluginApkPath
        val hostSharedLibraryFiles = hostApplicationInfo.sharedLibraryFiles
        val paths = arrayListOf<String>()
        val dependsOn = loadParameters.dependsOn
        if (dependsOn != null && dependsOn.isNotEmpty()) {
            dependsOn.forEach {
                pluginPartsMap[it]?.apply {
                    paths.add(pluginPackageManager.archiveFilePath)
                }
            }
        }
        val otherApksAddToResources = if (hostSharedLibraryFiles == null) {
            arrayOf(*paths.toTypedArray())
        } else {
            arrayOf(
                *paths.toTypedArray(),
                *hostSharedLibraryFiles
            )
        }
        applicationInfo.sharedLibraryFiles = otherApksAddToResources
    }

}
```
改动其实不多，不过我测试下来，假如插件A依赖common插件，appcompat在common插件里面，有webview的Activity不能是AppCompatActivity。