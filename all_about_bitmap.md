##All About Bitmap
###Bitmap到底占用了多少内存
设计到屏幕密度，先要弄清楚density和densityDpi两个变量的含义

* density：1 dp = density px
* densityDpi:屏幕每英寸对应多少个点（不是像素点）

在DisplayMetrics中，两者的关系是线性的：

|Table	|	|	|	|	|	|	|
|-------|---|---|---|---|---|---|
|	density		|	1	|	1.5		|	2		|	3|	3.5	|	4	|
|	denDpi	|	160	|	240	|	320	|	480	|	560	|	640|
####占了多少内存？
在客户端，每加一张图片都是需要慎重考虑的事。图片，吃内存，可能还会导致OOM。Android API提供了一个方法可以方便我们计算Bitmap的大小

	public final int getByteCount() {
    // int result permits bitmaps up to 46,340 x 46,340
    return getRowBytes() * getHeight();
	}
	
通过这个方法，我们可以获取到一张Bitmap在运行时到底占用多少内存~

举个例子
一张 522x686 的 PNG 图片，我把它放到 drawable-xxhdpi 目录下，在三星s6上加载，占用内存2547360B，就可以用这个方法获取到。

###给我一张图，我算出Bitmap多少大
每次都在运行时，调用API得到Bitmap的大小，这个办法有点笨~~作为Android应该能手动计算出Bitmap的大小。下面看getByteCount的源码
###Bitmap的OOM
###Bitmap.option的使用
###BitmapShader的使用
###如何根据Bitmap.Config手写Bitmap
###使用LruCache，SD卡，手机缓存