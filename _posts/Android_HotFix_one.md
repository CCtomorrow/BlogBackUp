---
title: '热修复探究（一）'
date: 2016-07-27 23:15:40
tags: [HotFix]
categories: [Android,HotFix]
---

### 前言
这次博客会分两篇，这篇介绍各个Android版本是怎么反射加载生成的patch文件的，下篇会详细的分析class对比和patch的生成。

写这次文章的原因是因为最近在研究热修复，发现其实他们实现的代码很少，其实就一个类，然后里面针对不同的版本做反射处理，就想好好找找不同版本的对于类加载的机制。
其次呢，关于bug版本和修复版本的class文件对比，dex的patch文件生成的脚本也想了解一下。

### Android ClassLoader区别
首先Android中加载类一般使用的是`PathClassLoader`和`DexClassLoader`。
区别:
`PathClassLoader`
* Android uses this class for its system class loader and for its application class
    loader(s).

可以看出，Android是使用这个类作为其系统类和应用类的加载器。并且对于这个类呢，只能去加载已经安装到Android系统中的apk文件。

`DexClassLoader`
* A class loader that loads classes from {@code .jar} and {@code .apk} files
    containing a {@code classes.dex} entry. This can be used to execute code not
    installed as part of an application.

可以看出，该类可以用来从.jar和.apk类型的文件内部加载classes.dex文件。可以用来执行非安装的程序代码。

`DexClassLoader`
构造函数：DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent)

* dexPath:被解压的dex路径，不能为空。
* optimizedDirectory：解压后的.dex文件的存储路径，不能为空。这个路径强烈建议使用应用程序的私有路径，不要放到sdcard上，否则代码容易被注入攻击。
* libraryPath：so库的存放路径，可以为空，若有so库，必须填写。
* parent：父亲加载器，一般为context.getClassLoader(),使用当前上下文的类加载器。

<!-- more -->

### 不同版本的不同实现
我们会发现，根本找不到源代码，那其实代码在android_libcore里面。
[https://github.com/Evervolv/android_libcore](https://github.com/Evervolv/android_libcore)
这个库里面可以找到不同版本的lib_core的实现。

#### Gingerbread 2.3 9
首先看**Gingerbread**的实现，在2.3的时候PathClassLoader和DexClassLoader是分别实现的。着重看findClass方法。
PathClassLoader
```java
private final String path;

private final String[] mPaths;
private final File[] mFiles;
private final ZipFile[] mZips;
private final DexFile[] mDexs;

@Override  protected Class<?> findClass(String name) throws ClassNotFoundException  {
    byte[] data = null;
    int length = mPaths.length;
    for (int i = 0; i < length; i++) {
      if (mDexs[i] != null) {
      Class clazz = mDexs[i].loadClassBinaryName(name, this);
      if (clazz != null)
        return clazz;
      } else if (mZips[i] != null) {
      String fileName = name.replace('.', '/') + ".class";
      data = loadFromArchive(mZips[i], fileName);
      } else {
      File pathFile = mFiles[i];
      if (pathFile.isDirectory()) {
        String fileName =  mPaths[i] + "/" +
        name.replace('.', '/') + ".class";
        data = loadFromDirectory(fileName);
        }
      }
    }
throw new ClassNotFoundException(name + " in loader " + this);
}
```
* The entries of the second list should be directories containing
* native library files. Both lists are separated using the
* character specified by the "path.separator" system property,
* which, on Android, defaults to ":".
里面有这段内容，就是说，dex文件路径存储在path里面，以分隔符分开，在Android里面分隔符是":"。

这里其实只能对mDexs[]处理，其余的zip，files并不能处理，后面有注释说明，详细的可以看源代码。

DexClassLoader
```java
private final File[] mFiles; // source file Files, for rsrc URLs
private final ZipFile[] mZips; // source zip files, with resources
private final DexFile[] mDexs; // opened, prepped DEX files

protected Class<?> findClass(String name) throws ClassNotFoundException {

    int length = mFiles.length;
    for (int i = 0; i < length; i++) {
        if (mDexs[i] != null) {
            String slashName = name.replace('.', '/');
            Class clazz = mDexs[i].loadClass(slashName, this);
            if (clazz != null) {
                if (VERBOSE_DEBUG)
                    System.out.println("    found");
                return clazz;
            }
        }
    }
    throw new ClassNotFoundException(name + " in loader " + this);
}
```
可以看到也只是可以从加载进来的dex文件里面找Class。

所以，热修复对他的反射处理如下:
```java
/**
 * Installer for platform versions 4 to 13.
 */
 private static final class V4 {

    private static void install(ClassLoader loader, List<File> additionalClassPathEntries)
            throws IllegalArgumentException, IllegalAccessException,
            NoSuchFieldException, IOException {
        /* The patched class loader is expected to be a descendant
         * of dalvik.system.DexClassLoader. We modify its fields mPaths,
         * mFiles, mZips and mDexs to append additional DEX file entries.
        */
        int extraSize = additionalClassPathEntries.size();
        Field pathField = findField(loader, "path");
        // 旧的path
  StringBuilder path = new StringBuilder((String) pathField.get(loader));
        String[] extraPaths = new String[extraSize];
        File[] extraFiles = new File[extraSize];
        ZipFile[] extraZips = new ZipFile[extraSize];
        DexFile[] extraDexs = new DexFile[extraSize];
        for (ListIterator<File> iterator = additionalClassPathEntries.listIterator();
             iterator.hasNext(); ) {
            File additionalEntry = iterator.next();

            // 添加新的dex文件路径到path里面
  String entryPath = additionalEntry.getAbsolutePath();
            path.append(':').append(entryPath);

            int index = iterator.previousIndex();
            extraPaths[index] = entryPath; // paths
  extraFiles[index] = additionalEntry; // files
  extraZips[index] = new ZipFile(additionalEntry); // zipfiles
  extraDexs[index] = DexFile.loadDex(entryPath, entryPath + ".dex", 0); // dexfiles
  }
        // 重新设置path
  pathField.set(loader, path.toString());
        // 重新设置mPaths，mFiles，mZips，mDexs
  expandFieldArray(loader, "mPaths", extraPaths);
        expandFieldArray(loader, "mFiles", extraFiles);
        expandFieldArray(loader, "mZips", extraZips);
        expandFieldArray(loader, "mDexs", extraDexs);
    }
}
```
实现即是这样的，其实很简单啦，就是把patch的dex文件路径加到path里面，再使用反射修改掉里面的四个字段，重新赋值。

#### Ice 4.0 14
然后**Ice**的实现，看在4.0的具体实现。在4.0以后DexClassLoader和PathClassLoader都继承了BaseDexClassLoader，处理的代码都在BaseDexClassLoader里面。
BaseDexClassLoader
```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    Class clazz = pathList.findClass(name);
    if (clazz == null) {
        throw new ClassNotFoundException(name);
    }
    return clazz;
}
```
代码很简洁，可以看到findClass是使用的DexPathList的实例，pathList去找到对应的class。
DexPathList里面的findClass
```java
public Class findClass(String name) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    return null;
}
```
从dexElements里面寻找，所以需要使用反射修改掉dexElements，即dex数组文件。
相关实现如下:
```java
/**
 * Installer for platform versions 14, 15, 16, 17 and 18.
 */
 private static final class V14 {
    private static void install(ClassLoader loader, List<File> additionalClassPathEntries, File optimizedDirectory)
            throws IllegalArgumentException, IllegalAccessException,
            NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
        /* The patched class loader is expected to be a descendant
        of dalvik.system.BaseDexClassLoader. We modify its
        dalvik.system.DexPathList pathList field to append additional
        DEX file entries.
        */
        Field pathListField = findField(loader, "pathList");
        Object dexPathList = pathListField.get(loader);
        expandFieldArray(dexPathList, "dexElements",
        makeDexElements(dexPathList,
           new ArrayList<File>(additionalClassPathEntries),
                optimizedDirectory));
    }

/**
 * A wrapper around {@code private static final
 dalvik.system.DexPathList#makeDexElements}.
 */
 private static Object[] makeDexElements(
        Object dexPathList, ArrayList<File> files, File
        optimizedDirectory)
            throws IllegalAccessException, InvocationTargetException,
            NoSuchMethodException {
        Method makeDexElements =
                findMethod(dexPathList, "makeDexElements",
                ArrayList.class, File.class);
        return (Object[]) makeDexElements.invoke
        (dexPathList, files, optimizedDirectory);
    }
}
```
实现也挺简单，直接把我们要加的dex文件添加dexElements数组的最前面即可。

#### kitkat 4.4 19
BaseDexClassLoader
```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        throw cnfe;
    }
    return c;
}
```
DexPathList
```java
public Class findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
          Class clazz = dex.loadClassBinaryName(name, definingContext,
            suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    if (dexElementsSuppressedExceptions != null) {
       suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```
可以看到只是新增了一些异常，合成dex文件的时候需要传递多一个参数用来存储找寻每个dex文件发生异常IO信息。
对应的处理:
```java
/**
 * Installer for platform versions 19.
 */
 private static final class V19 {

    private static void install(ClassLoader loader, List<File>
    additionalClassPathEntries, File optimizedDirectory)
            throws IllegalArgumentException, IllegalAccessException,
            NoSuchFieldException, InvocationTargetException,
            NoSuchMethodException {
        /* The patched class loader is expected to be a descendant of
         * dalvik.system.BaseDexClassLoader. We modify its
         * dalvik.system.DexPathList pathList field to append additional
         * DEX file entries.
         */
        Field pathListField = findField(loader, "pathList");
        Object dexPathList = pathListField.get(loader);
        ArrayList<IOException> suppressedExceptions = new
        ArrayList<IOException>();
        expandFieldArray(dexPathList, "dexElements",
        makeDexElements(dexPathList,
        new ArrayList<File>(additionalClassPathEntries),
        optimizedDirectory, suppressedExceptions));
        if (suppressedExceptions.size() > 0) {
            for (IOException e : suppressedExceptions) {
                Log.w(TAG, "Exception in makeDexElement", e);
            }
            Field suppressedExceptionsField =
                    findField(dexPathList, "dexElementsSuppressedExceptions");
            IOException[] dexElementsSuppressedExceptions =
                    (IOException[]) suppressedExceptionsField.get(dexPathList);

            if (dexElementsSuppressedExceptions == null) {
                dexElementsSuppressedExceptions =
                        suppressedExceptions.toArray(
                                new IOException[suppressedExceptions.size()]);
            } else {
                IOException[] combined =
                        new IOException[suppressedExceptions.size() +
                                dexElementsSuppressedExceptions.length];
                suppressedExceptions.toArray(combined);
                System.arraycopy(dexElementsSuppressedExceptions, 0, combined,
                        suppressedExceptions.size(), dexElementsSuppressedExceptions.length);
                dexElementsSuppressedExceptions = combined;
            }

            suppressedExceptionsField.set(dexPathList, dexElementsSuppressedExceptions);
        }
    }

    /**
     * A wrapper around
     * {@code private static final
     dalvik.system.DexPathList#makeDexElements}.
     */
     private static Object[] makeDexElements(
            Object dexPathList, ArrayList<File> files, File optimizedDirectory,
            ArrayList<IOException> suppressedExceptions)
            throws IllegalAccessException, InvocationTargetException,
            NoSuchMethodException {
        Method makeDexElements =
                findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class,
                        ArrayList.class);

        return (Object[]) makeDexElements.invoke(dexPathList, files, optimizedDirectory,
                suppressedExceptions);
    }
}
```
makeDexElements方法作了反射调用，并且如果找寻我们自己的新增dex出了问题也会去把做异常处理，最终会对dexElementsSuppressedExceptions这个异常数组做处理。
5.X的这部分代码并没有什么差别。

关于6.0和7.0的，这里并没有代码，现在就不分析，找到代码会继续分析。其实思路很简单，一个版本一个版本的去对比findClass的实现的差别，然后调整反射调用的代码。

具体可以看Github上面的两个开源库:
[https://github.com/bunnyblue/DroidFix](https://github.com/bunnyblue/DroidFix)

[https://github.com/dodola/RocooFix](https://github.com/dodola/RocooFix)

这里的代码都是取自那里，这篇文章分析热修复加载dex的部分，下篇会分析，CLASS_ISPREVERIFIED的实现，即class对比和patch的生成。

关于6.X的，有看到源代码，发现和5.X的基本一样，不明白作者为什么在V19和V23使用了两种不同的反射方式去实现，可能是在实践中踩出来的经验吧。