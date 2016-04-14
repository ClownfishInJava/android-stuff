##Bitmap
##Bitmap与OOM
Bitmap是android系统封装的一个类，想手动实现，可以看[Bitmap.Config](http://developer.android.com/intl/zh-cn/reference/android/graphics/Bitmap.Config.html)
Bitmap是一个很占用内存的类。在用BitmapFactory做decode的时候，android系统根据系统的版本，会有一个解析图片的上限。在多图情形下，需要注意Bitmap的回收；此外，最佳的策略应该是将Bitmap缩至所需要的规格来使用，而不是直接使用高分辨率的大图

用下面的策略可以尽量使得生成的图片更小

    Bitmap pic = null;
        int width = 640; //典型的取屏幕宽度或者ImageView的宽度
        try {
            BitmapFactory.Options options = new BitmapFactory.Options();
            //只是得到具体宽高，并不真正decode图片
            options.inJustDecodeBounds = true;
            pic = BitmapFactory.decodeFile(path, options);
            //真正进行decode
            options.inJustDecodeBounds = false;
            //int be = options.outWidth / width;
            //不使用2的幂数缩放
            options.inSampleSize = 1;
            if (options.outWidth > width){
                //使用比例缩放
                options.inScaled = true;
                options.inDensity = options.outWidth;
                options.inTargetDensity = width;
            }
            pic = BitmapFactory.decodeFile(path, options);
        }catch (OutOfMemoryError e){
            //还是要防止OOM 
        }
        
Bitmap类还可以做一些图片的简单处理，比如圆角，灰度等效果，比较容易找到实现代码。这里给出一个取Exif的方向的函数：

    public static int getExifOrientation(String filepath) {
        int degree = 0;
        try{
            Class cls = Class.forName("android.media.ExifInterface");
            Class[] types = new Class[]{ String.class };
            Constructor cons = cls.getConstructor(types);

            types = new Class[] { String.class, Integer.TYPE };
            Method method = cls.getMethod("getAttributeInt", types);

            Object[] args = new Object[] { filepath };
            Object exif = cons.newInstance(args);

            if (exif != null) {
                args = new Object[] {"Orientation", -1};
                int orientation = (Integer) method.invoke(exif, args);
                if (orientation != -1) {
                    switch(orientation) {
                        //case android.media.ExifInterface.ORIENTATION_ROTATE_90:
                        case 6:
                            degree = 90;
                            break;
                        //case android.media.ExifInterface.ORIENTATION_ROTATE_180:
                        case 3:
                            degree = 180;
                            break;
                        //case android.media.ExifInterface.ORIENTATION_ROTATE_270:
                        case 8:
                            degree = 270;
                            break;
                    }
                }
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        return degree;
    }

另外还推荐一个比较高性能的类似高斯模糊的函数，具体见fastblur。
对于滤镜本身，还没有看到比较靠谱的开源实现，因为性能原因，估计得用NDK，用c来实现。



