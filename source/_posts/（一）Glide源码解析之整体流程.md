---
title: （一）Glide源码解析之整体流程
categories:
  - Android源码分析
tags:
  - Android第三方库源码分析
date: 2020-08-05 10:31:29
---

## （一）Glide源码解析之整体流程

1. 使用Glide，我们就完全不用担心图片内存浪费，甚至是内存溢出的问题。因为Glide从来都不会直接将图片的完整尺寸全部加载到内存中，而是用多少加载多少。Glide会自动判断ImageView的大小，然后只将这么大的图片像素加载到内存当中，帮助我们节省内存开支。 

2. 我们分析这行最简单的加载图片的代码：Glide.wih(this).load(url).into(imageView);

3. #### Glide.with(...)方法：  

   with()静态方法，有以下几个重载方法

   ```java
   public class Glide implements ComponentCallbacks2 { 
   @NonNull
     public static RequestManager with(@NonNull Context context) {
       return getRetriever(context).get(context);
     }
   
     @NonNull
     public static RequestManager with(@NonNull Activity activity) {
       return getRetriever(activity).get(activity);
     }
   
     @NonNull
     public static RequestManager with(@NonNull FragmentActivity activity) {
       return getRetriever(activity).get(activity);
     }
   
     @NonNull
     public static RequestManager with(@NonNull Fragment fragment) {
       return getRetriever(fragment.getActivity()).get(fragment);
     }
   
     @SuppressWarnings("deprecation")
     @Deprecated
     @NonNull
     public static RequestManager with(@NonNull android.app.Fragment fragment) {
       return getRetriever(fragment.getActivity()).get(fragment);
     }
   
     @NonNull
     public static RequestManager with(@NonNull View view) {
       return getRetriever(view.getContext()).get(view);
     }
   }
   ```

   Glide.with(...)方法可以传入Activity、FragmentActivity、android.support.v4.app.Fragment、android.app.Fragment、View。with()方法里面都是通过getRetriever(...)方法得到RequestManagerRetriever对象。然后通过RequestManagerRetriever的get(...)方法得到RequestManager对象。 

   

   下面我们看看RequestManagerRetriever类里几个重载的get(...)方法，即获取RequestManager的几个相关方法。

   ```java
   public class RequestManagerRetriever implements Handler.Callback {
    @NonNull
     public RequestManager get(@NonNull Context context) {
       if (context == null) {
         throw new IllegalArgumentException("You cannot start a load on a null Context");
       } else if (Util.isOnMainThread() && !(context instanceof Application)) {
         if (context instanceof FragmentActivity) {
           return get((FragmentActivity) context);
         } else if (context instanceof Activity) {
           return get((Activity) context);
         } else if (context instanceof ContextWrapper) {
           return get(((ContextWrapper) context).getBaseContext());
         }
       }
   
       return getApplicationManager(context);
     }
   
     @NonNull
     public RequestManager get(@NonNull FragmentActivity activity) {
       if (Util.isOnBackgroundThread()) {
         return get(activity.getApplicationContext());
       } else {
         assertNotDestroyed(activity);
         FragmentManager fm = activity.getSupportFragmentManager();
         return supportFragmentGet(
             activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
       }
     }
   
     @NonNull
     public RequestManager get(@NonNull Fragment fragment) {
       Preconditions.checkNotNull(fragment.getActivity(),
             "You cannot start a load on a fragment before it is attached or after it is destroyed");
       if (Util.isOnBackgroundThread()) {
         return get(fragment.getActivity().getApplicationContext());
       } else {
         FragmentManager fm = fragment.getChildFragmentManager();
         return supportFragmentGet(fragment.getActivity(), fm, fragment, fragment.isVisible());
       }
     }
   
     @SuppressWarnings("deprecation")
     @NonNull
     public RequestManager get(@NonNull Activity activity) {
       if (Util.isOnBackgroundThread()) {
         return get(activity.getApplicationContext());
       } else {
         assertNotDestroyed(activity);
         android.app.FragmentManager fm = activity.getFragmentManager();
         return fragmentGet(
             activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
       }
     }
   
     @SuppressWarnings("deprecation")
     @NonNull
     public RequestManager get(@NonNull View view) {
       if (Util.isOnBackgroundThread()) {
         return get(view.getContext().getApplicationContext());
       }
   
       Preconditions.checkNotNull(view);
       Preconditions.checkNotNull(view.getContext(),
           "Unable to obtain a request manager for a view without a Context");
       Activity activity = findActivity(view.getContext());
       // The view might be somewhere else, like a service.
       if (activity == null) {
         return get(view.getContext().getApplicationContext());
       }
   
       // Support Fragments.
       // Although the user might have non-support Fragments attached to FragmentActivity, searching
       // for non-support Fragments is so expensive pre O and that should be rare enough that we
       // prefer to just fall back to the Activity directly.
       if (activity instanceof FragmentActivity) {
         Fragment fragment = findSupportFragment(view, (FragmentActivity) activity);
         return fragment != null ? get(fragment) : get(activity);
       }
   
       // Standard Fragments.
       android.app.Fragment fragment = findFragment(view, activity);
       if (fragment == null) {
         return get(activity);
       }
       return get(fragment);
     }
     
      @NonNull
     private RequestManager getApplicationManager(@NonNull Context context) {
       // Either an application context or we're on a background thread.
       if (applicationManager == null) {
         synchronized (this) {
           if (applicationManager == null) {
             // Normally pause/resume is taken care of by the fragment we add to the fragment or
             // activity. However, in this case since the manager attached to the application will not
             // receive lifecycle events, we must force the manager to start resumed using
             // ApplicationLifecycle.
   
             // TODO(b/27524013): Factor out this Glide.get() call.
             Glide glide = Glide.get(context.getApplicationContext());
             applicationManager =
                 factory.build(
                     glide,
                     new ApplicationLifecycle(),
                     new EmptyRequestManagerTreeNode(),
                     context.getApplicationContext());
           }
         }
       }
   
       return applicationManager;
     }
     
     @NonNull
     private RequestManager supportFragmentGet(
         @NonNull Context context,
         @NonNull FragmentManager fm,
         @Nullable Fragment parentHint,
         boolean isParentVisible) {
       SupportRequestManagerFragment current =
           getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
       RequestManager requestManager = current.getRequestManager();
       if (requestManager == null) {
         // TODO(b/27524013): Factor out this Glide.get() call.
         Glide glide = Glide.get(context);
         requestManager =
             factory.build(
                 glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
         current.setRequestManager(requestManager);
       }
       return requestManager;
     }
     
     @SuppressWarnings({"deprecation", "DeprecatedIsStillUsed"})
     @Deprecated
     @NonNull
     private RequestManager fragmentGet(@NonNull Context context,
         @NonNull android.app.FragmentManager fm,
         @Nullable android.app.Fragment parentHint,
         boolean isParentVisible) {
       RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
       RequestManager requestManager = current.getRequestManager();
       if (requestManager == null) {
         // TODO(b/27524013): Factor out this Glide.get() call.
         Glide glide = Glide.get(context);
         requestManager =
             factory.build(
                 glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
         current.setRequestManager(requestManager);
       }
       return requestManager;
     }
   }
   ```

   上面获取RequestManager对象的方法看着挺多，但是稍微归类一下还挺简单的。RequestManagerRetriever类中看似有很多个get()方法的重载，什么Context参数，Activity参数，Fragment参数等等，实际上只有两种情况而已，即传入Application类型的参数，和传入非Application类型的参数。

   下面我们分析这两种情况：

   - 传入Application类型的参数：这种情况会在get(...)方法内部调用getApplicationManager(context)方法，得到RequestManager对象。因为Application的生命周期就是应用的生命周期，即Glide会和应用程序的生命周期同步，不会有其他的特殊处理。如果应用程序关闭，Glide的加载也会同时终止。
   - 传入Activity、FragmentActivity、v4包下的Fragment、app包下的Fragment：这种情况最终流程是一样的，都是在get(...)方法里调用fragmentGet(...)或者supportFragmentGet(...)获取RequestManager对象。而且在fragmentGet(...)和supportFragmentGet(...)分别调用了getRequestManagerFragment(fm, parentHint, isParentVisible)和getSupportRequestManagerFragment(fm, parentHint, isParentVisible)在当前的Activity中添加一个隐藏的Fragment。之所以添加Fragment，是因为当我们加载页面、关闭页面等跟生命周期相关的操作时，Glide必须感知到生命周期方法的调用，从而开始加载图片、停止加载图片等操作，即Glide需要监听页面的生命周期方法，但是不能让开发者在每个页面都写Glide跟生命周期相关的方法的代码。又因为Activity和Fragment的生命周期方法是同步的，所有在Activity中添加一个Fragment，专门来处理跟Glide相关的操作，这样开发者无需手动再写Glide和页面生命周期方法相关的代码。即Glide通过在页面中添加Fragment，帮我们处理了生命周期相关的代码。

   注意：在非Application参数的get(...)方法里有一个Util.isOnBackgroundThread()的判断，即在非主线程当中使用Glide，不管传入的是Activity或者Fragment，统统当做Application来处理。

   总结：Glide.with()方法主要是通过传入的参数来确定图片加载的生命周期，并且得到一个RequestManager对象。

4. #### load() 

   因为Glide.with()方法返回的是RequestManager对象，所以load()方法是在RequestManager类中。

   ```java
   public class RequestManager implements LifecycleListener,ModelTypes<RequestBuilder<Drawable>>{
     @Override
     public RequestBuilder<Drawable> load(@Nullable String string) {
       return asDrawable().load(string);
     }
   
     public RequestBuilder<Drawable> asDrawable() {
         return as(Drawable.class);
     }
   
     @NonNull
     @CheckResult
     public <ResourceType> RequestBuilder<ResourceType> as(
         @NonNull Class<ResourceType> resourceClass) {
         //创建了一个RequestBuilder对象
       return new RequestBuilder<>(glide, this, resourceClass, context);
     }
   
     @NonNull
     @Override
     @CheckResult
     public RequestBuilder<TranscodeType> load(@Nullable String string) {
       return loadGeneric(string);
     }
   
     @NonNull
     private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
       //model为设置的url
       this.model = model;
       //记录是否设置了url
       isModelSet = true;
       return this;
     }
   }  
   ```

    load()方法主要是创建了一个RequestBuilder对象，并且给RequestBuilder设置了要加载的model(url)，并记录url已设置的状态。

5. #### into() 

   ```java
   @NonNull
   public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
     Util.assertMainThread();
     Preconditions.checkNotNull(view);
   
     RequestOptions requestOptions = this.requestOptions;
     if (!requestOptions.isTransformationSet()
         && requestOptions.isTransformationAllowed()
         && view.getScaleType() != null) {
       // Clone in this method so that if we use this RequestBuilder to load into a View and then
       // into a different target, we don't retain the transformation applied based on the previous
       // View's scale type.
       
       // 这个RequestOptions里保存了要设置的scaleType
       switch (view.getScaleType()) {
         case CENTER_CROP:
           requestOptions = requestOptions.clone().optionalCenterCrop();
           break;
         case CENTER_INSIDE:
           requestOptions = requestOptions.clone().optionalCenterInside();
           break;
         case FIT_CENTER:
         case FIT_START:
         case FIT_END:
           requestOptions = requestOptions.clone().optionalFitCenter();
           break;
         case FIT_XY:
           requestOptions = requestOptions.clone().optionalCenterInside();
           break;
         case CENTER:
         case MATRIX:
         default:
           // Do nothing.
       }
     }
   
   	//这个transcodeClass是指的drawable或bitmap
     //分析1
     return into(
         glideContext.buildImageViewTarget(view, transcodeClass),
         /*targetListener=*/ null,
         requestOptions);
   }
   
   @NonNull
   public <X> ViewTarget<ImageView, X> buildImageViewTarget(
       @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
     return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
   }
   
   /**
    * A factory responsible for producing the correct type of
    * {@link com.bumptech.glide.request.target.Target} for a given {@link android.view.View} subclass.
    */
   public class ImageViewTargetFactory {
     @NonNull
     @SuppressWarnings("unchecked")
     public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
         @NonNull Class<Z> clazz) {
       //如果你在使用Glide加载图片的时候调用了asBitmap()方法，那么这里就会构建出BitmapImageViewTarget对象
       if (Bitmap.class.equals(clazz)) {
         return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
       } else if (Drawable.class.isAssignableFrom(clazz)) {
         //至于DrawableImageViewTarget对象，这个通常都是用不到的，我们可以暂时不用管它。
         return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
       } else {
         throw new IllegalArgumentException(
             "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
       }
     }
   }
   
   
   ```

   上述代码在into()方法中通过buildImageViewTarget(view, transcodeClass)构建了一个ViewTarget对象，然后在into()方法的最后一行，又是一个into()方法，并且把ViewTarget这个对象当做参数，传入到了RequestBuilder中另一个接收Target对象的into()方法当中了。

   下面看一下这个into()方法。

   ```java
   private <Y extends Target<TranscodeType>> Y into(
       @NonNull Y target,
       @Nullable RequestListener<TranscodeType> targetListener,
       @NonNull RequestOptions options) {
     Util.assertMainThread();
     Preconditions.checkNotNull(target);
     if (!isModelSet) {
       throw new IllegalArgumentException("You must call #load() before calling #into()");
     }
   
     options = options.autoClone();
     //分析1：构建一个Request对象，用来加载图片请求。
     Request request = buildRequest(target, targetListener, options);
   
     Request previous = target.getRequest();
     if (request.isEquivalentTo(previous)
         && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
       request.recycle();
       // If the request is completed, beginning again will ensure the result is re-delivered,
       // triggering RequestListeners and Targets. If the request is failed, beginning again will
       // restart the request, giving it another chance to complete. If the request is already
       // running, we can let it continue running without interruption.
       if (!Preconditions.checkNotNull(previous).isRunning()) {
         // Use the previous request rather than the new one to allow for optimizations like skipping
         // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
         // that are done in the individual Request.
         previous.begin();
       }
       return target;
     }
   
     requestManager.clear(target);
     target.setRequest(request);
     //分析2：发起请求
     requestManager.track(target, request);
   
     return target;
   }
   ```

   #### 分析1：先来看buildRequest()方法是如何构建Request对象的。

   ```java
   private Request buildRequest(
       Target<TranscodeType> target,
       @Nullable RequestListener<TranscodeType> targetListener,
       RequestOptions requestOptions) {
     return buildRequestRecursive(
         target,
         targetListener,
         /*parentCoordinator=*/ null,
         transitionOptions,
         requestOptions.getPriority(),
         requestOptions.getOverrideWidth(),
         requestOptions.getOverrideHeight(),
         requestOptions);
   }
   
   private Request buildRequestRecursive(
       Target<TranscodeType> target,
       @Nullable RequestListener<TranscodeType> targetListener,
       @Nullable RequestCoordinator parentCoordinator,
       TransitionOptions<?, ? super TranscodeType> transitionOptions,
       Priority priority,
       int overrideWidth,
       int overrideHeight,
       RequestOptions requestOptions) {
   
     // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
     ErrorRequestCoordinator errorRequestCoordinator = null;
     if (errorBuilder != null) {
       //创建errorRequestCoordinator（异常处理对象）
       errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
       parentCoordinator = errorRequestCoordinator;
     }
   
     //递归建立缩略图请求
     Request mainRequest =
       //分析1：
         buildThumbnailRequestRecursive(
             target,
             targetListener,
             parentCoordinator,
             transitionOptions,
             priority,
             overrideWidth,
             overrideHeight,
             requestOptions);
   
     if (errorRequestCoordinator == null) {
       return mainRequest;
     }
   
     int errorOverrideWidth = errorBuilder.requestOptions.getOverrideWidth();
     int errorOverrideHeight = errorBuilder.requestOptions.getOverrideHeight();
     if (Util.isValidDimensions(overrideWidth, overrideHeight)
         && !errorBuilder.requestOptions.isValidOverride()) {
       errorOverrideWidth = requestOptions.getOverrideWidth();
       errorOverrideHeight = requestOptions.getOverrideHeight();
     }
   
     Request errorRequest = errorBuilder.buildRequestRecursive(
         target,
         targetListener,
         errorRequestCoordinator,
         errorBuilder.transitionOptions,
         errorBuilder.requestOptions.getPriority(),
         errorOverrideWidth,
         errorOverrideHeight,
         errorBuilder.requestOptions);
     errorRequestCoordinator.setRequests(mainRequest, errorRequest);
     return errorRequestCoordinator;
   }
   ```

   #### 分析1： buildThumbnailRequestRecursive()

   ```java
   private Request buildThumbnailRequestRecursive(
       Target<TranscodeType> target,
       RequestListener<TranscodeType> targetListener,
       @Nullable RequestCoordinator parentCoordinator,
       TransitionOptions<?, ? super TranscodeType> transitionOptions,
       Priority priority,
       int overrideWidth,
       int overrideHeight,
       RequestOptions requestOptions) {
     if (thumbnailBuilder != null) {
       // Recursive case: contains a potentially recursive thumbnail request builder.
       if (isThumbnailBuilt) {
         throw new IllegalStateException("You cannot use a request as both the main request and a "
             + "thumbnail, consider using clone() on the request(s) passed to thumbnail()");
       }
   
       TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions =
           thumbnailBuilder.transitionOptions;
   
       // Apply our transition by default to thumbnail requests but avoid overriding custom options
       // that may have been applied on the thumbnail request explicitly.
       if (thumbnailBuilder.isDefaultTransitionOptionsSet) {
         thumbTransitionOptions = transitionOptions;
       }
   
       Priority thumbPriority = thumbnailBuilder.requestOptions.isPrioritySet()
           ? thumbnailBuilder.requestOptions.getPriority() : getThumbnailPriority(priority);
   
       int thumbOverrideWidth = thumbnailBuilder.requestOptions.getOverrideWidth();
       int thumbOverrideHeight = thumbnailBuilder.requestOptions.getOverrideHeight();
       if (Util.isValidDimensions(overrideWidth, overrideHeight)
           && !thumbnailBuilder.requestOptions.isValidOverride()) {
         thumbOverrideWidth = requestOptions.getOverrideWidth();
         thumbOverrideHeight = requestOptions.getOverrideHeight();
       }
   
       //coordinator是一个协调器，用于协调两个单独的{@link Request}同时加载图像的缩略图和图像的完整尺寸图。
       ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
       //获取一个正常请求对象
       Request fullRequest =
           obtainRequest(
               target,
               targetListener,
               requestOptions,
               coordinator,
               transitionOptions,
               priority,
               overrideWidth,
               overrideHeight);
       isThumbnailBuilt = true;
       // Recursively generate thumbnail requests.
       //递归建议一个缩略图请求对象
       Request thumbRequest =
           thumbnailBuilder.buildRequestRecursive(
               target,
               targetListener,
               coordinator,
               thumbTransitionOptions,
               thumbPriority,
               thumbOverrideWidth,
               thumbOverrideHeight,
               thumbnailBuilder.requestOptions);
       isThumbnailBuilt = false;
       // 能够同时加载缩略图和正常的图的请求
       coordinator.setRequests(fullRequest, thumbRequest);
       return coordinator;
     } else if (thumbSizeMultiplier != null) {
       // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
       // 当设置了缩略的比例thumbSizeMultiplier(0 ~  1)时，
       // 不需要递归建立缩略图请求
       ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
       Request fullRequest =
           obtainRequest(
               target,
               targetListener,
               requestOptions,
               coordinator,
               transitionOptions,
               priority,
               overrideWidth,
               overrideHeight);
       RequestOptions thumbnailOptions = requestOptions.clone()
           .sizeMultiplier(thumbSizeMultiplier);
   
       Request thumbnailRequest =
           obtainRequest(
               target,
               targetListener,
               thumbnailOptions,
               coordinator,
               transitionOptions,
               getThumbnailPriority(priority),
               overrideWidth,
               overrideHeight);
   
       coordinator.setRequests(fullRequest, thumbnailRequest);
       return coordinator;
     } else {
       // Base case: no thumbnail.
       // 没有缩略图请求时，直接获取一个正常图请求
       return obtainRequest(
           target,
           targetListener,
           requestOptions,
           parentCoordinator,
           transitionOptions,
           priority,
           overrideWidth,
           overrideHeight);
     }
     
     private Request obtainRequest(
         Target<TranscodeType> target,
         RequestListener<TranscodeType> targetListener,
         RequestOptions requestOptions,
         RequestCoordinator requestCoordinator,
         TransitionOptions<?, ? super TranscodeType> transitionOptions,
         Priority priority,
         int overrideWidth,
         int overrideHeight) {
       return SingleRequest.obtain(
           context,
           glideContext,
           model,
           transcodeClass,
           requestOptions,
           overrideWidth,
           overrideHeight,
           priority,
           target,
           targetListener,
           requestListeners,
           requestCoordinator,
           glideContext.getEngine(),
           transitionOptions.getTransitionFactory());
     }
   }
   ```

   buildRequest()-->buildRequestRecursive()--> buildThumbnailRequestRecursive()

   这个过程里大部分是处理缩略图的，我们主要看一下最后一行的obtainRequest(...)方法，这个方法需要很多参数，比如requestOptions这个参数里有我们设置的placeholderId。diskCacheStrategy等等。

   以上into()方法总的过程就是在buildRequest()方法里建立了请求，且最多可同时进行缩略图和正常图的请求，into()方法的最后，调用了requestManager.track(target, request)方法。

   接着看看**RequestManager.track(target, request)**里面做了什么。

   ```java
   void track(@NonNull Target<?> target, @NonNull Request request) {
     // 加入一个target目标集合(Set)
     targetTracker.track(target);
     requestTracker.runRequest(request);
   }
   ```

   **RequestTracker.runRequest(@NonNull Request request)**

   这里有一个简单的逻辑判断，就是先判断Glide当前是不是处理暂停状态，如果不是暂停状态就调用Request的begin()方法来执行Request；否则的话就先清空请求，将Request添加到待执行的延迟请求队列里面，等暂停状态解除了之后再执行。

   ```java
   /**
    * Starts tracking the given request.
    */
   public void runRequest(@NonNull Request request) {
     requests.add(request);
     if (!isPaused) {
       //如果不是暂停状态，则开始请求
       request.begin();
     } else {
       request.clear();
       if (Log.isLoggable(TAG, Log.VERBOSE)) {
         Log.v(TAG, "Paused, delaying request");
       }
       
       pendingRequests.add(request);
     }
   }
   
   ```

   接下来再看**SingleRequest.begin()方法**

   ```java
   @Override
   public void begin() {
     assertNotCallingCallbacks();
     stateVerifier.throwIfRecycled();
     startTime = LogTime.getLogTime();
     //model(url)为空，则回调加载失败
     if (model == null) {
       ...
       onLoadFailed(new GlideException("Received null model"), logLevel);
       return;
     }
   
     if (status == Status.RUNNING) {
       throw new IllegalArgumentException("Cannot restart a running request");
     }
   
     // If we're restarted after we're complete (usually via something like a notifyDataSetChanged
     // that starts an identical request into the same Target or View), we can simply use the
     // resource and size we retrieved the last time around and skip obtaining a new size, starting a
     // new load etc. This does mean that users who want to restart a load because they expect that
     // the view size has changed will need to explicitly clear the View or Target before starting
     // the new load.
     if (status == Status.COMPLETE) {
       onResourceReady(resource, DataSource.MEMORY_CACHE);
       return;
     }
   
     // Restarts for requests that are neither complete nor running can be treated as new requests
     // and can run again from the beginning.
   
     status = Status.WAITING_FOR_SIZE;
     if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
       //当调用RequestOptions.override(int width, int height)设置图片为固定的宽高时，直接执行下面的onSizeReady。
       //并且真正的请求就在onSizeReady方法里
       onSizeReady(overrideWidth, overrideHeight);
     } else {
       // 根据imageView的宽高算出图片的宽高，这个方法最终也会调用onSizeReady
       target.getSize(this);
     }
   
     if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
         && canNotifyStatusChanged()) {
       //预先加载设置的缩略图
       target.onLoadStarted(getPlaceholderDrawable());
     }
     if (IS_VERBOSE_LOGGABLE) {
       logV("finished run method in " + LogTime.getElapsedMillis(startTime));
     }
   }
   
   ```

   RequestTracker.runRequest(...) -> SingleRequest.begin() -> onSizeReady()至此开始了真正的请求。 

   

   接下来看**SingleRequest.onSizeReady(...)**方法

   ```java
   /**
    * A callback method that should never be invoked directly.
    */
   @Override
   public void onSizeReady(int width, int height) {
     stateVerifier.throwIfRecycled();
     if (IS_VERBOSE_LOGGABLE) {
       logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
     }
     if (status != Status.WAITING_FOR_SIZE) {
       return;
     }
     status = Status.RUNNING;
   
     float sizeMultiplier = requestOptions.getSizeMultiplier();
     this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
     this.height = maybeApplySizeMultiplier(height, sizeMultiplier);
   
     if (IS_VERBOSE_LOGGABLE) {
       logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
     }
     // 根据给定的配置进行加载，engine是一个负责加载、管理活跃和缓存资源的引擎类
     loadStatus = engine.load(
         glideContext,
         model,
         requestOptions.getSignature(),
         this.width,
         this.height,
         requestOptions.getResourceClass(),
         transcodeClass,
         priority,
         requestOptions.getDiskCacheStrategy(),
         requestOptions.getTransformations(),
         requestOptions.isTransformationRequired(),
         requestOptions.isScaleOnlyOrNoTransform(),
         requestOptions.getOptions(),
         requestOptions.isMemoryCacheable(),
         requestOptions.getUseUnlimitedSourceGeneratorsPool(),
         requestOptions.getUseAnimationPool(),
         requestOptions.getOnlyRetrieveFromCache(),
         this);
   
     ...
   }
   
   ```

   该方法内部调用engine.load()开始了真正的请求。 

   

   下面看**Engine.load()**方法

   这里说一点前置的知识点，Glide的缓存有内存、磁盘、网络三级；内存缓存又可以分为活动缓存、内存缓存两级。

   

   **Glide之内存缓存（LRU算法）:**

   - 活动缓存：给正在使用的资源存储的，弱引用。
   - 内存缓存：为第二次缓存服务，LRU算法。 
   - LRU算法：最近没有使用的元素，会自动被移除掉
   - LruCache v4：利用LinkedHashMap<K, V>，LinkedHashMap: true==拥有访问排序的功能 (最少使用元素算法-LRU算法)

   

    **Glide之磁盘缓存** 

   - 保存时长比较长：保存在本地磁盘 文件的形式存储 （不再是保存在运行内存中，而是磁盘中） 
   - LRU算法: 最近没有使用的元素，会自动被移除掉
   - LruCahce -- Android中提供了 V4
   - DiskLruCache --- Android中没有提供了 --> DiskLruCache
   - DiskLruCache：回收方式：LRU算法， 访问排序
   - DiskLruCache: 面向磁盘文件保存

   ```java
   public class Engine implements EngineJobListener,MemoryCache.ResourceRemovedListener,
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
   		//1. 先获取活动缓存，如果有的话，回调onResourceReady(...)并返回。
       EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
       if (active != null) {
         cb.onResourceReady(active, DataSource.MEMORY_CACHE);
         if (VERBOSE_IS_LOGGABLE) {
           logWithTimeAndKey("Loaded resource from active resources", startTime, key);
         }
         return null;
       }
   		//2. 如果没有活动缓存，再从内存中找，如果有的话， put到Map<Key, ResourceWeakReference>	(内部维护的弱引用缓存map)
       //并且回调 onResourceReady(...)并返回。 
       EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
       if (cached != null) {
         cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
         if (VERBOSE_IS_LOGGABLE) {
           logWithTimeAndKey("Loaded resource from cache", startTime, key);
         }
         return null;
       }
   
       EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
       if (current != null) {
         current.addCallback(cb);
         if (VERBOSE_IS_LOGGABLE) {
           logWithTimeAndKey("Added to existing load", startTime, key);
         }
         return new LoadStatus(cb, current);
       }
   		// 如果内存中没有，则创建engineJob（decodejob的回调类，管理下载过程以及状态）
       EngineJob<R> engineJob =
           engineJobFactory.build(
               key,
               isMemoryCacheable,
               useUnlimitedSourceExecutorPool,
               useAnimationPool,
               onlyRetrieveFromCache);
   		// 创建解析工作对象
       DecodeJob<R> decodeJob =
           decodeJobFactory.build(
               glideContext,
               model,
               key,
               signature,
               width,
               height,
               resourceClass,
               transcodeClass,
               priority,
               diskCacheStrategy,
               transformations,
               isTransformationRequired,
               isScaleOnlyOrNoTransform,
               onlyRetrieveFromCache,
               options,
               engineJob);
   
       // 放在Jobs内部维护的HashMap中
       jobs.put(key, engineJob);
   		// 注册ResourceCallback接口
       engineJob.addCallback(cb);
       // 内部开启线程去请求
       engineJob.start(decodeJob);
   
       if (VERBOSE_IS_LOGGABLE) {
         logWithTimeAndKey("Started new load", startTime, key);
       }
       return new LoadStatus(cb, engineJob);
       }
   }      
   
   ```

   **engineJob.start(decodeJob)**方法

   ```java
   class EngineJob<R> implements DecodeJob.Callback<R>,Poolable{
     public void start(DecodeJob<R> decodeJob) {
         this.decodeJob = decodeJob;
       	//根据不同的情况返回下面四种不同的线程池
         //GlideExecutor diskCacheExecutor;
   		  //GlideExecutor sourceExecutor;
   		  //GlideExecutor sourceUnlimitedExecutor;
   		  //GlideExecutor animationExecutor;
         GlideExecutor executor = decodeJob.willDecodeFromCache()
             ? diskCacheExecutor
             : getActiveSourceExecutor();
         executor.execute(decodeJob);
     }    
   }
   
   ```

   EngineJob是一个通过为图片的加载添加和移除callbacks来管理图片加载，并在加载完成后通知callbacks的类。

   它的主要作用就是用来开启线程的，为后面的异步加载图片做准备。DecodeJob对象，从名字上来看，它好像是用来对图片进行解码的，但实际上它的任务十分繁重，Engine.load()最后调用了EngineJob的start()方法来运行DecodeJob对象，这实际上就是让DecodeJob的run()方法在子线程当中执行了。 

   DecodeJob.run()方法又调用了runWrapped()，我们来看**runWrapped()**方法。

   ```java
   private void runWrapped() {
     switch (runReason) {
       case INITIALIZE:
         stage = getNextStage(Stage.INITIALIZE);
         //分析1：
         currentGenerator = getNextGenerator();
         //分析2：
         runGenerators();
         break;
       case SWITCH_TO_SOURCE_SERVICE:
         runGenerators();
         break;
       case DECODE_DATA:
       	//分析3：
         decodeFromRetrievedData();
         break;
       default:
         throw new IllegalStateException("Unrecognized run reason: " + runReason);
     }
   }
   
   ```

   分析1：完整情况下，会异步依次生成这里的ResourceCacheGenerator、DataCacheGenerator和SourceGenerator对象，并在之后执行其中的startNext()

   ```
   private DataFetcherGenerator getNextGenerator() {
     switch (stage) {
       case RESOURCE_CACHE:
         return new ResourceCacheGenerator(decodeHelper, this);
       case DATA_CACHE:
         return new DataCacheGenerator(decodeHelper, this);
       case SOURCE:
         return new SourceGenerator(decodeHelper, this);
       case FINISHED:
         return null;
       default:
         throw new IllegalStateException("Unrecognized stage: " + stage);
     }
   }
   
   ```

   分析2： runGenerators()里面调用了**startNext()**。

   startNext()是接口类DataFetcherGenerator里的抽象方法，在SourceGenerator、DataCacheGenerator、ResourceCacheGenerator分别重写了startNext()，这三个类分别对应着图片的原始数据、缓存文件里原始未修改的数据、缓存文件里经过采样\变换的数据。

   下面我们看一下从网络加载图片时，也就是**SourceGenerator.startNext()**方法

   根据磁盘缓存策略，先将源数据写入磁盘，然后从缓存文件中加载而不是直接返回。

   ```java
   @Override
   public boolean startNext() {
     //dataToCache数据不为空的话缓存到硬盘
     if (dataToCache != null) {
       Object data = dataToCache;
       dataToCache = null;
       cacheData(data);
     }
   
     if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
       return true;
     }
     sourceCacheGenerator = null;
   
     loadData = null;
     boolean started = false;
     while (!started && hasNextModelLoader()) {
       //分析1：
       //getLoadData()方法内部会在modelLoaders里面找到ModelLoder对象
       // （每个Generator对应一个ModelLoader），
       // 并使用modelLoader.buildLoadData方法返回一个loadData列表
       loadData = helper.getLoadData().get(loadDataListIndex++);
       if (loadData != null
           && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
           || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
         started = true;
         
         //分析2 
         //通过loadData对象的fetcher对象（其实现类为HttpUrlFetcher）的
         // loadData方法来获取图片数据
         loadData.fetcher.loadData(helper.getPriority(), this);
       }
     }
     return started;
   }
   
   ```

   分析1：

   ```java
   final class DecodeHelper<Transcode>{
     List<LoadData<?>> getLoadData() {
       if (!isLoadDataSet) {
         isLoadDataSet = true;
         loadData.clear();
         List<ModelLoader<Object, ?>> modelLoaders = glideContext.getRegistry().getModelLoaders(model);
         //noinspection ForLoopReplaceableByForEach to improve perf
         for (int i = 0, size = modelLoaders.size(); i < size; i++) {
           ModelLoader<Object, ?> modelLoader = modelLoaders.get(i);
           // 注意：这里最终是通过HttpGlideUrlLoader的buildLoadData获取到实际的loadData对象
           LoadData<?> current =modelLoader.buildLoadData(model, width, height, options);
           if (current != null) {
             loadData.add(current);
           }
         }
       }
       return loadData;
     }
   } 
   
   ```

   HttpGlideUrlLoader.buildLoadData(...)

   ```java
   @Override
   public LoadData<InputStream> buildLoadData(@NonNull GlideUrl model, int width, int height,
       @NonNull Options options) {
     // GlideUrls memoize parsed URLs so caching them saves a few object instantiations and time
     // spent parsing urls.
     //GlideUrls会记住已解析的URL，因此对它们进行缓存可以节省一些对象实例化和解析URL所花费的时间。
     GlideUrl url = model;
     if (modelCache != null) {
       url = modelCache.get(model, 0, 0);
       if (url == null) {
         //分析1.1：
         modelCache.put(model, 0, 0, model);
         url = model;
       }
     }
     int timeout = options.get(TIMEOUT);
     //在这里创建了一个DataFetcher的实现类HttpUrlFetcher
     return new LoadData<>(url, new HttpUrlFetcher(url, timeout));
   }
   
   ```

   分析1.1：modelCache.put(...)

   ```java
   /**
    * Add a value.
    *
    * @param model  The model.
    * @param width  The width in pixels of the view the image is being loaded into.
    * @param height The height in pixels of the view the image is being loaded into.
    * @param value  The value to store.
    */
   public void put(A model, int width, int height, B value) {
     ModelKey<A> key = ModelKey.get(model, width, height);
     // 最终是通过LruCache来缓存对应的值，key是一个ModelKey对象（由model、width、height三个属性组成）
     cache.put(key, value);
   }
   
   ```

   从这里的分析，我们明白了HttpUrlFetcher实际上就是最终的请求执行者，而且，我们知道了Glide会使用LruCache来对解析后的url来进行缓存，以便后续可以省去解析url的时间。

   分析2：HttpUrlFetcher.loadData(...)

   ```java
   @Override
   public void loadData(@NonNull Priority priority,
       @NonNull DataCallback<? super InputStream> callback) {
     long startTime = LogTime.getLogTime();
     try {
       //分析1 loadDataWithRedirects内部是通过HttpURLConnection网络请求数据
       InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
       // 请求成功回调onDataReady()
       callback.onDataReady(result);
     } catch (IOException e) {
       if (Log.isLoggable(TAG, Log.DEBUG)) {
         Log.d(TAG, "Failed to load data for url", e);
       }
       callback.onLoadFailed(e);
     } finally {
       if (Log.isLoggable(TAG, Log.VERBOSE)) {
         Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
       }
     }
   }
   
   ```

   分析1：HttpUrlFetcher.loadDataWithRedirects(...) 

   ```java
   private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl,
       Map<String, String> headers) throws IOException {
     if (redirects >= MAXIMUM_REDIRECTS) {
       throw new HttpException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
     } else {
       // Comparing the URLs using .equals performs additional network I/O and is generally broken.
       // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
       try {
         if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
           throw new HttpException("In re-direct loop");
   
         }
       } catch (URISyntaxException e) {
         // Do nothing, this is best effort.
       }
     }
   
     urlConnection = connectionFactory.build(url);
     for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
       urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
     }
     urlConnection.setConnectTimeout(timeout);
     urlConnection.setReadTimeout(timeout);
     urlConnection.setUseCaches(false);
     urlConnection.setDoInput(true);
   
     // Stop the urlConnection instance of HttpUrlConnection from following redirects so that
     // redirects will be handled by recursive calls to this method, loadDataWithRedirects.
     urlConnection.setInstanceFollowRedirects(false);
   
     // Connect explicitly to avoid errors in decoders if connection fails.
     urlConnection.connect();
     // Set the stream so that it's closed in cleanup to avoid resource leaks. See #2352.
     stream = urlConnection.getInputStream();
     if (isCancelled) {
       return null;
     }
     final int statusCode = urlConnection.getResponseCode();
     //2xx的状态码，代表请求成功
     if (isHttpOk(statusCode)) {
       //从urlConnection中获取资源流
       return getStreamForSuccessfulRequest(urlConnection);
     } else if (isHttpRedirect(statusCode)) {
       //重定向请求
       String redirectUrlString = urlConnection.getHeaderField("Location");
       if (TextUtils.isEmpty(redirectUrlString)) {
         throw new HttpException("Received empty or null redirect url");
       }
       URL redirectUrl = new URL(url, redirectUrlString);
       // Closing the stream specifically is required to avoid leaking ResponseBodys in addition
       // to disconnecting the url connection below. See #2352.
       cleanup();
       return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
     } else if (statusCode == INVALID_STATUS_CODE) {
       throw new HttpException(statusCode);
     } else {
       throw new HttpException(urlConnection.getResponseMessage(), statusCode);
     }
   }
   
   private InputStream getStreamForSuccessfulRequest(HttpURLConnection urlConnection)
         throws IOException {
       if (TextUtils.isEmpty(urlConnection.getContentEncoding())) {
         int contentLength = urlConnection.getContentLength();
         stream = ContentLengthInputStream.obtain(urlConnection.getInputStream(), contentLength);
       } else {
         if (Log.isLoggable(TAG, Log.DEBUG)) {
           Log.d(TAG, "Got non empty content encoding: " + urlConnection.getContentEncoding());
         }
         stream = urlConnection.getInputStream();
       }
       return stream;
   }
   
   ```

   HttpUrlFetcher.loadData(...)方法主要是通过调用loadDataWithRedirects(...)方法，使用原生的HttpURLConnection进行网络请求后，调用getStreamForSuccessfulRequest()方法获取最终的图片流。

6. 现在我们回到DecodeJob类中的run() ->runWrapped() 

   下面我们再次贴出runWrapped()方法的代码

   ```java
   private void runWrapped() {
     switch (runReason) {
       case INITIALIZE:
         stage = getNextStage(Stage.INITIALIZE);
         currentGenerator = getNextGenerator();
         runGenerators();
         break;
       case SWITCH_TO_SOURCE_SERVICE:
         runGenerators();
         break;
       case DECODE_DATA:
         // 分析1：
         decodeFromRetrievedData();
         break;
       default:
         throw new IllegalStateException("Unrecognized run reason: " + runReason);
     }
   }
   
   ```

   分析1：我们在刚才获取到图片的流后，还需要**decodeFromRetrievedData()**方法对流进行处理以得到我们想要的资源。

   ```java
   class DecodeJob<R> implements DataFetcherGenerator.FetcherReadyCallback,
       Runnable,
       Comparable<DecodeJob<?>>,
       Poolable {
   private void decodeFromRetrievedData() {
     if (Log.isLoggable(TAG, Log.VERBOSE)) {
       logWithTimeAndKey("Retrieved data", startFetchTime,
           "data: " + currentData
               + ", cache key: " + currentSourceKey
               + ", fetcher: " + currentFetcher);
     }
     Resource<R> resource = null;
     try {
       //从数据中解码得到资源
       resource = decodeFromData(currentFetcher, currentData, currentDataSource);
     } catch (GlideException e) {
       e.setLoggingDetails(currentAttemptingKey, currentDataSource);
       throwables.add(e);
     }
     if (resource != null) {
       // 编码和发布最终得到的Resource<Bitmap>对象
       notifyEncodeAndRelease(resource, currentDataSource);
     } else {
       runGenerators();
     }
   }
   
   
    private <Data> Resource<R> decodeFromData(DataFetcher<?> fetcher, Data data,
         DataSource dataSource) throws GlideException {
       try {
         if (data == null) {
           return null;
         }
         long startTime = LogTime.getLogTime();
         //执行解码的方法
         Resource<R> result = decodeFromFetcher(data, dataSource);
         if (Log.isLoggable(TAG, Log.VERBOSE)) {
           logWithTimeAndKey("Decoded result " + result, startTime);
         }
         return result;
       } finally {
         fetcher.cleanup();
       }
     }
         
     @SuppressWarnings("unchecked")
     private <Data> Resource<R> decodeFromFetcher(Data data, DataSource dataSource)
         throws GlideException {
       LoadPath<Data, ?, R> path = decodeHelper.getLoadPath((Class<Data>) data.getClass());
       //将解码任务分发给LoadPath
       return runLoadPath(data, dataSource, path);
     }
         
     private <Data, ResourceType> Resource<R> runLoadPath(Data data, DataSource dataSource,
           LoadPath<Data, ResourceType, R> path) throws GlideException {
         Options options = getOptionsWithHardwareConfig(dataSource);
       	// 将数据进一步包装
         DataRewinder<Data> rewinder = glideContext.getRegistry().getRewinder(data);
         try {
           // ResourceType in DecodeCallback below is required for compilation to work with gradle.
           //分析1：
           //将解码任务分发给LoadPath类里的load(...)方法
           return path.load(
               rewinder, options, width, height, new DecodeCallback<ResourceType>(dataSource));
         } finally {
           rewinder.cleanup();
         }
       }    
   }      
   
   ```

   分析1：**LoadPath.load(...)** 

   ```java
   public class LoadPath<Data, ResourceType, Transcode> {
    
     public Resource<Transcode> load(DataRewinder<Data> rewinder, @NonNull Options options, int width,
         int height, DecodePath.DecodeCallback<ResourceType> decodeCallback) throws GlideException {
       List<Throwable> throwables = Preconditions.checkNotNull(listPool.acquire());
       try {
         //分析2：
         return loadWithExceptionList(rewinder, options, width, height, decodeCallback, throwables);
       } finally {
         listPool.release(throwables);
       }
     } 
     
     private Resource<Transcode> loadWithExceptionList(DataRewinder<Data> rewinder,
         @NonNull Options options,
         int width, int height, DecodePath.DecodeCallback<ResourceType> decodeCallback,
         List<Throwable> exceptions) throws GlideException {
       Resource<Transcode> result = null;
       //noinspection ForLoopReplaceableByForEach to improve perf
       for (int i = 0, size = decodePaths.size(); i < size; i++) {
         DecodePath<Data, ResourceType, Transcode> path = decodePaths.get(i);
         try {
           //分析3
           //将解码任务又进一步分发给DecodePath的decode方法去解码
           result = path.decode(rewinder, width, height, options, decodeCallback);
         } catch (GlideException e) {
           exceptions.add(e);
         }
         if (result != null) {
           break;
         }
       }
   
       if (result == null) {
         throw new GlideException(failureMessage, new ArrayList<>(exceptions));
       }
   
       return result;
     }
   }  
   
   ```

   分析3：**DecodePath.decode(...)**

   ```java
   public class DecodePath<DataType, ResourceType, Transcode> {
     public Resource<Transcode> decode(DataRewinder<DataType> rewinder, int width, int height,
           @NonNull Options options, DecodeCallback<ResourceType> callback) throws GlideException {
       
       	//继续调用DecodePath的decodeResource方法去解析出数据
         Resource<ResourceType> decoded = decodeResource(rewinder, width, height, options);
         Resource<ResourceType> transformed = callback.onResourceDecoded(decoded);
         return transcoder.transcode(transformed, options);
      }
     
     @NonNull
     private Resource<ResourceType> decodeResource(DataRewinder<DataType> rewinder, int width,
         int height, @NonNull Options options) throws GlideException {
       List<Throwable> exceptions = Preconditions.checkNotNull(listPool.acquire());
       try {
         //分析4：
         return decodeResourceWithList(rewinder, width, height, options, exceptions);
       } finally {
         listPool.release(exceptions);
       }
     }
     
     private Resource<ResourceType> decodeResourceWithList(DataRewinder<DataType> rewinder, int width,
         int height, @NonNull Options options, List<Throwable> exceptions) throws GlideException {
       Resource<ResourceType> result = null;
       //noinspection ForLoopReplaceableByForEach to improve perf
       for (int i = 0, size = decoders.size(); i < size; i++) {
         ResourceDecoder<DataType, ResourceType> decoder = decoders.get(i);
         try {
           DataType data = rewinder.rewindAndGet();
           if (decoder.handles(data, options)) {
             //获取包装的数据
             data = rewinder.rewindAndGet();
             // 根据data（DataType）和ResourceType的类型分发给不同的解码器Decoder
             result = decoder.decode(data, width, height, options);
           }
           // Some decoders throw unexpectedly. If they do, we shouldn't fail the entire load path, but
           // instead log and continue. See #2406 for an example.
         } catch (IOException | RuntimeException | OutOfMemoryError e) {
           if (Log.isLoggable(TAG, Log.VERBOSE)) {
             Log.v(TAG, "Failed to decode data for " + decoder, e);
           }
           exceptions.add(e);
         }
   
         if (result != null) {
           break;
         }
       }
   
       if (result == null) {
         throw new GlideException(failureMessage, new ArrayList<>(exceptions));
       }
       return result;
     }
   }
   
   ```

   可以看到，经过一连串的嵌套调用，最终执行到了decoder.decode()这行代码，decode是一个ResourceDecoder<DataType, ResourceType>资源解码器接口，根据不同的DataType和ResourceType它会有不同的实现类，这里的实现类是ByteBufferBitmapDecoder，接下来让我们来看看这个解码器内部的解码流程。

    **ByteBufferBitmapDecoder.decode(...)** 

   ```java
   /**
    * Decodes {@link android.graphics.Bitmap Bitmaps} from {@link java.nio.ByteBuffer ByteBuffers}.
    */
   public class ByteBufferBitmapDecoder implements ResourceDecoder<ByteBuffer, Bitmap> {
     private final Downsampler downsampler;
   
     @Override
     public Resource<Bitmap> decode(@NonNull ByteBuffer source, int width, int height,
         @NonNull Options options)
         throws IOException {
       InputStream is = ByteBufferUtil.toStream(source);
       //核心代码
       return downsampler.decode(is, width, height, options);
     }
   }
   
   ```

   可以看到，最终是使用了一个downsampler，它是一个压缩器，主要是对流进行解码，压缩，圆角等处理。

     

   ```
   public final class Downsampler {
    public Resource<Bitmap> decode(InputStream is, int outWidth, int outHeight,
         Options options) throws IOException {
       return decode(is, outWidth, outHeight, options, EMPTY_CALLBACKS);
     }
     
     //根据给定的is解码成Bitmap
      public Resource<Bitmap> decode(InputStream is, int requestedWidth, int requestedHeight,
         Options options, DecodeCallbacks callbacks) throws IOException {
       Preconditions.checkArgument(is.markSupported(), "You must provide an InputStream that supports"
           + " mark()");
   
       byte[] bytesForOptions = byteArrayPool.get(ArrayPool.STANDARD_BUFFER_SIZE_BYTES, byte[].class);
       BitmapFactory.Options bitmapFactoryOptions = getDefaultOptions();
       bitmapFactoryOptions.inTempStorage = bytesForOptions;
   
       DecodeFormat decodeFormat = options.get(DECODE_FORMAT);
       DownsampleStrategy downsampleStrategy = options.get(DownsampleStrategy.OPTION);
       boolean fixBitmapToRequestedDimensions = options.get(FIX_BITMAP_SIZE_TO_REQUESTED_DIMENSIONS);
       boolean isHardwareConfigAllowed =
         options.get(ALLOW_HARDWARE_CONFIG) != null && options.get(ALLOW_HARDWARE_CONFIG);
   		//核心代码
       try {
         Bitmap result = decodeFromWrappedStreams(is, bitmapFactoryOptions,
             downsampleStrategy, decodeFormat, isHardwareConfigAllowed, requestedWidth,
             requestedHeight, fixBitmapToRequestedDimensions, callbacks);
         //将Bitmap包装成BitmapResource  
         return BitmapResource.obtain(result, bitmapPool);
       } finally {
         releaseOptions(bitmapFactoryOptions);
         byteArrayPool.put(bytesForOptions);
       }
     }
     
     private Bitmap decodeFromWrappedStreams(InputStream is,
         BitmapFactory.Options options, DownsampleStrategy downsampleStrategy,
         DecodeFormat decodeFormat, boolean isHardwareConfigAllowed, int requestedWidth,
         int requestedHeight, boolean fixBitmapToRequestedDimensions,
         DecodeCallbacks callbacks) throws IOException {
      
       //省略计算压缩比例等操作
        ... 
        
       Bitmap downsampled = decodeStream(is, options, callbacks, bitmapPool);
       callbacks.onDecodeComplete(bitmapPool, downsampled);
   
       //Bitmap旋转处理
       ...
       return rotated;
     }
     
     private static Bitmap decodeStream(InputStream is, BitmapFactory.Options options,
         DecodeCallbacks callbacks, BitmapPool bitmapPool) throws IOException {
         
   		...
   		final Bitmap result;
       TransformationUtils.getBitmapDrawableLock().lock();
       try {
       	//核心代码
         result = BitmapFactory.decodeStream(is, null, options);
       } catch (IllegalArgumentException e) {
         ...
       } finally {
         TransformationUtils.getBitmapDrawableLock().unlock();
       }
   
       if (options.inJustDecodeBounds) {
         is.reset();
   
       }
       return result;
     }
   }
   
   ```

   从以上源码流程我们知道，最后是在DownSampler的decodeStream()方法中使用了BitmapFactory.decodeStream()来得到Bitmap对象。然后，我们来分析下图片时如何显示的，我们回到DownSampler.decode()方法，看到关注点7，这里是将Bitmap包装成BitmapResource对象返回，通过内部的get方法可以得到Bitmap对象，再回到DecodeJob.run()方法，这是使用了notifyEncodeAndRelease()方法对Resource对象进行了发布。

    

   我们再回到上面分析的**HttpUrlFetcher.loadData(...)**方法

   ```java
   @Override
   public void loadData(@NonNull Priority priority,
       @NonNull DataCallback<? super InputStream> callback) {
     long startTime = LogTime.getLogTime();
     try {
       //loadDataWithRedirects内部是通过HttpURLConnection网络请求数据
       InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
       // 分析1：请求成功回调onDataReady()
       callback.onDataReady(result);
     } catch (IOException e) {
       if (Log.isLoggable(TAG, Log.DEBUG)) {
         Log.d(TAG, "Failed to load data for url", e);
       }
       callback.onLoadFailed(e);
     } finally {
       if (Log.isLoggable(TAG, Log.VERBOSE)) {
         Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime));
       }
     }
   }
   
   ```

   分析1: **onDataReady(result)**

   ```java
   class SourceGenerator implements DataFetcherGenerator,
       DataFetcher.DataCallback<Object>,
       DataFetcherGenerator.FetcherReadyCallback {
         
     @Override
     public void onDataReady(Object data) {
       DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
       if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
         dataToCache = data;
         // We might be being called back on someone else's thread. Before doing anything, we should
         // reschedule to get back onto Glide's thread.
         cb.reschedule();
       } else {
         // 分析2：通知回调加载已完成
         cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
             loadData.fetcher.getDataSource(), originalKey);
       }
     }
   }
   
   ```

   ​    

    分析2：**onDataFetcherReady(...)**

   ```java
   class DecodeJob<R> implements DataFetcherGenerator.FetcherReadyCallback,
       Runnable,
       Comparable<DecodeJob<?>>,
       Poolable {
   
     @Override
     public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
         DataSource dataSource, Key attemptedKey) {
       this.currentSourceKey = sourceKey;
       this.currentData = data;
       this.currentFetcher = fetcher;
       this.currentDataSource = dataSource;
       this.currentAttemptingKey = attemptedKey;
       if (Thread.currentThread() != currentThread) {
         //分析3：
         runReason = RunReason.DECODE_DATA;
         callback.reschedule(this);
       } else {
         GlideTrace.beginSection("DecodeJob.decodeFromRetrievedData");
         try {
           decodeFromRetrievedData();
         } finally {
           GlideTrace.endSection();
         }
       }
     }
         
     private void runWrapped() {
       switch (runReason) {
         case INITIALIZE:
           stage = getNextStage(Stage.INITIALIZE);
           currentGenerator = getNextGenerator();
           runGenerators();
           break;
         case SWITCH_TO_SOURCE_SERVICE:
           runGenerators();
           break;
         case DECODE_DATA:
           decodeFromRetrievedData();
           break;
         default:
           throw new IllegalStateException("Unrecognized run reason: " + runReason);
       }
     }
         
     private void decodeFromRetrievedData() {
       ...
       Resource<R> resource = null;
       try {
         resource = decodeFromData(currentFetcher, currentData, currentDataSource);
       } catch (GlideException e) {
         e.setLoggingDetails(currentAttemptingKey, currentDataSource);
         throwables.add(e);
       }
       if (resource != null) {
         //1
         notifyEncodeAndRelease(resource, currentDataSource);
       } else {
         runGenerators();
       }
   	}
         
     private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
       ...
   		//2
       notifyComplete(result, dataSource);
   
       ...
       // Call onEncodeComplete outside the finally block so that it's not called if the encode process
       // throws.
       onEncodeComplete();
     }
   
     private void notifyComplete(Resource<R> resource, DataSource dataSource) {
       setNotifiedOrThrow();
       //3
       callback.onResourceReady(resource, dataSource);
     }
   }  
   
   ```

   分析3：在这里我们看到了RunReason.DECODE_DATA，这个枚举代表处理返回的图片资源。然后调用了callback.reschedule(this);然后就会重新执行DecodeJob里的run()->runWrapped()，因为刚才runReason=DECODE_DATA，所以下一步会执行到decodeFromRetrievedData()方法。经过1.notifyEncodeAndRelease(...) ->2. notifyComplete(...) ->  3.callback.onResourceReady(...)。

   

   ```java
   class EngineJob<R> implements DecodeJob.Callback<R>,Poolable {
     //成功加载资源时调用。
     @Override
     public void onResourceReady(Resource<R> resource, DataSource dataSource) {
       this.resource = resource;
       this.dataSource = dataSource;
       //通过Handler发送消息MSG_COMPLETE到主线程
       MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
     }
     
     private static class MainThreadCallback implements Handler.Callback {
   
       @Synthetic
       @SuppressWarnings("WeakerAccess")
       MainThreadCallback() { }
   
       @Override
       public boolean handleMessage(Message message) {
         EngineJob<?> job = (EngineJob<?>) message.obj;
         switch (message.what) {
           case MSG_COMPLETE:
             //在主线程处理结果
             job.handleResultOnMainThread();
             break;
             
           ...
             
         }
         return true;
       }
   	}
     void handleResultOnMainThread() {
       stateVerifier.throwIfRecycled();
       ...
   		
       //noinspection ForLoopReplaceableByForEach to improve perf
       for (int i = 0, size = cbs.size(); i < size; i++) {
         ResourceCallback cb = cbs.get(i);
         if (!isInIgnoredCallbacks(cb)) {
           engineResource.acquire();
           //这里通过for循环调用了所有的ResourceCallback的方法。
           cb.onResourceReady(engineResource, dataSource);
         }
       }
       ...
     }
   }
   
   ```

   那这里的ResourceCallback是一个接口，那么是在哪里实现的这个接口呢？下面我们梳理一下

   首先看一下SingleRequest的onSizeReady()，这个方法里调用了engine.load(...,this)，并将this（SingleRequest）作为参数，又因为SingleRequest实现了ResourceCallback，所以，上面for循环里回调的onResourceReady(...)方法，就是SingleRequest重写的onResourceReady(...)方法。

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
   
         //ResourceCallback在这里进行了注册
         engineJob.addCallback(cb);
         engineJob.start(decodeJob);
   
         return new LoadStatus(cb, engineJob);
       }   
     }    
   
   ```

   

   ```java
   public final class SingleRequest<R> implements Request,
       SizeReadyCallback,
       ResourceCallback,
       FactoryPools.Poolable {
         
    @Override
     public void onSizeReady(int width, int height) {
       
       loadStatus = engine.load(
           glideContext,
           model,
           requestOptions.getSignature(),
           this.width,
           this.height,
           requestOptions.getResourceClass(),
           transcodeClass,
           priority,
           requestOptions.getDiskCacheStrategy(),
           requestOptions.getTransformations(),
           requestOptions.isTransformationRequired(),
           requestOptions.isScaleOnlyOrNoTransform(),
           requestOptions.getOptions(),
           requestOptions.isMemoryCacheable(),
           requestOptions.getUseUnlimitedSourceGeneratorsPool(),
           requestOptions.getUseAnimationPool(),
           requestOptions.getOnlyRetrieveFromCache(),
           this);//关键代码
     }     
         
    @Override
     public void onResourceReady(Resource<?> resource, DataSource dataSource) {
       ...
       //通过get方法获取Bitmap
       Object received = resource.get();
   		...
       onResourceReady((Resource<R>) resource, (R) received, dataSource);
     }
         
     private void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
       ...
       try {
   			...
         if (!anyListenerHandledUpdatingTarget) {
           Transition<? super R> animation =
               animationFactory.build(dataSource, isFirstResource);
           //核心代码
           target.onResourceReady(result, animation);
         }
       } finally {
         isCallingCallbacks = false;
       }
   
       notifyLoadSuccess();
     }
   }
   
   ```

      SingleRequest的onResourceReady(...)方法最后调用了target.onResourceReady(result, animation);这个target就是我们在最开始调用Glide.into(imageview)时Imageview的包装类BitmapImageViewTarget。但是在BitmapImageViewTarget这个类里并没有onResourceReady()方法，因为BitmapImageViewTarget继承了ImageViewTarget，我们在ImageViewTarget中找到了这个重写的onResourceReady()方法。 

   

   ```java
   public abstract class ImageViewTarget<Z> extends ViewTarget<ImageView, Z>
       implements Transition.ViewAdapter {
       
       @Override
       public void onResourceReady(@NonNull Z resource, @Nullable Transition<? super Z> transition) {
         if (transition == null || !transition.transition(resource, this)) {
           // 核心代码
           setResourceInternal(resource);
         } else {
           maybeUpdateAnimatable(resource);
         }
       }  
       
       private void setResourceInternal(@Nullable Z resource) {
         // Order matters here. Set the resource first to make sure that the Drawable has a valid and
         // non-null Callback before starting it.
         //这里调用了抽象方法setResource。
         setResource(resource);
         maybeUpdateAnimatable(resource);
     	}
     	
     	protected abstract void setResource(@Nullable Z resource);
   }
   
   ```

      **BitmapImageViewTarget重写了setResource(@Nullable Z resource)方法**

   ```java
   /**
    * A {@link com.bumptech.glide.request.target.Target} that can display an {@link
    * android.graphics.Bitmap} in an {@link android.widget.ImageView}.
    */
   public class BitmapImageViewTarget extends ImageViewTarget<Bitmap> {
     /**
      * Sets the {@link android.graphics.Bitmap} on the view using {@link
      * android.widget.ImageView#setImageBitmap(android.graphics.Bitmap)}.
      *
      * @param resource The bitmap to display.
      */
     @Override
     protected void setResource(Bitmap resource) {
       view.setImageBitmap(resource);
     }
   }
   
   ```

   至此，通过 view.setImageBitmap(resource)将Bitmap设置到了Imageview上，图片也就显示出来了。可见into()方法是极其复杂的。Glide整个主线流程到此分析结束。