[TOC]

# Bitmap解析

### 参考链接

* [Android Bitmap的内存大小是如何计算的？](https://ivonhoe.github.io/2017/03/22/Bitmap&Memory/)
* [Android性能优化：Bitmap详解&你的Bitmap占多大内存？](https://www.jianshu.com/p/4ba3e63c8cdc)
* [Bitmap的高效加载以及缓存机制](https://blog.csdn.net/Mr_azheng/article/details/79369881)

## 一、前言

### 1.1 基础知识

#### 1.1.1 px

* 像素(piexl)，指的是屏幕上的物理点，最小的独立显示单位。

#### 1.1.2 ppi

**含义**

* 每英寸像素数(Pixels Per Inch)。

**计算公式**

* 像素数量/英寸

* `√(水平像素值²+竖直像素值²)/对角线英寸`

**例子**

* 小米9se，分辨率`1080×2340`，5.97英寸。
* 所以`ppi=√(1080²+2340²)/5.97=432`。432ppi。

#### 1.1.3 dpi

**含义**

* 每英寸点(Dots Per Inch)

**计算**

* Android中的dpi是由ppi的值来确定的，可以参考下图。但是Android的dpi只有确定的几种。比如小米9se的ppi为432ppi。但是通过代码输出来的是480dpi。

* ![dpi](..\..\..\images\设计思想解读开源框架库\图片加载\Bitmap\dpi.png)

* ```java
  public static final int DENSITY_LOW = 120;
  public static final int DENSITY_MEDIUM = 160;
  public static final int DENSITY_TV = 213;
  public static final int DENSITY_HIGH = 240;
  public static final int DENSITY_260 = 260;
  public static final int DENSITY_280 = 280;
  public static final int DENSITY_300 = 300;
  public static final int DENSITY_XHIGH = 320;
  public static final int DENSITY_340 = 340;
  public static final int DENSITY_360 = 360;
  public static final int DENSITY_400 = 400;
  public static final int DENSITY_420 = 420;
  public static final int DENSITY_XXHIGH = 480;
  public static final int DENSITY_560 = 560;
  public static final int DENSITY_XXXHIGH = 640;
  ```

**代码获取**

* `getResources().getDisplayMetrics().densityDpi`来获取dpi。

#### 1.1.4 dp

**含义**

* 像素无关点(Density-Independent pixel)，这个是Android定义的虚拟值。

**px换算公式**

* `px = dp * (dpi / 160)`

**例子**

* 对于小米9se而言。
* `1dp*(480/160)=3px`也就是1dp=3px

## 二、Bitmap占用内存大小

### 2.1 DisplayMetrics类

#### 2.1.1 density

```java
float density = displayMetrics.density;
```

* `density`值，例如小米9se，上面已经说了`1dp=3px`，所以`density`等于3

#### 2.1.2 densityDpi

```java
int densityDpi = displayMetrics.densityDpi;
```

* `densityDpi`也就是上述说的`dpi`。`densityDpi`和`density`的对应关系如下。

  ![densityDpi](..\..\..\images\设计思想解读开源框架库\图片加载\Bitmap\densityDpi.png)

### 2.2 Bitmap格式

#### 2.2.1 Bitmap.Config源码

```java
public enum Config {
	ALPHA_8     (1),

	RGB_565     (3),

    @Deprecated
    ARGB_4444   (4),

	ARGB_8888   (5),

	RGBA_F16    (6),

    HARDWARE    (7);
}
```

* 以上为Android中`Bitmap`的 格式。位图位数越高表示可以存储的颜色信息越多，图像也就越清晰逼真。

#### 2.2.2 格式说明

* ALPHA_8：表示8位Alpha位图，每像素占1byte内存。

* RGB_565：表示R为5位，G为6位，B为5位，一共16位，每像素占2byte内存。

* ARGB_4444：表示16位位图，每像素占2byte内存（poor quality - Android Deprecated）。

* ARGB_8888：表示32位ARGB位图，每像素占4byte内存（Recommended）。

### 2.3 drawable/mipmap文件夹

#### 2.3.1 源码

* **BitmapFactory.cpp**中源码

  ```c
  scale = (float) targetDensity / density;
  .......
  scaledWidth = int(scaledWidth * scale + 0.5f);
  scaledHeight = int(scaledHeight * scale + 0.5f);
  ```

* `targetDensity`也就是`DisplayMetrics`的`densityDpi`，也就是`dpi`，对于小米9se就是480
* `density`就是文件的dpi。
* 真实bitmap的width和height要通过上面的计算得到。
* 加上0.5f目的是为了四舍五入。

#### 2.3.2 实例

* 采取上述举例的小米9se，小米9se的dpi为480，应该属于xxhdpi。假设有一张180*240px照片。
* 照片**放入**xxhdpi的话，则是正确的，依旧是180*240
* 照片**放入**xhdpi的话，`public static final int DENSITY_XHIGH = 320;`可知xhdpi的`density`为320。所以根据上面的公式。`scale=480/320`。所以`scaledWidth=180*(480/320)+0.5f=270`；`scaledHeight=240*(480/320)+0.5f=360`。所以结果是270*360。
* 照片放入hdpi的话，就是`scaledWidth=180*(480/240)+0.5f=360`；`scaledHeight=240*(480/240)+0.5f=480`，所以结果是360*480。

### 2.4 Bitmap占用的内存

#### 2.4.1 计算

* 实际像素内存大小为：``scaledWidth*scaledHeight*格式字节`

#### 2.4.2 实例

* 对于Android中，Bitmap默认格式为ARGB_8888，每像素为4字节。所以上述例子中，不同的文件夹内存大小如下
* xxhdpi的内存大小：`180*240*4=172800‬字节`。
* xhdpi的内存大小：`270*360*4=388800‬字节`。
* hdpi的内存大小：`360*480*4=691200‬字节`。

## 三、ImageView设置图片

### 3.1 setImageResource

* ```java
  setImageResource(@DrawableRes int resId)
  ```

* 需要的是`resId`，同时在官方文档中也注明了，这个方法是在UI线程中对图片读取和解析的，所以有可能对一个`Activity`的启动造成延迟。所以如果顾虑到这个官方建议用`setImageDrawable`和`setImageBitmap`来代替。

### 3.2 setImageBitmap

* ```java
  public void setImageBitmap(Bitmap bm) {
  	mDrawable = null;
      if (mRecycleableBitmapDrawable == null) {
      	mRecycleableBitmapDrawable = new BitmapDrawable(mContext.getResources(), bm);
  	} else {
      	mRecycleableBitmapDrawable.setBitmap(bm);
  	}
      setImageDrawable(mRecycleableBitmapDrawable);
  }
  ```

* 需要的是`bitmap`对象，同时我们可以通过源码发现最终调用的还是`setImageDrawable`方法。所以如果需要频繁调用这个方法的话最好自己封装个固定的`Drawable`对象，直接调用`setImageDrawable`，这样可以减少`Drawable`对象。因为每次调用`setImageBitmap`方法都会对`Bitmap`对象`new`出一个`Drawable`。多线程并发的时候可能会出现内存抖动。

### 3.3 setImageDrawable

* ```java
  setImageDrawable(@Nullable Drawable drawable)
  ```

* 需要的是`Drawable`对象，`setImageDrawable`是最省内存高效的，如果担心图片过大或者图片过多影响内存和加载效率，可以自己解析图片然后通过调用`setImageDrawable`方法进行设置。

## 四、Bitmap的创建和高效加载

### 4.1 Bitmap创建流程

#### 4.1.1 三种常用方法

* `decodeFile, decodeResource, decodeByteArray`

![Bitmap创建](..\..\..\images\设计思想解读开源框架库\图片加载\Bitmap\Bitmap创建.png)

* **decodeResourceStream**主要做了两件事：一是对 opts.inDensity 赋值，没有设置默认值 160；二是对 opts.inTargetDensity 赋值，没有赋值为当前设备 densityDpi；
* **decodeStream**主要也做了两件事：一是调用 native 方法解析 Bitmap；二是对解析得到的 Bitmap 调用 setDensityFraomOptions(bmp, opts) 进行设置；
* **setDensityFromOptions(bmp, opts)**：当opts.inDensity != opts.inTargetDensity || opts.inDensity != opts.inScreenDensity && (inScaled = true || isNinePatch) 时，将设置 outputBitmap.mDensity = inTargetDensity；

#### 4.1.2 源码

* **decodeResourceStream**

* ```java
  public static Bitmap decodeResourceStream(Resources res, TypedValue value,
              InputStream is, Rect pad, Options opts) {
  	validate(opts);
      if (opts == null) {
      	opts = new Options();
  	}
  
      if (opts.inDensity == 0 && value != null) {
      	final int density = value.density;
          if (density == TypedValue.DENSITY_DEFAULT) {
          	opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
  		} else if (density != TypedValue.DENSITY_NONE) {
          	opts.inDensity = density;
  		}
  	}
          
      if (opts.inTargetDensity == 0 && res != null) {
      	opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
  	}
          
      return decodeStream(is, pad, opts);
  }
  ```

* **setDensityFromOptions**

* ```java
  private static void setDensityFromOptions(Bitmap outputBitmap, Options opts) {
  	if (outputBitmap == null || opts == null) return;
  
      final int density = opts.inDensity;
      if (density != 0) {
      	outputBitmap.setDensity(density);
          final int targetDensity = opts.inTargetDensity;
          if (targetDensity == 0 || density == targetDensity 
              || density == opts.inScreenDensity) {
          	return;
  		}
  
  		byte[] np = outputBitmap.getNinePatchChunk();
          final boolean isNinePatch = np != null && NinePatch.isNinePatchChunk(np);
          if (opts.inScaled || isNinePatch) {
          	outputBitmap.setDensity(targetDensity);
  		}
  	} else if (opts.inBitmap != null) {
      	// bitmap was reused, ensure density is reset
          outputBitmap.setDensity(Bitmap.getDefaultDensity());
  	}
  }
  ```

### 4.2 BitmapFactory类

#### 4.2.1 重要参数

* inJustDecodeBounds：为true时仅返回 Bitmap 宽高等属性，返回bmp=null；为false时才返回占内存的 bmp。
* inSampleSize：表示 Bitmap 的压缩比例，值必须 大于1 & 是2的幂次方。inSampleSize = 2 时，表示压缩宽高各1/2，最后返回原始图1/4大小的Bitmap。  

### 4.3 Bitmap的高效加载

#### 4.3.1 采样率方式

* 获取需要的长和宽，一般获取控件的长和宽。

* 设置`BitmapFactory.Options`中的`inJustDecodeBounds`为true，可以帮助我们在不加载进内存的方式获得`Bitmap`的长和宽。

* 对需要的长和宽和Bitmap的长和宽进行对比，从而获得压缩比例，放入`BitmapFactory.Options`中的`inSampleSize`属性。

* 设置`BitmapFactory.Options`中的`inJustDecodeBounds`为false，将图片加载进内存，进而设置到控件中。

* ```java
  //width,height一般为imageview的大小
  private Bitmap setBitmapImage(Resources res, int id, int width, int height){
  	BitmapFactory.Options options = new BitmapFactory.Options();
      options.inJustDecodeBounds = true;
      BitmapFactory.decodeResource(res,id,options);
  
      //获取inSampleSize值
      options.inSampleSize = getinSampleSize(options,width,height);
  
      options.inJustDecodeBounds = false;
      return BitmapFactory.decodeResource(res,id,options);
  }
  
  private int getinSampleSize(BitmapFactory.Options options, int width, int height) {
  	int w = options.outWidth;
      int h = options.outHeight;
      int inSampleSize = 1;
      if (w > width || h > height){
      	int halfw = w / 2;
          int halfh = h / 2;
          while ((halfw / inSampleSize) >= width && (halfh / inSampleSize) >= height){
          	inSampleSize *= 2;
  		}
  	}
      return inSampleSize;
  }
  ```

  
