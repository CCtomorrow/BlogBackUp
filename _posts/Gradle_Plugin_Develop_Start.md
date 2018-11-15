---
title: 'Android Gradle插件开发介绍'
date: 2018-09-23 23:43:40
tags: [Gradle]
categories: [Android,Gradle]
---
### 前言
其实很早就想聊一聊Android Gradle插件的开发了，这次终于开始啦。Android Gradle插件是Google开发的一个用于编译打包安卓apk的插件，其实也就是一个gradle插件而已，我们这里讲的Android Gradle插件的开发是基于gradle以及Android Gradle(例如`com.android.application`)插件的再次开发，开发的方式基本都是在Android Gradle的task之前、之间或者之后插入一些我们的自定义的task等等，主要是为了实现我们想要的一些功能。

### 基础与参考
本文章不讲基础，麻烦自己去看对应的文章，我这里会列出来我认为比较好的开发Gradle插件的资料。**当然这些基础的前提也是要对Gradle有所了解**
[Writing Custom Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html#custom_plugins)
[Plugin Development Guides](https://gradle.org/guides/?q=Plugin%20Development)
[操作Task](https://chaosleong.gitbooks.io/gradle-for-android/content/advanced_build_customization/manipulating_tasks.html)
[断点调试Gradle插件](http://fucknmb.com/2017/07/05/%E5%8F%88%E6%8E%8C%E6%8F%A1%E4%BA%86%E4%B8%80%E9%A1%B9%E6%96%B0%E6%8A%80%E8%83%BD-%E6%96%AD%E7%82%B9%E8%B0%83%E8%AF%95Gradle%E6%8F%92%E4%BB%B6/)
[Gradle插件-提高篇](https://linxiaotao.github.io/2018/05/21/Gradle%E6%8F%92%E4%BB%B6-%E6%8F%90%E9%AB%98%E7%AF%87/)
[写给Android开发者的Gradle系列三](https://juejin.im/post/5b02113a5188254289190671)

<!-- more -->

### 在你的插件里面应用别人的插件
有时候我们会想，我在项目里面又需要应用google的安卓插件，又需要应用自己开发的插件，那么我们可以不可以直接使用我们自己的插件，然后在我们自己的插件里面应用google的安卓插件呢，当然是可以的了。
如下就可以搞定了，是不是很简单
```groovy
project.apply plugin: AppPlugin
//project.apply plugin: 'com.android.application'//换成这个也可以，因为他们代表的是同一个东西，如果理解不了，请看上面的基础文章
```
不过这种方式有一个缺点，由于gradle是边读取边解释的，所以如果我们想在`void apply(Project project)`一执行就读取android的配置，那么我们在项目的build.gradle里面应用我们的插件的位置就要放在`android {}`代码块之后，像下面这样。
![apply位置](/images/gradle_plugin_order.png)
如果这样的话，就不能在我们的插件里面apply安卓gradle插件啦。因为安卓gradle插件的apply语句要放在`android {}`代码块之前的，因为extension的定义要在apply插件之后的。

### Android项目的Variant
Android 项目中会有大量相同的 task，并且它们的名字基于 Build Types 和 Product Flavor 生成。
为了解决这个问题，android 对象有三个属性(后面还增加了featureVariants，暂时不讨论)：
applicationVariants（只适用于 app plugin）
libraryVariants（只适用于 library plugin）
testVariants（app、library plugin 均适用）
这三个属性会分别返回一个 ApplicationVariant、LibraryVariant 和 TestVariant 对象的 DomainObjectCollection (集合，因为有不同的情况组合)。

注意，使用这三个 collection 中的其中一个都会触发生成所有对应的 task。这意味着使用 collection 之后不需要重新配置。
DomainObjectCollection 可以直接访问所有对象，或者通过过滤器进行筛选。
```
android.applicationVariants.each { variant ->
    ....
}
```
这三个 variant 类都拥有下面的属性：

| 属性名 | 属性类型 | 说明 |
| --- | --- | --- |
|name|String|Variant 的名字，唯一|
|description|String|Variant 的描述说明|
|dirName|String|Variant 的子文件夹名，唯一。可能有不止一个子文件夹，例如 “debug/flavor2”|
|baseName|String|Variant 输出的基础名字，必须唯一|
|outputFile|File|Variant 的输出，该属性可读可写|
|processManifest|ProcessManifest|处理 Manifest 的 task|
|aidlCompile|AidlCompile|编译 AIDL 文件的 task|
|renderscriptCompile|RenderscriptCompile|编译 Renderscript 文件的 task|
|mergeResources|MergeResources|合并资源文件的 task|
|mergeAssets|MergeAssets|合并 assets 的 task|
|processResources|ProcessAndroidResources|处理并编译资源文件的 task|
|generateBuildConfig|GenerateBuildConfig|生成 BuildConfig 类的 task|
|javaCompile|JavaCompile|编译 Java 源代码的 task|
|processJavaResources|Copy|处理 Java 资源的 task|
|assemble|DefaultTask|Variant 的标志性 assemble task|

ApplicationVariant 类还有以下附加属性：

|属性名|属性类型|说明|
|--|--|--|
|buildType|BuildType|Variant 的 BuildType|
|productFlavors|List<ProductFlavor>|Variant 的 ProductFlavor，一般不为空但允许为空|
|mergedFlavor|ProductFlavor|android.defaultConfig 和 variant.productFlavors 的组合|
|signingConfig|SigningConfig|Variant 使用的 SigningConfig 对象|
|isSigningReady|boolean|如果是 true 则表明该 Variant 已经具备了所有需要签名的信息|
|testVariant|BuildVariant|将会测试该 Variant 的 TestVariant|
|dex|Dex|将代码打包成 dex 的 task。如果该 Variant 是 Library，该值可为空|
|packageApplication|PackageApplication|打包最终 APK 的 task。如果该 Variant 是 Library，该值可为空|
|zipAlign|ZipAlign|对 APK 进行 zipalign 的 task。如果该 Variant 是 Library 或者 APK 不能被签名时，该值可为空|
|install|DefaultTask|负责安装的 task，可为空|
|uninstall|DefaultTask|负责卸载的 task|

LibraryVariant 类还有以下附加属性：

|属性名|属性类型|说明|
|--|--|--|
|buildType|BuildType|Variant 的 BuildType|
|mergedFlavor|ProductFlavor|只有 android.defaultConfig|
|testVariant|BuildVariant|用于测试 Variant|
|packageLibrary|Zip|用于打包 Library 项目的 AAR 文件。如果是 Library 项目，该值不能为空|

TestVariant 类还有以下属性：

|属性名|属性类型|说明|
|--|--|--|
|buildType|BuildType|Variant 的 Build Type|
|productFlavors|List<ProductFlavor>|Variant 的 ProductFlavor。一般不为空但允许为空|
|mergedFlavor|ProductFlavor|android.defaultConfig 和 variant.productFlavors 的组合|
|signingConfig|SigningConfig|Variant 使用的 SigningConfig 对象|
|isSigningReady|boolean|如果是 true 则表明该 Variant 已经具备了所有需要签名的信息|
|testedVariant|BaseVariant|TestVariant 测试的 BaseVariant|
|dex|Dex|将代码打包成 dex 的 task。如果该 Variant 是 Library，该值可为空|
|packageApplication|PackageApplication|打包最终 APK 的 task。如果该 Variant 是 Library，该值可为空|
|zipAlign|ZipAlign|对 APK 进行 zipalign 的 task。如果该 Variant 是 Library 或者 APK 不能被签名时，该值可为空|
|install|DefaultTask|负责安装的 task，可为空|
|uninstall|DefaultTask|负责卸载的 task|
|connectedAndroidTest|DefaultTask|在连接设备上执行 Android 测试的 task|
|providerAndroidTest|DefaultTask|使用扩展 API 执行 Android 测试的 task|

Android task 特有类型的 API：
- ProcessManifest
 File manifestOutputFile
- AidlCompile
 File sourceOutputDir
- RenderscriptCompile
 File sourceOutputDir
 File resOutputDir
- MergeResources
 File outputDir
- MergeAssets
 File outputDir
- ProcessAndroidResources
 File manifestFile
 File resDir
 File assetsDir
 File sourceOutputDir
 File textSymbolOutputDir
 File packageOutputFile
 File proguardOutputFile
- GenerateBuildConfig
 File sourceOutputDir
- Dex
 File outputFolder
- PackageApplication
 File resourceFile
 File dexFile
 File javaResourceDir
 File jniDir
 File outputFile
  直接在 Variant 对象中使用 “outputFile” 可以改变最终的输出文件夹。
- ZipAlign
 File inputFile
 File outputFile
  直接在 Variant 对象中使用 “outputFile” 可以改变最终的输出文件夹。

每个 task 类型的 API 都受 Gradle 的工作方式和 Android plugin 的配置方式限制。
首先，Gradle 中存在的 task 只能配置输入输出的路径以及部分可能使用的可选标识。因此，这些 task 只能定义一些输入或者输出。

其次，这里面大多数 task 的输入都不是单一的，一般都混合了 sourceSet、Build Type 和 Product Flavor 中的值。所以为了保持构建文件的简洁和可读性，目标自然是让开发者通过 DSL 修改这些对象来影响构建的过程，而不是深入修改输入和 task 的选项。

另外需要注意，上面的 task 中除了 ZipAlign，其它都要求设置私有数据进行工作。这意味着不能手动创建这些类型的 task 实例。

这些 API 也可能会被修改。一般来说，目前的 API 是围绕着 task 的输出和输入（如果可能）来添加额外的处理（必要时）。欢迎反馈意见，特别是那些没有考虑过的需求。

### 自定义插件修改bugly热修复的tinkerId配置
了解bugly的热修复的人，知道他是利用的tinker，然后bugly写了自己的gradle插件，当然这个插件里面依赖了tinker的gradle插件。
用热修复就涉及到多渠道的问题，一般不管多少渠道，我们的代码都是同一套的，意思就是说我们可以使用同一个补丁包，bugly博客也专门写了一篇文章来讲解怎么去做。
[Bugly多渠道热更新解决方案](https://buglydevteam.github.io/2017/05/15/solution-of-multiple-channel-hotpatch/)
那我们公司比较特殊，开发人员只会给运营同事打几个不同名称的apk包给运营，然后由运营利用我们开发的一个打包工具，加上他们想要的渠道名称，进行二次打包，那这样的话，就会有一些问题了，打不同名称的apk需要用到多渠道打包，这样再加上使用bugly插件那么最后生成的apk包每个渠道的tinkerId都不一样，但是我们只是apk名称改了，其他代码资源没有任何操作，这样的话其实可以使用同一个补丁包的。
这种情况就想到自己再次开发一个gradle插件，在bugly插件对应的task后面添加task，然后重新生成tinkerId。
```groovy
public class LinghitBuglyManifestPlugin implements Plugin<Project> {

    @Override
    void apply(Project project) {
        //不是Application项目不执行
        if (!project.plugins.hasPlugin(AppPlugin)) {
            return
        }
        //应用bugly的插件，必须要先应用，Gradle是边读取边解释，所以下面的LinghitHotFixExtension只能在项目评估完成之后
        //也就意味着LinghitHotFixExtension.enable只能控制自身，可以有另一种方案，不以extension的形式，二是把参数放在
        //gradle.properties里面，这样enable就可以控制整个插件包括自身，tinkerSupport,tinker
        //但是由于tinkerSupport,tinker的project.afterEvaluate是在当前插件的project.afterEvaluate之前执行的，所以
        //tinkerId的设置要在当前插件的project.afterEvaluate之前
        project.apply plugin: TinkerSupportPlugin
        project.extensions.create("LHotFix", LinghitHotFixExtension)
        AppExtension android = project.extensions.getByType(AppExtension.class)
        LinghitHotFixExtension fixExtension = project.extensions.LHotFix
        //参数
        def format = new SimpleDateFormat("yyyy-MMdd-HHmm")
        def id = "APP-${android.defaultConfig.versionName}-${android.defaultConfig.versionCode}-${format.format(new Date())}"
        def bakPath = "${project.getProjectDir()}/bakApk/"
        def patchOutDir = "${project.getProjectDir()}/patch/"
        def extension = project.extensions.getByType(TinkerSupportExtension)
        extension.enable = true
        extension.autoGenerateTinkerId = false
        //上面设置为true，这个就不起作用了
        extension.tinkerId = id
        extension.autoBackupApkDir = bakPath
        extension.patchOutputDir = patchOutDir
        extension.enableProxyApplication = true
        extension.supportHotplugComponent = true
        extension.overrideTinkerPatchConfiguration = true
        project.afterEvaluate {
            //不需要热修复的话，直接return
            if (!fixExtension.enable) {
                extension.enable = false
                project.extensions.tinkerPatch.tinkerEnable = false
                return
            }
            extension.baseApk = "${bakPath}${fixExtension.oldApk}/app-release.apk"
            extension.baseApkProguardMapping = "${bakPath}${fixExtension.oldApk}/app-release-mapping.txt"
            extension.baseApkResourceMapping = "${bakPath}${fixExtension.oldApk}/app-release-R.txt"
            extension.buildAllFlavorsDir = "${bakPath}/${fixExtension.oldApk}"
            extension.isProtectedApp = fixExtension.isProtectedApp

            project.getLogger().error("tinker old apk:${project.extensions.tinkerPatch.oldApk}")

            project.getLogger().error("buglyTinker old apk:${project.extensions.tinkerSupport.baseApk}")

//        String tinkerValue = project.extensions.tinkerPatch.buildConfig.tinkerId
//        if (tinkerValue == null || tinkerValue.isEmpty()) {
//            project.extensions.tinkerPatch.buildConfig.tinkerId = extension.tinkerId
//        }

           //这个操作不行
//        TinkerPatchExtension patchExtension = project.extensions.getByType(TinkerPatchExtension)
//        TinkerBuildConfigExtension configExtension = project.extensions.tinkerPatch.buildConfig
//        configExtension.tinkerId = extension.tinkerId

            android.applicationVariants.all { ApplicationVariantImpl variant ->
                String variantName = variant.name.capitalize()
                ApkVariantData variantData = variant.variantData
                ProductFlavor flavor = variant.mergedFlavor
                BaseVariantOutput output = variant.outputs.first()
                ManifestProcessorTask mpTask = output.processManifest
                ProcessAndroidResources rpTask = output.processResources
                TinkerSupportManifestTask tmpTask = project.tasks.getByName("tinkerSupportProcess${variantName}Manifest")
                LinghitManifestTask lmpTask = project.tasks.create("linghitProcess${variantName}Manifest", LinghitManifestTask)
                if (mpTask.properties['manifestOutputFile'] != null) {
                    lmpTask.manifestPath = mpTask.manifestOutputFile
                } else if (rpTask.properties['manifestFile'] != null) {
                    lmpTask.manifestPath = rpTask.manifestFile
                }
                lmpTask.finalTinkerId = extension.tinkerId
                lmpTask.mustRunAfter(tmpTask)
                rpTask.dependsOn lmpTask
                //project.getLogger().error("variantName:${variantName}")
                project.getLogger().error("灵机bugly插件要处理的manifestPath:${lmpTask.manifestPath}")
            }
        }
    }
}
```

```groovy
public class LinghitManifestTask extends DefaultTask {

    public static
    final String MANIFEST_XML = "build/intermediates/linghit_bugly_intermediates/AndroidManifest.xml"
    private static
    final String TINKER_ID = "TINKER_ID"

    String manifestPath
    String finalTinkerId

    public LinghitManifestTask() {
        group = 'l-tinker-support'
    }

    @TaskAction
    def updateManifest() {
        def ns = new Namespace("http://schemas.android.com/apk/res/android", "android")
        def isr = null
        def pw = null
        try {
            isr = new InputStreamReader(new FileInputStream(manifestPath), "utf-8")
            def xml = new XmlParser().parse(isr)
            getLogger().error("manifestPath：${manifestPath}")
            //def application = xml.application[0]
            def application = xml.application[0]
            def oldId
            if (application) {
                def metaDataTags = application['meta-data']
                // find TINKER_ID elements
                metaDataTags.findAll {
                    it.attributes()[ns.name].equals(TINKER_ID)
                }.each {
                    //<application>
                    //    <meta-data
                    //        android:name="TINKER_ID"
                    //        android:value="App_oppo-release-2.1.2.212_0922-23-11-02" />
                    //</application>
                    oldId = it.attributes()[ns.value]
                    it.parent().remove(it)
                }
                getLogger().error("Remove ${TINKER_ID}：${oldId}")

                //Add the new TINKER_ID element
                application.appendNode('meta-data', [(ns.name): TINKER_ID, (ns.value): finalTinkerId])

                //APP-1.0.0-100-2018-0923-1513
                getLogger().error("Add New ${TINKER_ID}：${finalTinkerId}")

                // Write the manifest file
                pw = new PrintWriter(manifestPath, "utf-8")
                def printer = new XmlNodePrinter(pw)
                printer.preserveWhitespace = true
                printer.print(xml)
            }
        } finally {
            StreamUtil.closeQuietly(pw)
            StreamUtil.closeQuietly(isr)
        }

        File manifestFile = new File(manifestPath)
        if (manifestFile.exists()) {
            FileOperation.copyFileUsingStream(manifestFile, project.file(MANIFEST_XML))
            project.logger.error("linghit bugly support gen AndroidManifest.xml in ${MANIFEST_XML}")
        }
    }

}
```
