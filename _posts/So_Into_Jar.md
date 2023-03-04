---
title: 'Android将SO库封装到JAR包中并加载其中的SO库'
date: 2023-03-04 20:47:00
tags: [ndk]
categories: [Android,ndk]
---

#### 说明
因为一些原因，我们提供给客户的sdk，只能是jar包形式的，一些情况下，sdk里面有native库的时候，就不太方便操作了，此篇文章主要解决如何把so库放入jar包里面，如何打包成jar，以及如何加载。

#### 1.如何把so库放入jar包
so库放入jar参考此文章[ANDROID将SO库封装到JAR包中并加载其中的SO库](https://www.freesion.com/article/8348550727/)
![放置路径](/images/97801677933126_.pic.jpg)
将so库改成.jet后缀，放置和加载so库的SoLoader类同一个目录下面。

<!-- more -->

#### 2.如何使用groovy打包jar
![打包jar](/images/97841677933397_.pic.jpg)
先把需要打包的class放置到同一个文件夹下面，然后打包即可，利用groovy的copy task完成这项工作非常简单。

#### 3.如何加载jar包里面的so
##### 3.1.首先判断当前jar里面是否存在so
```java
InputStream inputStream = SoLoader.class.getResourceAsStream("/com/dianping/logan/arm64-v8a/liblogan.jet");
```
如果inputStream不为空就表示存在。

##### 3.2.拷贝
判断是否已经把so库拷贝到手机里面了，如果没有拷贝过就进行拷贝，这个代码逻辑很简单。
```java
public class SoLoader {
    private static final String TAG = "SoLoader";

    /**
     * so库释放位置
     */
    public static String getPath() {
        String path = GlobalCtx.getApp().getFilesDir().getAbsolutePath();
        //String path = GlobalCtx.getApp().getExternalFilesDir(null).getAbsolutePath();
        return path;
    }

    public static String get64SoFilePath() {
        String path = SoLoader.getPath();
        String v8a = path + File.separator + "jniLibs" + File.separator +
                "arm64-v8a" + File.separator + "liblogan.so";
        return v8a;
    }

    public static String get32SoFilePath() {
        String path = SoLoader.getPath();
        String v7a = path + File.separator + "jniLibs" + File.separator +
                "armeabi-v7a" + File.separator + "liblogan.so";
        return v7a;
    }

    /**
     * 支持两种模式，如果InputStream inputStream = SoLoader.class.getResourceAsStream("/com/dianping/logan/arm64-v8a/liblogan.jet");
     * 返回了空，表示可能此库是aar接入的，普通加载so库就行，不为空，需要拷贝so库，动态加载
     */
    public static boolean jarMode() {
        boolean jarMode = false;
        InputStream inputStream = SoLoader.class.getResourceAsStream("/com/dianping/logan/arm64-v8a/liblogan.jet");
        if (inputStream != null) {
            jarMode = true;
            try {
                inputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return jarMode;
    }

    /**
     * 是否已经拷贝过so了
     */
    public static boolean alreadyCopySo() {
        String v8a = SoLoader.get64SoFilePath();
        File file = new File(v8a);
        if (file.exists()) {
            String v7a = SoLoader.get32SoFilePath();
            file = new File(v7a);
            return file.exists();
        }
        return false;
    }

    /**
     * 拷贝logan的so库
     */
    public static boolean copyLoganJni() {
        boolean load;
        File dir = new File(getPath(), "jniLibs");
        if (!dir.exists()) {
            load = dir.mkdirs();
            if (!load) {
                return false;
            }
        }
        File subdir = new File(dir, "arm64-v8a");
        if (!subdir.exists()) {
            load = subdir.mkdirs();
            if (!load) {
                return false;
            }
        }
        File dest = new File(subdir, "liblogan.so");
        //load = copySo("/lib/arm64-v8a/liblogan.so", dest);
        load = copySo("/com/dianping/logan/arm64-v8a/liblogan.jet", dest);
        if (load) {
            subdir = new File(dir, "armeabi-v7a");
            if (!subdir.exists()) {
                load = subdir.mkdirs();
                if (!load) {
                    return false;
                }
            }
            dest = new File(subdir, "liblogan.so");
            //load = copySo("/lib/armeabi-v7a/liblogan.so", dest);
            load = copySo("/com/dianping/logan/armeabi-v7a/liblogan.jet", dest);
        }
        return load;
    }

    public static boolean copySo(String name, File dest) {
        InputStream inputStream = SoLoader.class.getResourceAsStream(name);
        if (inputStream == null) {
            Log.e(TAG, "inputStream == null");
            return false;
        }
        boolean result = false;
        FileOutputStream outputStream = null;
        try {
            outputStream = new FileOutputStream(dest);
            int i;
            byte[] buf = new byte[1024 * 4];
            while ((i = inputStream.read(buf)) != -1) {
                outputStream.write(buf, 0, i);
            }
            result = true;
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                inputStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            if (outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return result;
    }

}
```

##### 3.3.加载
首先判断当前应用是32位还是64位`Process.is64Bit();`。然后加载对应的32或者64位的so。
```java
static {
    try {
        if (SoLoader.jarMode()) {
            if (SoLoader.alreadyCopySo()) {
                sIsCloganOk = loadLocalSo();
            } else {
                boolean copyLoganJni = SoLoader.copyLoganJni();
                if (copyLoganJni) {
                    sIsCloganOk = loadLocalSo();
                }
            }
        } else {
            System.loadLibrary(LIBRARY_NAME);
            sIsCloganOk = true;
        }
    } catch (Throwable e) {
        e.printStackTrace();
        sIsCloganOk = false;
    }
}

static boolean loadLocalSo() {
    boolean bit = Process.is64Bit();
    if (bit) {
        String v8a = SoLoader.get64SoFilePath();
        try {
            System.load(v8a);
            return true;
        } catch (Throwable e) {
            e.printStackTrace();
            return false;
        }
    } else {
        String v7a = SoLoader.get32SoFilePath();
        try {
            System.load(v7a);
            return true;
        } catch (Throwable e) {
            e.printStackTrace();
            return false;
        }
    }
}
```
