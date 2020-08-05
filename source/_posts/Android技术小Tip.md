---
title: Android技术小Tip
categories:
  - Android
tags:
  - 技术小Tips
date: 2020-08-05 11:39:37
---

1. LinearLayout等空间在布局的根节点设置margin无效，padding有效。

2. CheckBox

   - 自定义样式使用`android:button="@drawable/selector_edit_activity_checkbox"`，但是paddingLeft设置的是图片与文字的间距，不是左间距，可以通过marginLeft设置左间距。
   - 如果只需要更改CheckBox的颜色，可以使用`android:buttonTint="@color/black"`,但是有个版本的限制条件：**Attribute buttonTint is only used in API level 21**
   - CheckBox 如果在代码中设置了 `cbxCheck.setOnClickListener(new View.OnClickListener(){}` 会导致没有点击后的涟漪效果

3. Imageview 的tint属性：**可以将图片的背景色改为想要的颜色**

   - 在xml文件中 `android:tint="@color/white"`

   - 在java代码中 `imagview.setColorFilter(color);`

   - 扩展

     How to apply the color. The standard mode is{@link PorterDuff.Mode#SRC_ATOP}

     `setColorFilter(int color, PorterDuff.Mode mode)`

4. 前后台接口里的字段类型一定一致，否则会出现不可预知的bug:

   - 如果后台字段是String类型，前台是long类型，如果后台传了null，可能会导致奔溃，如果后台是long类型，不会出现null,即使没有这个字段，前台也会有默认值，减少崩溃率。不要每个字段判空，这是最low的，会导致项目里全是if else.

5. 如果想下载的sdk不能通过代理下载，注意在gradle.properties文件里，把相关代码删除或者注释。

   报错信息：

   ```
   Unable to resolve dependency for ':app@debug/compileClasspath': Could not resolve com.mob:MobTools:+.
   
   Could not resolve com.mob:MobTools:+.
   Required by:
       project :app
    > Could not resolve com.mob:MobTools:+.
       > Failed to list versions for com.mob:MobTools.
             > Unable to load Maven meta-data from http://jcenter.bintray.com/com/mob/MobTools/maven-metadata.xml.
                      > Could not get resource 'http://jcenter.bintray.com/com/mob/MobTools/maven-metadata.xml'.
                                  > Could not GET 'http://jcenter.bintray.com/com/mob/MobTools/maven-metadata.xml'.
                                                 > Connect to 127.0.0.1:1080 [/127.0.0.1] failed: Connection refused (Connection refused)
                                                                   > Connection refused (Connection refused)
   ```

   解决方式：关闭代理

   ```
   android.enableAapt2=false
   #如果需要代理把这两行放开
   #systemProp.http.proxyHost=127.0.0.1
   #systemProp.http.proxyPort=1080
   org.gradle.jvmargs=-Xmx2048m -XX\:MaxPermSize\=512m -XX\:+HeapDumpOnOutOfMemoryError -Dfile.encoding\=UTF-8
   org.gradle.daemon=true
   org.gradle.configureondemand=true
   org.gradle.parallel=true
   android.useDeprecatedNdk=true
   ```

6. RecyclerView等列表控件，如果获取position的item时要注意有header和没有header的情况，比如：

   ```
   以BaseRecyclerViewAdapterHelper 这个库举例：
   
   //因为有addHeaderView,所以position的计算一定不能少了减去getHeaderLayoutCount()
     int position = mViewHolder.get(i).getLayoutPosition() - getHeaderLayoutCount();
   ```

7. Recyclerview:adapter.notifyDataSetChanged()类似的刷新方法，会引起已缓存的ViewHolder相应的onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position)方法的调用，因为复用，并不会把整个列表都刷新一遍

8. 代码压缩、混淆相关 ：参见：https://developer.android.com/studio/build/shrink-code?hl=zh-cn

9. TextView的自动适应文字大小属性：app:autoSizeTextType="uniform"兼容到API14，

   如果android:autoSizeTextType="uniform"不能兼容Android 8.0以下的手机。和android:singleLine不兼容，会显示“...”,可以使用android:maxLines=“1” 。width和height要写死，否则在Recyclerview中滑动的时候，可能会导致缩放不符合预期。

10. 抽象类A实现接口B，A可以实现B中的部分方法，也可以全部实现。

11. 音频详情问题：1.为什么PendingIntent的数据没有传过来。是序列化的问题，改为传递基本类型 2.为什么在initViews()调用finish方法，还会走到fillData();

12. 沉浸式状态栏，根布局"android:fitsSystemWindows"的影响要注意。