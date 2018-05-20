---
title: 'Android中图片到底占多大空间'
date: 2018-05-20 16:30:00
tags: [Custom View]
categories: [Android,Custom View]
---
### 屏幕尺寸，屏幕分辨率，屏幕像素密度是什么?
屏幕尺寸是指屏幕对角线长度，单位是英寸，常见尺寸，5.1，5.5，5.8寸。

屏幕分辨率是指在横纵向上的像素点数，单位是px，1px=一个像素点，一般纵向像素x横向像素，如分辨率1920x1080。

屏幕像素密度是指每英寸上的像素点数，单位dpi，即dot per inch的意思，屏幕像素密度与屏幕尺寸和屏幕分辨率有关，在单一变化条件下，屏幕尺寸越小、分辨率越高，像素密度越大，反之越小。

density是个可根据屏幕面板素质而设定的常数。简单来说，可以理解为 density 的数值是 1dp=density px。

|desity|1|1.5|2|3|3.5|4|
|-|-|-|-|-|-|-|
|desityDpi|160|240|320|480|560|640|

drawable文件对应的desityDpi

|drawable|ldpi|mdpi|hdpi|xhdpi|xxhdpi|xxxhdpi|
|-|-|-|-|-|-|-|
|desityDpi|120|160|240|320|480|640|

<!-- more -->

![PPI计算公式](/images/ppi_cal.png)

ppi是实际的dpi，是实际计算出来的，公式如下，然后根据这个值，在上图中找到最接近的dpi。
Nexus5 4.95inch
1920*1080
DPI = 445
1920^2+1080^2开根号/4.95 = 445，所以Nexus5的desity为3

普通情况下，一张10241024的图，采用ARGB_8888(4字节，一个字节8位，所以叫8888)，大小为10241024*4byte，即4M。

### 安卓图片读取情况
#### 1.读取drawable下面的图片
```java
public static Bitmap decodeResourceStream(Resources res, TypedValue value,
        InputStream is, Rect pad, Options opts) {
    //实际上，我们这里的opts是null的，所以在这里初始化。
    if (opts == null) {
        opts = new Options();
    }
    if (opts.inDensity == 0 && value != null) {
        final int density = value.density;
        if (density == TypedValue.DENSITY_DEFAULT) {
            opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
        } else if (density != TypedValue.DENSITY_NONE) {
            //这里density的值如果对应资源目录为xhdpi的话，就是320
            opts.inDensity = density;
        }
    }
    if (opts.inTargetDensity == 0 && res != null) {
        //请注意，inTargetDensity就是当前的显示密度，比如三星Nexus5就是445
        opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
    }
    return decodeStream(is, pad, opts);
}
```
最终一张10241024的ARGB_8888图片在Nexus5上面放在xhdpi下面 
(inTargetDensity / inDensity) (width * height) * 4=(445/320) * (1024 * 1024) * 4=5.6M

实际上bitmap.getByteCount()不是这样算的，有一些差别导致，它是按照下面的算的。
(inTargetDensity / inDensity * width +0.5) * (inTargetDensity / inDensity * height +0.5) * 4 
这样的话其实计算出的数据和上面的差别挺大的。

我们也可以手动控制--->
```java
BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = false;
options.inSampleSize = 1;
options.inDensity = 160;
options.inTargetDensity = 160;
bitmap = BitmapFactory.decodeResource(getResources(),
        R.drawable.origin, options);
// Nexus5上，虽然density = 3
// 但是通过设置inTargetDensity / inDensity = 160 / 160 = 1
// 解码后图片大小为1024x1024
System.out.println("w:" + bitmap.getWidth()
        + ", h:" + bitmap.getHeight());
```

#### 2.读取SD卡的图片，正常大小

#### 3.获取bitmap大小的api
```
public final int getByteCount() {
    // int result permits bitmaps up to 46,340 x 46,340
    return getRowBytes() * getHeight();
}
```
总结===>bitmap的大小，由以下三个决定：
- 单像素点字节数
- 图片文件存放的drawable文件夹位置
- 手机屏幕密度

#### 4.inSampleSize
BitmapFactory.Options类有一个参数inSampleSize，该参数为int型，他的值指示了在解析图片为Bitmap时在长宽两个方向上像素缩小的倍数。inSampleSize的默认值和最小值为1（当小于1时，解码器将该值当做1来处理），且在大于1时，该值只能为2的幂（当不为2的幂时，解码器会取与该值最接近的2的幂）。例如，当inSampleSize为2时，一个20001000的图片，将被缩小为1000500，相应地，它的像素数和内存占用都被缩小为了原来的1/4。
```java
public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // 原始图片的宽高
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;
    if (height > reqHeight || width > reqWidth) {
        final int halfHeight = height / 2;
        final int halfWidth = width / 2;
        // 在保证解析出的bitmap宽高分别大于目标尺寸宽高的前提下，取可能的inSampleSize的最大值
        while ((halfHeight / inSampleSize) > reqHeight
                && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }
    return inSampleSize;
}
```

#### 5.图片加载时候的三级缓存 
具体的缓存策略可以是这样的：内存作为一级缓存，本地作为二级缓存，网络加载为最后。其中，内存使用 LruCache ，其内部通过 LinkedhashMap 来持有外界缓存对象的强引用；对于本地缓存，使用 DiskLruCache。加载图片的时候，首先使用 LRU 方式进行寻找，找不到指定内容，按照三级缓存的方式，进行本地搜索，还没有就网络加载。

#### 6.图片的缓存算法，LruCache使用的最近最少使用算法等 
LruCache最近最少使用，利用了LinkedHashMap(双向链表)实现，当访问某个item，就把这个itme移动到链表头部，超过了，就移除尾部的item就达到了目的。