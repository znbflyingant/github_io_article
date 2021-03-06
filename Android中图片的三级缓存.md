---
title: Android中图片的三级缓存
date: 2017-03-06
categories: Android
tags: 性能优化
---

##为什么要使用三级缓存
* 如今的 Android App 经常会需要网络交互，通过网络获取图片是再正常不过的事了

* 假如每次启动的时候都从网络拉取图片的话，势必会消耗很多流量。在当前的状况下，对于非wifi用户来说，

* 流量还是很贵的，一个很耗流量的应用，其用户数量级肯定要受到影响

* 特别是，当我们想要重复浏览一些图片时，如果每一次浏览都需要通过网络获取，流量的浪费可想而知

* 所以提出三级缓存策略，通过网络、本地、内存三级缓存图片，来减少不必要的网络交互，避免浪费流量

##什么是三级缓存
* 网络缓存, 不优先加载, 速度慢,浪费流量

* 本地缓存, 次优先加载, 速度快

* 内存缓存, 优先加载, 速度最快

##三级缓存原理
* 首次加载 Android App 时，肯定要通过网络交互来获取图片，之后我们可以将图片保存至本地SD卡和内存中

* 之后运行 App 时，优先访问内存中的图片缓存，若内存中没有，则加载本地SD卡中的图片

* 总之，只在初次访问新内容时，才通过网络获取图片资源

##具体实现及代码
####1、 自定义的图片缓存工具类（MyBitmapUtils）

* 通过new MyBitmapUtils().display(ImageView ivPic, String url) 提供给外部方法进行图片缓存的接口

* 参数含义：ivPic 用于显示图片的ImageView，url 获取图片的网络地址

```
 /**
     * 自定义的BitmapUtils,实现三级缓存
     */
    public class MyBitmapUtils {

        private NetCacheUtils mNetCacheUtils;
        private LocalCacheUtils mLocalCacheUtils;
        private MemoryCacheUtils mMemoryCacheUtils;

        public MyBitmapUtils(){
            mMemoryCacheUtils=new MemoryCacheUtils();
            mLocalCacheUtils=new LocalCacheUtils();
            mNetCacheUtils=new NetCacheUtils(mLocalCacheUtils,mMemoryCacheUtils);
        }

        public void disPlay(ImageView ivPic, String url) {
            ivPic.setImageResource(R.mipmap.pic_item_list_default);
            Bitmap bitmap;
            //内存缓存
            bitmap=mMemoryCacheUtils.getBitmapFromMemory(url);
            if (bitmap!=null){
                ivPic.setImageBitmap(bitmap);
                System.out.println("从内存获取图片啦.....");
                return;
            }

            //本地缓存
            bitmap = mLocalCacheUtils.getBitmapFromLocal(url);
            if(bitmap !=null){
                ivPic.setImageBitmap(bitmap);
                System.out.println("从本地获取图片啦.....");
                //从本地获取图片后,保存至内存中
                mMemoryCacheUtils.setBitmapToMemory(url,bitmap);
                return;
            }
            //网络缓存
            mNetCacheUtils.getBitmapFromNet(ivPic,url);
        }
    }
```

####2、 网络缓存（NetCacheUtils）
* 网络缓存中主要用到了AsyncTask来进行异步数据的加载

* 简单来说，AsyncTask可以看作是一个对handler和线程池的封装，通常，AsyncTask主要用于数据简单时，handler+thread主要用于数据量多且复杂时，当然这也不是必须的，仁者见仁智者见智

* 同时，为了避免内存溢出的问题，我们可以在获取网络图片后。对其进行图片压缩

```
/**
     * 三级缓存之网络缓存
     */
    public class NetCacheUtils {

        private LocalCacheUtils mLocalCacheUtils;
        private MemoryCacheUtils mMemoryCacheUtils;

        public NetCacheUtils(LocalCacheUtils localCacheUtils, MemoryCacheUtils memoryCacheUtils) {
            mLocalCacheUtils = localCacheUtils;
            mMemoryCacheUtils = memoryCacheUtils;
        }

        /**
         * 从网络下载图片
         * @param ivPic 显示图片的imageview
         * @param url   下载图片的网络地址
         */
        public void getBitmapFromNet(ImageView ivPic, String url) {
            new BitmapTask().execute(ivPic, url);//启动AsyncTask

        }

        /**
         * AsyncTask就是对handler和线程池的封装
         * 第一个泛型:参数类型
         * 第二个泛型:更新进度的泛型
         * 第三个泛型:onPostExecute的返回结果
         */
        class BitmapTask extends AsyncTask<Object, Void, Bitmap> {

            private ImageView ivPic;
            private String url;

            /**
             * 后台耗时操作,存在于子线程中
             * @param params
             * @return
             */
            @Override
            protected Bitmap doInBackground(Object[] params) {
                ivPic = (ImageView) params[0];
                url = (String) params[1];

                return downLoadBitmap(url);
            }

            /**
             * 更新进度,在主线程中
             * @param values
             */
            @Override
            protected void onProgressUpdate(Void[] values) {
                super.onProgressUpdate(values);
            }

            /**
             * 耗时方法结束后执行该方法,主线程中
             * @param result
             */
            @Override
            protected void onPostExecute(Bitmap result) {
                if (result != null) {
                    ivPic.setImageBitmap(result);
                    System.out.println("从网络缓存图片啦.....");

                    //从网络获取图片后,保存至本地缓存
                    mLocalCacheUtils.setBitmapToLocal(url, result);
                    //保存至内存中
                    mMemoryCacheUtils.setBitmapToMemory(url, result);

                }
            }
        }

        /**
         * 网络下载图片
         * @param url
         * @return
         */
        private Bitmap downLoadBitmap(String url) {
            HttpURLConnection conn = null;
            try {
                conn = (HttpURLConnection) new URL(url).openConnection();
                conn.setConnectTimeout(5000);
                conn.setReadTimeout(5000);
                conn.setRequestMethod("GET");

                int responseCode = conn.getResponseCode();
                if (responseCode == 200) {
                    //图片压缩
                    BitmapFactory.Options options = new BitmapFactory.Options();
                    options.inSampleSize=2;//宽高压缩为原来的1/2
                    options.inPreferredConfig=Bitmap.Config.ARGB_4444;
                    Bitmap bitmap = BitmapFactory.decodeStream(conn.getInputStream(),null,options);
                    return bitmap;
                }
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                conn.disconnect();
            }

            return null;
        }
    }
```

####3、本地缓存（LocalCacheUtils）
* 在初次通过网络获取图片后，我们可以在本地SD卡中将图片保存起来

* 可以使用MD5加密图片的网络地址，来作为图片的名称保存

```
/**
     * 三级缓存之本地缓存
     */
    public class LocalCacheUtils {

        private static final String CACHE_PATH= Environment.getExternalStorageDirectory().getAbsolutePath()+"/WerbNews";

        /**
         * 从本地读取图片
         * @param url
         */
        public Bitmap getBitmapFromLocal(String url){
            String fileName = null;//把图片的url当做文件名,并进行MD5加密
            try {
                fileName = MD5Encoder.encode(url);
                File file=new File(CACHE_PATH,fileName);

                Bitmap bitmap = BitmapFactory.decodeStream(new FileInputStream(file));

                return bitmap;
            } catch (Exception e) {
                e.printStackTrace();
            }

            return null;
        }

        /**
         * 从网络获取图片后,保存至本地缓存
         * @param url
         * @param bitmap
         */
        public void setBitmapToLocal(String url,Bitmap bitmap){
            try {
                String fileName = MD5Encoder.encode(url);//把图片的url当做文件名,并进行MD5加密
                File file=new File(CACHE_PATH,fileName);

                //通过得到文件的父文件,判断父文件是否存在
                File parentFile = file.getParentFile();
                if (!parentFile.exists()){
                    parentFile.mkdirs();
                }

                //把图片保存至本地
                bitmap.compress(Bitmap.CompressFormat.JPEG,100,new FileOutputStream(file));
            } catch (Exception e) {
                e.printStackTrace();
            }

        }
    }
```

####4、 内存缓存（MemoryCacheUtils）
* 这是本文中最重要且需要重点介绍的部分

* 进行内存缓存，就一定要注意一个问题，那就是内存溢出（OutOfMemory）

* 为什么会造成内存溢出？
 - Android 虚拟机默认分配给每个App 16M的内存空间，真机会比16M大，但任会出现内存溢出的情况

 - Android 系统在加载图片时是解析每一个像素的信息，再把每一个像素全部保存至内存中
 - 图片大小 = 图片的总像素 * 每个像素占用的大小 
>单色图：每个像素占用1/8个字节，16色图：每个像素占用1/2个字节，256色图：每个像素占用1个字节，24位图：每个像素占用3个字节（常见的rgb构成的图片）

 -  例如一张1920x1080的JPG图片，在Android 系统中是以ARGB格式解析的，即一个像素需占用4个字节，图片的大小=1920x1080x4=7M

* 实现方法：
 - 通过 HashMap<String,Bitmap>键值对的方式保存图片，key为地址，value为图片对象，但因是强引用对象，很容易造成内存溢出，可以尝试SoftReference软引用对象

 - 通过 HashMap<String, SoftReference<Bitmap>>SoftReference 为软引用对象（GC垃圾回收会自动回收软引用对象），但在Android2.3+后，系统会优先考虑回收弱引用对象，官方提出使用LruCache

 - 通过 LruCache<String,Bitmap> least recentlly use 最少最近使用算法
会将内存控制在一定的大小内, 超出最大值时会自动回收, 这个最大值开发者自己定

```
 /**
     * 三级缓存之内存缓存
     */
    public class MemoryCacheUtils {

        // private HashMap<String,Bitmap> mMemoryCache=new HashMap<>();//1.因为强引用,容易造成内存溢出，所以考虑使用下面弱引用的方法
        // private HashMap<String, SoftReference<Bitmap>> mMemoryCache = new HashMap<>();//2.因为在Android2.3+后,系统会优先考虑回收弱引用对象,官方提出使用LruCache
        private LruCache<String,Bitmap> mMemoryCache;

        public MemoryCacheUtils(){
            long maxMemory = Runtime.getRuntime().maxMemory()/8;//得到手机最大允许内存的1/8,即超过指定内存,则开始回收
            //需要传入允许的内存最大值,虚拟机默认内存16M,真机不一定相同
            mMemoryCache=new LruCache<String,Bitmap>((int) maxMemory){
                //用于计算每个条目的大小
                @Override
                protected int sizeOf(String key, Bitmap value) {
                    int byteCount = value.getByteCount();
                    return byteCount;
                }
            };

        }

        /**
         * 从内存中读图片
         * @param url
         */
        public Bitmap getBitmapFromMemory(String url) {
            //Bitmap bitmap = mMemoryCache.get(url);//1.强引用方法
            /*2.弱引用方法
            SoftReference<Bitmap> bitmapSoftReference = mMemoryCache.get(url);
            if (bitmapSoftReference != null) {
                Bitmap bitmap = bitmapSoftReference.get();
                return bitmap;
            }
            */
            Bitmap bitmap = mMemoryCache.get(url);
            return bitmap;

        }

        /**
         * 往内存中写图片
         * @param url
         * @param bitmap
         */
        public void setBitmapToMemory(String url, Bitmap bitmap) {
            //mMemoryCache.put(url, bitmap);//1.强引用方法
            /*2.弱引用方法
            mMemoryCache.put(url, new SoftReference<>(bitmap));
            */
            mMemoryCache.put(url,bitmap);
        }
    }
```