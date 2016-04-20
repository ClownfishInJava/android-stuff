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
在getByteCount中分别用到了getRowBytes和getHeight来计算Bitmap的大小；getHeight就是获取Bitmap的高度；getRowBytes??

	public final int getrowBytes() {
   	if (mRecycled) {
          Log.w(TAG, "Called getRowBytes() on a 		recycle()'d bitmap! This is undefined behavior!");
   	}
   		return nativeRowBytes(mFinalizer.mNativeBitmap);
	}
	
getRowBytes实际调用的native方法

	static jint Bitmap_rowBytes(JNIEnv* env, jobject, jlong bitmapHandle) {
     SkBitmap* bitmap = reinterpret_cast<SkBitmap*>(bitmapHandle)
     return static_cast<jint>(bitmap->rowBytes());
	}
	
从Native代码中，发现Bitmap本质上是SkBitmap。看SkBitmap代码

	/** Return the number of bytes between subsequent rows of the bitmap. */
	size_t rowBytes() const { return fRowBytes; }
	
	size_t SkBitmap::ComputeRowBytes(Config c, int width) {
    return SkColorTypeMinRowBytes(SkBitmapConfigToColorType(c), width);
	}
SkImageInfo.h
 
	static int SkColorTypeBytesPerPixel(SkColorType ct) {
   	static const uint8_t gSize[] = {
    0,  // Unknown
    1,  // Alpha_8
    2,  // RGB_565
    2,  // ARGB_4444
    4,  // RGBA_8888
    4,  // BGRA_8888
    1,  // kIndex_8
  	};
  	SK_COMPILE_ASSERT(SK_ARRAY_COUNT(gSize) == (size_t)(kLastEnum_SkColorType + 1),
                size_mismatch_with_SkColorType_enum);
 
   	SkASSERT((size_t)ct < SK_ARRAY_COUNT(gSize));
   	return gSize[ct];
	}
 
	static inline size_t SkColorTypeMinRowBytes(SkColorType ct, int width) {
    return width * SkColorTypeBytesPerPixel(ct);
	}

好，最后我们发现ARGB_8888（我们最常用的Bitmap格式）的一个像素占用4byte，那么rowBytes实际上就是4*width byte。
那么结论就是

一张 ARGB_8888 的 Bitmap 占用内存的计算公式

	bitmapInRam = bitmapWidth*bitmapHeight *4 bytes
	
然后，还有坑。你会发现你算出来的和你获取到的值之间差了倍数~~

	一张522*686的 PNG 图片，我把它放到 drawable-xxhdpi 目录下，在三星s6上加载，占用内存2547360B，就可以用这个方法获取到。然而公式计算出来的可是1432368B

####Density
为什么举例要强调xxx目录，xxx手机，Bitmap加载只和宽高有关吗？

我们读取的是drawable目录下面的图片，用的是decodeResource方法，该方法本质上完成：

* 读取原始资源，这个调用了 Resource.openRawResource 方法，这个方法调用完成之后会对 TypedValue 进行赋值，其中包含了原始资源的 density 等信息；
* 调用 decodeResourceStream 对原始资源进行解码和适配。这个过程实际上就是原始资源的 density 到屏幕 density 的一个映射。

原始资源的 density 其实取决于资源存放的目录（比如 xxhdpi 对应的是480。而屏幕的density，看BitmapFactory源码

	public static Bitmap decodeResourceStream(Resources res, TypedValue value,    InputStream is, Rect pad, Options opts) { 	//实际上，我们这里的opts是null的，所以在这里初始化。	if (opts == null) {    opts = new Options();	} 	if (opts.inDensity == 0 && value != null) {    final int density = value.density;    if (density == TypedValue.DENSITY_DEFAULT) {        opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;    } else if (density != TypedValue.DENSITY_NONE) {        opts.inDensity = density; //这里density的值如果对应资源目录为hdpi的话，就是240    }	} 	if (opts.inTargetDensity == 0 && res != null) {	//请注意，inTargetDensity就是当前的显示密度，比如三星s6时就是640    opts.inTargetDensity = res.getDisplayMetrics().densityDpi;	} 	return decodeStream(is, pad, opts);	}

我们看到 opts 这个值被初始化，而它的构造居然如此简单：

	public Options() {
   		inDither = false;
   		inScaled = true;
   		inPremultiplied = true;
	}

所以我们就很容易的看到，Option.inScreenDensity 这个值没有被初始化，而实际上后面我们也会看到这个值根本不会用到；我们最应该关心的是什么呢？是 inDensity 和 inTargetDensity，这两个值与下面 cpp 文件里面的 density 和 targetDensity 相对应——重复一下，inDensity 就是原始资源的 density，inTargetDensity 就是屏幕的 density。

紧接着，用到了 nativeDecodeStream 方法，不重要的代码直接略过，直接给出最关键的 doDecode 函数的代码：

	static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding, jobject options) { 	......    if (env->GetBooleanField(options, gOptions_scaledFieldID)) {        const int density = env->GetIntField(options, gOptions_densityFieldID);//对应hdpi的时候，是240        const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);//三星s6的为640        const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);        if (density != 0 && targetDensity != 0 && density != screenDensity) {            scale = (float) targetDensity / density;        }    }	} 	const bool willScale = scale != 1.0f;	......	SkBitmap decodingBitmap;	if (!decoder->decode(stream, &decodingBitmap, prefColorType,decodeMode)) {   	return nullObjectReturn("decoder->decode returned false");	}	//这里这个deodingBitmap就是解码出来的bitmap，大小是图片原始的大小	int scaledWidth = decodingBitmap.width();	int scaledHeight = decodingBitmap.height();	if (willScale && decodeMode !=SkImageDecoder::kDecodeBounds_Mode) {    scaledWidth = int(scaledWidth * scale + 0.5f);    scaledHeight = int(scaledHeight * scale + 0.5f);	}	if (willScale) {    const float sx = scaledWidth / float(decodingBitmap.width());    const float sy = scaledHeight / float(decodingBitmap.height());     // TODO: avoid copying when scaled size equals decodingBitmap size    SkColorType colorType = colorTypeForScaledOutput(decodingBitmap.colorType());    // FIXME: If the alphaType is kUnpremul and the image has alpha, the    // colors may not be correct, since Skia does not yet support drawing    // to/from unpremultiplied bitmaps.    outputBitmap->setInfo(SkImageInfo::Make(scaledWidth, scaledHeight,            colorType, decodingBitmap.alphaType()));    if (!outputBitmap->allocPixels(outputAllocator, NULL)) {        return nullObjectReturn("allocation failed for scaled bitmap");    }     // If outputBitmap's pixels are newly allocated by Java, there is no need    // to erase to 0, since the pixels were initialized to 0.    if (outputAllocator != &javaAllocator) {        outputBitmap->eraseColor(0);    }     SkPaint paint;    paint.setFilterLevel(SkPaint::kLow_FilterLevel);     SkCanvas canvas(*outputBitmap);    canvas.scale(sx, sy);    canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);		}	......	}
注意到其中有个 density 和 targetDensity，前者是 decodingBitmap 的 density，这个值跟这张图片的放置的目录有关（比如 hdpi 是240，xxhdpi 是480），这部分代码我跟了一下，太长了，就不列出来了；targetDensity 实际上是我们加载图片的目标 density，这个值的来源我们已经在前面给出了，就是 DisplayMetrics 的 densityDpi，如果是三星s6那么这个数值就是640。sx 和sy 实际上是约等于 scale 的，因为 scaledWidth 和 scaledHeight 是由 width 和 height 乘以 scale 得到的。我们看到 Canvas 放大了 scale 倍，然后又把读到内存的这张 bitmap 画上去，相当于把这张 bitmap 放大了 scale 倍。
####精度
越来越有趣了是不是，你肯定会发现我们这么细致的计算还是跟获取到的数值不一样~~
结果已经很接近了，是不是精度问题？？！！
	outputBitmap->setInfo(SkImageInfo::Make(scaledWidth, scaledHeight,            colorType, decodingBitmap.alphaType()));
 我们看到最终输出的 outputBitmap 的大小是scaledWidth*scaledHeight，我们把这两个变量计算的片段拿出来给大家一看就明白了：
 	if (willScale && decodeMode != SkImageDecoder::kDecodeBounds_Mode) {    scaledWidth = int(scaledWidth * scale + 0.5f);    scaledHeight = int(scaledHeight * scale + 0.5f);	}
在我们的例子中，
scaledWidth = int( 522 * 640 / 480f + 0.5) = int(696.5) = 696
scaledHeight = int( 686 * 640 / 480f + 0.5) = int(915.16666…) = 915
下面就是见证奇迹的时刻：
915 * 696 * 4 = 2547360
有木有很兴奋！有木有很激动！！
写到这里，突然想起《STL源码剖析》一书的扉页，侯捷先生只写了一句话：
“源码之前，了无秘密”。

####小节
综上，Bitmap在内存中的大小取决于：

* 色彩格式，前面我们已经提到，如果是 ARGB8888 那么就是一个像素4个字节，如果是 RGB565 那就是2个字节
* 原始文件存放的资源目录（是 hdpi 还是 xxhdpi 可不能傻傻分不清楚哈）
* 目标屏幕的密度（所以同等条件下，红米在资源方面消耗的内存肯定是要小于三星S6的）

####想办法减少Bitmap内存占用
######Jpg和Png
同样一张图片，jpg格式会比png格式会小一些，原因很简单，jpg是一种有损压缩的图片存储格式，而png是无损压缩的图片存储格式，显而易见，jpg会比png小，代价也是显而易见的。
可是，这说的是文件存储范畴的事，它们只存在于文件系统，而非内存或者显存。打个比方，jpg是压缩包，png是解压之后的东西。

jpg和png格式 在内存上有什么不一样~~
jpg图片读到的内存会小？因为jpg图片没有alpha通道，所以读到内存的时候如果用RGB565格式存到内存，这下大小只有ARGB8888的一半~~
抛开Android平台不谈，从出图的角度看，jpg格式的图片也不一定比png的小，这取决于图像信息的内容：

	JPG 不适用于所含颜色很少、具有大块颜色相近的区域或亮度差异十分明显的较简单的图片。对于需要高保真的较复杂的图像，PNG 虽然能无损压缩，但图片文件较大。
如果仅仅是为了Bitmap读到内存中的大小而考虑的话，jpg和png，没有实质的差别；二者的差别主要体现在：

* alpha 你是否真的需要？如果需要alpha通道，那没有选择，只能用png
* 你的图片值丰富还是单调？就像刚才提到的，如果色值丰富，那么用jpg，如果作为btn的背景，用png
* 对安装包大小的要求是否严格？如果你的app资源很少，安装包大小问题不是很凸显，看情况选择jpg和png（不过现在对资源文件没有苛求的应用会很少~~~）
* 目标用户的cpu是否强劲？jpg的图像压缩算法比png耗时。这方面还要酌情考虑（在cocos2dx中，由于资源比较多，会要求统一使用png）

所以，jpg和png的选择，于减少内存基本没有关系~~

######使用inSampleSize
这个方法主要用在图片资源本身较大，或者适当地采样并不会影响视觉效果的条件下，这时候我们输出地目标可能相对较小，对图片分辨率、大小要求不是非常严格。
举例
现在有个需求，要求将一张图片进行模糊，然后作为ImageView的src呈现给用户，而我们的原始图片大小为1080*1920；如果我们直接拿来模糊的话，一方面模糊的过程费时费力，另一方面生成的图片又占用内存，实际上在模糊运算过程中可能会存在输入和输出并存的情况，此时内存将会有一个短暂的峰值。这时候就有可能OOM。
既然图片最后是要模糊的，也看不清，就还不如直接用一张采样后的图片，如果采样率为2，那么读出来的图片只有原始图片的1/4大小

	BitmapFactory.Options options = new Options();	options.inSampleSize = 2;	Bitmap bitmap = BitmapFactory.decodeResource(getResources(), resId, options);
#####使用矩阵
用到Bitmap的地方，总会见到Matrix。其实，Bitmap的像素点阵，也就是个矩阵。
**大图小用用采用，小图大用用矩阵**
还是用前面模糊图片的例子，采样之后内存小了，图的尺寸也小了。我要用Canvas绘制这张图怎么办？当然用矩阵~
	Matrix matrix = new Matrix();	matrix.preScale(2, 2, 0f, 0f);	//如果使用直接替换矩阵的话，在Nexus6 5.1.1上必须关闭硬件加速	canvas.concat(matrix);	canvas.drawBitmap(bitmap, 0,0, paint);
需要注意的是，在使用搭载5.1.1原生系统的Nexus6进行测试是时发现，如果使用Canvas的setMatrix方法，bitmap将不会出现在屏幕上。因此尽量使用canvas的scale、ratate方法，或者使用concat方法。
	Matrix matrix = new Matrix();	matrix.preScale(2, 2, 0, 0);	canvas.drawBitmap(bitmap, matrix, paint);
	
这样，绘制出来的图就是放大以后的效果了，不过占用的内存却仍然是我们采样出来的大小。
如果要把图片放到ImageView中？一样可以

	Matrix matrix = new Matrix();	matrix.postScale(2, 2, 0, 0);	imageView.setImageMatrix(matrix);	imageView.setScaleType(ScaleType.MATRIX);	imageView.setImageBitmap(bitmap);
#####合理选择Bitmap的像素格式
前面已经提到这个问题，ARGB8888格式的图片，每像素占用4字节，而RGB则是2字节。
|	格式	|	描述	||-------|------|
|	ALPHA_8	|只有一个alpha通道	|
|	ARGB_4444	|这个从API 13开始不建议使用，因为质量太差|
|	ARGB_8888	|ARGB四个通道，每个通道8bit	|
|	RGB_565	|每个像素占2Byte，其中红色占5bit，绿色占6bit，蓝色占5bit|	
其中，ALPHA8没必要用，随便用个色值就行；AGRB4444虽然占用内存只有ARGB8888的一半，不过官方已经弃用了；
ARGB8888最常用，应该比较熟悉
RGB565，如果不需要alpha通道，特别是资源本身为jpg格式的情况下，用这个格式比较理想

#####索引位图（Indexed Bitmap）
索引位图，每个像素只占1Byte，不仅支持RGB，还支持alpha，而且看上去效果不错~但是Android官方不支持这个
	
	public enum Config {    // these native values must match up with the enum in SkBitmap.h     ALPHA_8     (2),    RGB_565     (4),    ARGB_4444   (5),    ARGB_8888   (6);     final int nativeInt;	}
不过，Skia引擎是支持的：
	enum Config {   		kNo_Config,   //!< bitmap has not been configured     	kA8_Config,   //!< 8-bits per pixel, with only alpha specified (0 is transparent, 0xFF is opaque) 	   //看这里看这里！！↓↓↓↓↓    kIndex8_Config, //!< 8-bits per pixel, using SkColorTable to specify the colors      kRGB_565_Config, //!< 16-bits per pixel, (see SkColorPriv.h for packing)    kARGB_4444_Config, //!< 16-bits per pixel, (see SkColorPriv.h for packing)    kARGB_8888_Config, //!< 32-bits per pixel, (see SkColorPriv.h for packing)    kRLE_Index8_Config,     kConfigCount	};
其实Java层的枚举变量的nativeInt对应的就是Skia库中的枚举的索引值，所以如果我们能拿到这个索引是不是就可以了？可是好像拿不到
不过，在png的解码库中有这么一段代码：
	bool SkPNGImageDecoder::getBitmapColorType(png_structp png_ptr, png_infop info_ptr,                                       SkColorType* colorTypep,                                       bool* hasAlphap,                                       SkPMColor* SK_RESTRICT theTranspColorp) {	png_uint_32 origWidth, origHeight;	int bitDepth, colorType;	png_get_IHDR(png_ptr, info_ptr, &origWidth, &origHeight, &bitDepth,             &colorType, int_p_NULL, int_p_NULL, int_p_NULL); 		#ifdef PNG_sBIT_SUPPORTED  		// check for sBIT chunk data, in case we should disable dithering because  		// our data is not truely 8bits per component 	 	png_color_8p sig_bit;  		if (this->getDitherImage() && png_get_sBIT(png_ptr, info_ptr, &sig_bit)) {		#if 0    	SkDebugf("----- sBIT %d %d %d %d\n", sig_bit->red, sig_bit->green,             sig_bit->blue, sig_bit->alpha);	#endif    // 0 seems to indicate no information available    if (pos_le(sig_bit->red, SK_R16_BITS) &&        pos_le(sig_bit->green, SK_G16_BITS) &&        pos_le(sig_bit->blue, SK_B16_BITS)) {        this->setDitherImage(false);    	}	}	#endif  	if (colorType == PNG_COLOR_TYPE_PALETTE) {    bool paletteHasAlpha = hasTransparencyInPalette(png_ptr, info_ptr);    *colorTypep = this->getPrefColorType(kIndex_SrcDepth, paletteHasAlpha);    // now see if we can upscale to their requested colortype    //这段代码，如果返回false，那么colorType就被置为索引了，那么我们看看如何返回false    if (!canUpscalePaletteToConfig(*colorTypep, paletteHasAlpha)) {        *colorTypep = kIndex_8_SkColorType;    }	} else {	...... 	}	return true;	}
	
canUpscalePaletteToConfig 函数如果返回false，那么 colorType 就被置为 kIndex_8_SkColorType 了。
	static bool canUpscalePaletteToConfig(SkColorType dstColorType, bool srcHasAlpha) {  		switch (dstColorType) {    		case kN32_SkColorType:    		case kARGB_4444_SkColorType:        		return true;    		case kRGB_565_SkColorType:        		// only return true if the src is opaque (since 565 is opaque)        		return !srcHasAlpha;    		default:        		return false;			}		}
如果传入的 dstColorType 是 kRGB_565_SkColorType，同时图片还有 alpha 通道，那么返回 false~~咳咳，那么问题来了，这个dstColorType 是哪儿来的？？就是我们在 decode 的时候，传入的 Options 的 inPreferredConfig。

下面是实验时间

**准备**：在 assets 目录当中放了一个叫 index.png 的文件，大小192*192，这个文件是通过 PhotoShop 编辑之后生成的索引格式的图片。

	try {   	Options options = new Options();   	options.inPreferredConfig = Config.RGB_565;	Bitmap bitmap = BitmapFactory.decodeStream(getResources().getAssets().open("index.png"), null, options);   	Log.d(TAG, "bitmap.getConfig() = " + bitmap.getConfig());   	Log.d(TAG, "scaled bitmap.getByteCount() = " + bitmap.getByteCount());   	imageView.setImageBitmap(bitmap);	} catch (IOException e) {    	e.printStackTrace();	}
	
程序运行在 Nexus6上，由于从 assets 中读取不涉及前面讨论到的 scale 的问题，所以这张图片读到内存以后的大小理论值（ARGB8888）：
192 * 192 *4=147456

运行我们的代码，看输出的Configuration和ByteCOunt：

	D/MainActivity: bitmap.getConfig() = null	D/MainActivity: scaled bitmap.getByteCount() = 36864

先说大小为什么只有 36864，我们知道如果前面的讨论是没有问题的话，那么这次解码出来的 Bitmap 应该是索引格式，那么占用的内存只有 ARGB 8888 的1/4是意料之中的；再说 Config 为什么为 null。。额。。黑户。。官方说：

	public final Bitmap.Config getConfig ()
	Added in API level 1
	If the bitmap’s internal config is in one of the public formats, return that config, otherwise return null.

看来这个法子还真行啊，占用内存一下小很多。不过由于官方并未做出支持，因此这个方法有诸多限制，比如不能在 xml 中直接配置，，生成的 Bitmap 不能用于构建 Canvas 等等

###Bitmap的OOM
android的OOM是一个经常碰到的问题~~一种简便的解决方式就是分配更少的内存空间来存储，即在载入图片的时候以牺牲图片质量为代价，将图片进行缩放。但是，这种方法得不偿失，牺牲了图片质量。
在BitmapFactory中有一个内部类BitmapFactory.Options，其中值得我们注意的是inSampleSize和inJustDecodeBounds两个属性

* inSampleSize是以2的指数的倒数被进行放缩

  If set to a value > 1, requests the decoder to subsample the original image, returning a smaller image to save memory. (1 -> decodes full size; 2 -> decodes 1/4th size; 4 -> decode 1/16th size). Because you rarely need to show and have full size bitmap images on your phone. For manipulations smaller sizes are usually enough.
* inJustDecodeBounds为Boolean型

  设置inJustDecodeBounds为true后，decodeFile并不分配空间，但可计算出原始图片的长度和宽度，即options.outWidth和options.outHeight。
  
要对图片进行缩放，最大的问题就是怎么运行时改变inSampleSize的值，通过上面的inJustDecodeBounds可以知道图片原始的大小，那么这样就可以通过算法来得到一个恰当的inSampleSize。其动态算法可参考：

	/**
 	* compute Sample Size
 	* 
 	* @param options
 	* @param minSideLength
 	* @param maxNumOfPixels
 	* @return
 	*/
	public static int computeSampleSize(BitmapFactory.Options options,
		int minSideLength, int maxNumOfPixels) {
	int initialSize = computeInitialSampleSize(options, minSideLength,
			maxNumOfPixels);

	int roundedSize;
	if (initialSize <= 8) {
		roundedSize = 1;
		while (roundedSize < initialSize) {
			roundedSize <<= 1;
		}
	} else {
		roundedSize = (initialSize + 7) / 8 * 8;
	}

	return roundedSize;
	}

	/**
 	* compute Initial Sample Size
 	* 
 	* @param options
 	* @param minSideLength
 	* @param maxNumOfPixels
 	* @return
	*/
	private static int computeInitialSampleSize(BitmapFactory.Options options,
		int minSideLength, int maxNumOfPixels) {
	double w = options.outWidth;
	double h = options.outHeight;

	// 上下限范围
	int lowerBound = (maxNumOfPixels == -1) ? 1 : (int) Math.ceil(Math
			.sqrt(w * h / maxNumOfPixels));
	int upperBound = (minSideLength == -1) ? 128 : (int) Math.min(
			Math.floor(w / minSideLength), Math.floor(h / minSideLength));

	if (upperBound < lowerBound) {
		// return the larger one when there is no overlapping zone.
		return lowerBound;
	}

	if ((maxNumOfPixels == -1) && (minSideLength == -1)) {
		return 1;
	} else if (minSideLength == -1) {
		return lowerBound;
	} else {
		return upperBound;
	}
	}
	
有了上面的算法，我们可以轻易的get到Bitmap

	/**
 	* get Bitmap
 	* 
 	* @param imgFile
 	* @param minSideLength
 	* @param maxNumOfPixels
 	* @return
 	*/
	public static Bitmap tryGetBitmap(String imgFile, int minSideLength,
		int maxNumOfPixels) {
	if (imgFile == null || imgFile.length() == 0)
		return null;

	try {
		FileDescriptor fd = new FileInputStream(imgFile).getFD();
		BitmapFactory.Options options = new BitmapFactory.Options();
		options.inJustDecodeBounds = true;
		// BitmapFactory.decodeFile(imgFile, options);
		BitmapFactory.decodeFileDescriptor(fd, null, options);

		options.inSampleSize = computeSampleSize(options, minSideLength,
				maxNumOfPixels);
		try {
			// 这里一定要将其设置回false，因为之前我们将其设置成了true
			// 设置inJustDecodeBounds为true后，decodeFile并不分配空间，即，BitmapFactory解码出来的Bitmap为Null,但可计算出原始图片的长度和宽度
			options.inJustDecodeBounds = false;

			Bitmap bmp = BitmapFactory.decodeFile(imgFile, options);
			return bmp == null ? null : bmp;
		} catch (OutOfMemoryError err) {
			return null;
		}
	} catch (Exception e) {
		return null;
	}
	}

###Bitmap.option的使用
首先列出BitmapFactory.options选项的所有字段

|	|	Field|Description	|
|---|--------|-------------|
|public Bitmap|inBitmap|	|
|public int	|inDensity|	|
|public boolean|inDither|	|
|public boolean|inInputShareable|	|
|public boolean|inJustDecodeBounds|	|
|public boolean|inMutable|	|
|public boolean|inPreferQualityOverSpeed|	|
|public Bitmap.Config|inPreferredCOnfig|	|
|public boolean|inPurgeable|	|
|public int|inSampleSize|	|
|public boolean|inScaled|	|
|public int|inScreenDensity|	|
|public int|inTargetDensity|	|
|public byte[]|inTempStorage|	|
|public boolean|mCancel|	|
|public int|outHeight|	|
|public String|outMimeType|	|
|public int|outWidth|	|

#####怎么获取图片大小
首先将图片转化为Bitmap，然后再利用Bitmap的getwidth()和geiHeight()方法就可以取得图片的宽和高。

但是，问题：在通过BitmapFacctory.decodeFile(String file)方法转化Bitmap时，遇到大一点的图片时，经常会碰到OOM的情况，怎么避免？

此时，就要用到BitmapFactory.options这个类：
BitmapFactory.options这个类中有一个inJustDecodeBounds这个字段；如果把它设为true，那么BitmapFactory.decodeFile（String path,Options opt）并不会真的返回一个Bitmap给你，仅仅会把它的宽高取回来，这样就不会占用太多的内存，因此也不会频繁发生OOM。

	BitmapFactory.Options options = new BitmapFactory.Options();
	options.inJustDecodeBounds = true;
	Bitmap bmp = BitmapFactory.decodeFile(path, options);
	/* 这里返回的bmp是null */

这段代码之后，options.outWidth 和 options.outHeight就是我们想要的宽和高了。

有了宽高信息，问题：怎么在图片不变形的情况下获取到图片指定大小的缩略图？
比如，需要在图片不变形的情况下得到宽度200的缩略图。首先需要计算一下缩放之后，图片的高度是多少~~

	int height = options.outHeight * 200 / options.outWidth;
	options.outWidth = 200；
	options.outHeight = height; 
	/* 这样才能真正的返回一个Bitmap给你 */
	options.inJustDecodeBounds = false;
	Bitmap bmp = BitmapFactory.decodeFile(path, options);
	image.setImageBitmap(bmp);

虽然这样得到了期望大小的ImageView，但是在执行BitmapFactory.decodeFile(path,options)时，并没有节约内存。想要节约内存，还需要使用inSampleSize这个成员变量。

	inSampleSize = options.outWidth / 200;

另外，为了节约内存还可以使用以下几个字段：

	options.inPreferredConfig = Bitmap.Config.ARGB_4444;    // 默认是Bitmap.Config.ARGB_8888
	/* 下面两个字段需要组合使用 */
	options.inPurgeable = true;
	options.inInputShareable = true;

###BitmapShader的使用
Android提供的Shader类主要是渲染图像以及一些几何图形
Shader有几个直接子类：

* BitmapShader:主要用来渲染图像
* LinearGradient：用来进行线性渲染
* RadialGradient：用来进行环形渲染
* SweepGradient：扫面渐变---围绕一个中心点扫描渐变就像电影里那种雷达扫描，用来梯度渲染
* ComposeShader：组合渲染，可以和其他几个子类组合起来使用

####BitmapShader
渲染器着色一个位图作为一个纹理。位图可以重复或设置模式

	public   BitmapShader(Bitmap bitmap,Shader.TileMode tileX,Shader.TileMode tileY)
	调用这个方法来产生一个画有一个位图的渲染器（Shader）。
	bitmap   在渲染器内使用的位图
	tileX      The tiling mode for x to draw the bitmap in.   在位图上X方向花砖模式
	tileY     The tiling mode for y to draw the bitmap in.    在位图上Y方向花砖模式

	TileMode：（一共有三种）
	CLAMP  ：如果渲染器超出原始边界范围，会复制范围内边缘染色。
	REPEAT ：横向和纵向的重复渲染器图片，平铺。
	MIRROR ：横向和纵向的重复渲染器图片，这个和REPEAT重复方式不一样，他是以镜像方式平铺。
	
具体实现

	public class BitmapShaders extends View  
	{  
    private  BitmapShader bitmapShader = null;  
    private Bitmap bitmap = null;  
    private Paint paint = null;  
    private ShapeDrawable shapeDrawable = null;  
    private int BitmapWidth  = 0;  
    private int BitmapHeight = 0;  
    public BitmapShaders(Context context)  
    {  
        super(context);  
        //得到图像  
        bitmap = ((BitmapDrawable) getResources().getDrawable(R.drawable.h)).getBitmap();    
        BitmapWidth = bitmap.getWidth();  
        BitmapHeight = bitmap.getHeight();  
        //构造渲染器BitmapShader  
        bitmapShader = new BitmapShader(bitmap,Shader.TileMode.MIRROR,Shader.TileMode.REPEAT);  
    }  
    @Override  
    protected void onDraw(Canvas canvas)  
    {  
        super.onDraw(canvas);  
        //将图片裁剪为椭圆形    
        //构建ShapeDrawable对象并定义形状为椭圆    
        shapeDrawable = new ShapeDrawable(new OvalShape());  
        //得到画笔并设置渲染器  
        shapeDrawable.getPaint().setShader(bitmapShader);  
        //设置显示区域  
        shapeDrawable.setBounds(20, 20,BitmapWidth-60,BitmapHeight-60);  
        //绘制shapeDrawable  
        shapeDrawable.draw(canvas);  
    }  
	}  


###如何根据Bitmap.Config手写Bitmap
###使用LruCache，SD卡，手机缓存
