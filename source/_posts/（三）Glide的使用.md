---
title: （三）Glide的使用
categories:
  - Android源码分析
tags:
  - Android第三方库源码分析
date: 2020-08-05 10:50:29
---

## Glide 记录

1. Android SDK 要求 

   **Min Sdk Version** - 使用 Glide 需要 min SDK 版本 API **14** (Ice Cream Sandwich) 或更高。

   **Compile Sdk Version** - Glide 必须使用 API **27** (Oreo MR1) 或更高版本的 SDK 来编译。

   **Support Library Version** - Glide 使用的支持库版本为 **27**。

   如果你需要使用不同的支持库版本，你需要在你的 `build.gradle` 文件里去从 Glide 的依赖中去除 `"com.android.support"`。例如，假如你想使用 v26 的支持库：

   ```
   dependencies {
     implementation ("com.github.bumptech.glide:glide:4.8.0") {
       exclude group: "com.android.support"
     }
     implementation "com.android.support:support-fragment:26.1.0"
   }
   ```

2. 多数情况下，使用Glide加载图片非常简单，一行代码足矣：

   ```
   Glide.with(fragment)
       .load(myUrl)
       .into(imageView);
   ```

   取消加载同样很简单：

   ```
   Glide.with(fragment).clear(imageView);
   ```

   尽管及时取消不必要的加载是很好的实践，但这并不是必须的操作。实际上，当 [`Glide.with()`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/Glide.html#with-android.app.Fragment-) 中传入的 Activity 或 Fragment 实例销毁时，Glide 会自动取消加载并回收资源。  

3. 在 ListView 和 RecyclerView 中的使用

   在 ListView 或 RecyclerView 中加载图片的代码和在单独的 View 中加载完全一样。Glide 已经自动处理了 View 的复用和请求的取消。对 url 进行 null 检验并不是必须的，如果 url 为 null，Glide 会清空 View 的内容，或者显示 [placeholder Drawable](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/request/RequestOptions.html#placeholder-int-) 或 [fallback Drawable](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/request/RequestOptions.html#fallback-int-) 的内容。

4. Glide允许用户指定三种不同类型的占位符，分别在三种不同场景使用：

   - [placeholder](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/request/RequestOptions.html#placeholder-int-)
   - [error](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/request/RequestOptions.html#error-int-)
   - [fallback](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/request/RequestOptions.html#fallback-int-)

5. 请求选项

   Glide中的大部分设置项都可以通过 [`RequestOptions`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/request/RequestOptions.html) 类和 [`apply()`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/RequestBuilder.html#apply-com.bumptech.glide.request.RequestOptions-) 方法来应用到程序中。

   使用 `request options` 可以实现（包括但不限于）：

   - 占位符(`Placeholders`)
   - 转换(`Transformations`)
   - 缓存策略(`Caching Strategies`)
   - 组件特有的设置项，例如编码质量，或`Bitmap`的解码配置等。 
