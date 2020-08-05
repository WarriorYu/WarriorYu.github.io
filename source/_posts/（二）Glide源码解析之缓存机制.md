---
title: （二）Glide源码解析之缓存机制
categories:
  - Android源码分析
tags:
  - Android第三方库源码分析
date: 2020-08-05 10:33:21
---

## （二）Glide源码解析之缓存机制

### 1.  Glide的缓存介绍： 

   - 活动缓存：
   - 内存缓存
   - 磁盘缓存

### 2.  缓存Key :

从Engine的load()方法里开始分析。我们根据下面的代码看Key是怎么生成的。

```java
public <R> LoadStatus load(
    GlideContext glideContext,
    Object model,
    Key signature,
    int width,
    int height,
    Class<?> resourceClass,
    Class<R> transcodeClass,
    Priority priority,
    DiskCacheStrategy diskCacheStrategy,
    Map<Class<?>, Transformation<?>> transformations,
    boolean isTransformationRequired,
    boolean isScaleOnlyOrNoTransform,
    Options options,
    boolean isMemoryCacheable,
    boolean useUnlimitedSourceExecutorPool,
    boolean useAnimationPool,
    boolean onlyRetrieveFromCache,
    ResourceCallback cb) {
  ...
  // 这里有8个参数，其中model是图片的地址
  EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
      resourceClass, transcodeClass, options);
  ...
}
```

可以看出key的生成，是通过图片的地址、宽高以及给图片设置的其他参数一起组合，生成了Key。生成Key主要是通过在EngineKey中重写了equals()和hashCode()方法，保证所有参数相同的情况下，才认为是同一个Key。

### 3. 内存缓存

内存缓存分为两部分

- 活动缓存 （弱引用）
- 内存缓存

当Glide加载完一张图片后，首先会放到活动缓存，当需要从活动缓存移除时，会保存到内存缓存。这样下次加载同一张图片时，不需要从网络或者磁盘上加载，只要内存中有这张图片，就直接在内存中加载。既省了流量，也提高了加载显示图片的效率，因为加载内存中图片是最快的。

```java
Glide.with(this)
	.load("")
	.skipMemoryCache(false)// 默认是使用缓存的，可以自由配置
	.into(imageView);
```

从加载内存缓存的代码开始分析

```java
public class Engine implements EngineJobListener,
    MemoryCache.ResourceRemovedListener,
    EngineResource.ResourceListener {
  public <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb) {
    Util.assertMainThread();
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);


    //分析点1：活动缓存 
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      //活动缓存里有，则直接回调
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }
    //分析点2：内存缓存
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

    @Nullable
    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
      //判断是否需要缓存，不需要的话，直接返回null
      if (!isMemoryCacheable) {
        return null;
      }
      // 分析点1.1   
      //这里
      EngineResource<?> active = activeResources.get(key);
      if (active != null) {
        //在活动缓存中成功获取到，则+1,这个操作的作用就相当于记录资源的引用次数
        active.acquire();
      }
      return active;
    }
}
```

 分析1.1

```java
final class ActiveResources {
   synchronized EngineResource<?> get(Key key) {
    //ResourceWeakReference 是弱引用，GC的时候，把活动缓存回收
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
      return null;
    }
    EngineResource<?> active = activeRef.get();
    if (active == null) {
      //
      cleanupActiveReference(activeRef);
    }
    return active;
  }
}
}
```

//分析2：内存缓存

```java
public class Engine implements EngineJobListener,
    MemoryCache.ResourceRemovedListener,
    EngineResource.ResourceListener {
      
  private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    //判断是否需要缓存，不需要的话，直接返回null
    if (!isMemoryCacheable) {
      return null;
    }
    //分析点2.1：
		//获取缓存
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      //在内存缓存中成功获取到，则+1,这个操作的作用就相当于记录资源的引用次数
      cached.acquire();
      //分析点2.2
      //添加到活动缓存
      activeResources.activate(key, cached);
    }
    return cached;
  }
      
  private EngineResource<?> getEngineResourceFromCache(Key key) {
    //从缓存中取出，并且从缓存中删除
    Resource<?> cached = cache.remove(key);

    final EngineResource<?> result;
    if (cached == null) {
      result = null;
    } else if (cached instanceof EngineResource) {
      // Save an object allocation if we've cached an EngineResource (the typical case).
      result = (EngineResource<?>) cached;
    } else {
      result = new EngineResource<>(cached, true /*isMemoryCacheable*/, true /*isRecyclable*/);
    }
    return result;
}
}
```

分析2.1：

```java
public final class GlideBuilder {
Glide build(@NonNull Context context) {
   	...

    if (memoryCache == null) {
     //LruResourceCache extends LruCache,内存缓存是基于LruCache算法实现的
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
		...
}
```

分析点2.2：

activeResources就是一个弱引用的HashMap，用来缓存正在使用中的图片，我们可以看到，loadFromActiveResources()方法就是从activeResources这个HashMap当中取值的。使用activeResources来缓存正在使用中的图片，可以保护这些图片不会被LruCache算法回收掉。

