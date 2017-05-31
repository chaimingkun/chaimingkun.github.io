title: Volley是否可以代替图片加载框架
date: 2015-08-18 20:07:42
tags: Volley
---
volley出来之后，陆续的出来很多博客讲解volley的使用方法，大多数的讲解仅仅是停留着用的角度上，并没有根据volley的源码来具体情况来具体分析，这就导致很多技术能力稍弱的朋友模仿出来后反而导致应用的性能降低。那么本文就根据volley源码来分析，vollly如何使用才是最合适的。

volley+okhttp是很好的搭配，okhttp可以说是对大多数人熟知，如果不了解，自行搜索okhttp的强大之处，总之就是okhttp比httpclient、URLConnection牛逼，如果不知道两者之间如何搭配可以看一下我写的[demo](https://github.com/chaimingkun/volleyOkhttp)（该demo还包含了volley应该如何使用）以及我的blog[Android 网络: Volley+OkHttp+Https](http://chaimingkun.github.io/2015/08/14/Android-%E7%BD%91%E7%BB%9C-Volley-OkHttp-Https/)。

### volley 加载图片模块可以替代强大的网络框架吗？
现在最强大且最常用的网络框架有哪些？
- Picasso
- ImageLoader
- Glide
- Fresco 这是facebook出品的一个强大图片框架

那么问题来了，volley的图片加载能否代替这些框架？
那么我们看看volley是如何加载一张图片的：
 大家应该都知道Volley中的ImageLoader其实就是加强版的ImageRequest,ImageLoader的构造函数需要传入一个实现了ImageCache 的一个LruCache(官方建议)，那么显而易见，我们 就应该从这个ImageLoader入手了。

ImageLoader的构造方法
```java
/**
 *传入RequestQueue对象及实现了实现了ImageCache *的一个LruCache
*/
public ImageLoader(RequestQueue queue, ImageCache imageCache) {
        mRequestQueue = queue;
        mCache = imageCache;
    }
```
ImageLoader加载图片的方法
```java
    /**
     *
     */
    public ImageContainer get(String requestUrl, ImageListener imageListener,
            int maxWidth, int maxHeight, ScaleType scaleType) {

        // 判断是否是在主线程调用，如果不是抛异常
        throwIfNotOnMainThread();
        //得到改图片请求的key
        final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight, scaleType);

        // 检查在缓存中是否含有该图片如果有则直接返回该图片
        Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
        if (cachedBitmap != null) {
            // Return the cached bitmap.
            ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
            imageListener.onResponse(container, true);
            return container;
        }

        
        ImageContainer imageContainer =
                new ImageContainer(null, requestUrl, cacheKey, imageListener);

        // 如果缓存中没有，那么通知调用者显示默认加载图片
        imageListener.onResponse(imageContainer, true);

        // 判断队列中请求是否已经存在了
        BatchedImageRequest request = mInFlightRequests.get(cacheKey);
        if (request != null) {
            // 如果存在则添加该监听
            request.addContainer(imageContainer);
            return imageContainer;
        }

        // 否则则添加该imageRequest到请求队列中
        Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType,
                cacheKey);

        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey,
                new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;
    }
```
NetworkDispatcher实际上就是继承自Thread的一个新线程，它负责不断同RequestQueue中不断取出Request来取获取。那么我们分析一下NetworkDispatcher中的run方法
```java
    @Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request<?> request;
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            // release previous request object to avoid leaking request object when mQueue is drained.
            request = null;
            try {
                //从RequestQueue中取出一个request
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // 如果改请求已经被取消，那么挑出该次循环，继续获取新的request
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // 这才是真真正正网络获取数据，mNetwork实际上就是HttpStack
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // 这个是用相应的request将从网络上获取到的数据转换为相应的数据
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);
            } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }
```
好，我们就看看parseNetworkResponse
```java
    @Override
    protected Response<Bitmap> parseNetworkResponse(NetworkResponse response) {
        // Serialize all decode on a global lock to reduce concurrent heap usage.
        synchronized (sDecodeLock) {
            try {
                //看下面的方法
                return doParse(response);
            } catch (OutOfMemoryError e) {
                VolleyLog.e("Caught OOM for %d byte image, url=%s", response.data.length, getUrl());
                return Response.error(new ParseError(e));
            }
        }
    }

    
    private Response<Bitmap> doParse(NetworkResponse response) {
        //注意！！！看到了什么~直接将整个图片的数据放到字节数组里，也就是说直接把请求的图片数据放到了内存里。好，我们做最后的总结。
        byte[] data = response.data;
        BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
        Bitmap bitmap = null;
        if (mMaxWidth == 0 && mMaxHeight == 0) {
            decodeOptions.inPreferredConfig = mDecodeConfig;
            bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
        } else {
            // If we have to resize this image, first get the natural bounds.
            decodeOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
            int actualWidth = decodeOptions.outWidth;
            int actualHeight = decodeOptions.outHeight;

            // Then compute the dimensions we would ideally like to decode to.
            int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight,
                    actualWidth, actualHeight, mScaleType);
            int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth,
                    actualHeight, actualWidth, mScaleType);

            // Decode to the nearest power of two scaling factor.
            decodeOptions.inJustDecodeBounds = false;
            // TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
            // decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
            decodeOptions.inSampleSize =
                findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
            Bitmap tempBitmap =
                BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);

            // If necessary, scale down to the maximal acceptable size.
            if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth ||
                    tempBitmap.getHeight() > desiredHeight)) {
                bitmap = Bitmap.createScaledBitmap(tempBitmap,
                        desiredWidth, desiredHeight, true);
                tempBitmap.recycle();
            } else {
                bitmap = tempBitmap;
            }
        }

        if (bitmap == null) {
            return Response.error(new ParseError(response));
        } else {
            return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
        }
    }
```
从上面的分析可以看出，volley是现将请求到的数据放到内存中，在进行相应的转换，如果是一张大图，那么极有可能会导致OOM。所以，如果你的app中只有少量图片并且是图片很小，那么用volley是可以的，否则还是乖乖使用你的图片加载框架吧。

总而言之一句话，如果你的app有大图，或者有大量的图片，并不适合用volley替代你的图片加载框架。

既然本文提到了这几个图片加载框架，后面我会写一篇博客专门介绍这些图片框架的优缺点及使用注意事项，敬请期待。
 
 
 