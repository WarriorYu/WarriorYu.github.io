---
title: 移动Web(1)
categories:
  - 大前端
tags:
  - H5
date: 2020-08-05 11:30:59
---

- - - - - [1. 概念](#1-概念)
        - [2. 适配要求](#2-适配要求)
        - [3. 适配设置](#3-适配设置)
        - [4 . viewport的功能](#4-viewport的功能)
        - [5. 非主流的适配方案：](#5-非主流的适配方案)
        - [6. 不建议在移动端使用jquery](#6-不建议在移动端使用jquery)
        - [7. box-sizing](#7-box-sizing)
        - [8. 点击高亮](#8-点击高亮)
        - [9. 防止出现图片下间隙](#9-防止出现图片下间隙)
        - [10.移动端滑动事件：](#10移动端滑动事件)
        - [11.click在移动端有300ms的延时：](#11click在移动端有300ms的延时)
        - [12.区域滚动效果：](#12区域滚动效果)
        - [13.媒体查询](#13媒体查询)
        - [14.reset.css 和normalize.css区别](#14resetcss-和normalizecss区别)
        - [15.响应式布局](#15响应式布局)
        - [16.自定义字体图标](#16自定义字体图标)
        - [17.boostrap实现导航条](#17boostrap实现导航条)
        - [18.模板引擎](#18模板引擎)
        - [补充：](#补充)
        - [注意事项：](#注意事项)
        - [MarkDown快捷键：](#markdown快捷键)



##### 1. 概念

- 流式布局：就是百分比布局，非固定像素，内容向两侧填充，理解成流动的布局，称为流式布局
- 视觉窗口：viewport,是移动端特有。这是一个虚拟的区域，承载网页的。
- 承载关系：浏览器---->viewport---->网页

##### 2. 适配要求

- 网页宽度必须和浏览器保持一致
- 默认显示的缩放比例和PC端保持（缩放比例1.0）
- 不允许用户自行缩放网页
- 满足这些要求达到了适配，国际上通用的适配方案，标准的移动端适配方案。

##### 3. 适配设置

- 如果任何设置都没有，默认走的就是viewport的默认设置
- 去设置新的viewport设置,达到适配要求，<meta name="viewport"> 设置视口的标签 在head里面并且应该紧接着编码设置

##### 4 . viewport的功能

- width 可以设置宽度 (device-width 当前设备的宽度)
- height 可以设置高度
- initial-scale 可以设置默认的缩放比例
- user-scalable 可以设置是否允许用户自行缩放
- maximum-scale 可以设置最大缩放比例
- minimum-scale 可以设置最小缩放比例
  - 在<meta name="viewport" content="" > content="" 使用以上参数
    1. width=device-width 宽度一致比例是1.0
    2. initial-scale=1.0 宽度一致比例是1.0
    3. user-scalable=no 不允许用户自行缩放 （yes，no 1,0）
- 标准适配方案

```
<meta name="viewport" content="width=device-width,initial-scale=1.0,user-scalable=0">
```

快捷方式：meta:vp + tab

##### 5. 非主流的适配方案：

```
1.页面的真实尺寸会比在设备的上尺寸要大几倍
2.假设设备是iphone4 -> 320px -> 网页尺寸 640px
3.缩放操作，有2倍的  有3倍  和屏幕像素比有关系
4.什么是屏幕像素（物理像素，像素点） px(页面的尺寸单位)
5.物理像素 是设备显示屏的最小可视颗粒的大小   以前的手机（直板手机）
6.现在有 高清显示屏  视网膜屏  retina屏
7.显示的效果就提高了更细腻，但是在显示同等质量的图片的时候（模糊效果）

8.在屏幕像素比（一个px宽的屏幕能放几个物理像素）高的设备  图片（非矢量）显示会模糊
9.提高网页的清晰度  根据屏幕的像素比 来缩放网页
10.但是这样的适配方案成本非常高
11.一般的企业开发当中使用的还是标准化设置

在高清显示屏当中：图片可能会失真（模糊）
```

##### 6. 不建议在移动端使用jquery

```
1.可以使用jquery,但是不建议
2.jquery  做了很多桌面浏览器的兼容问题 特别是IE，但是移动端没有IE浏览器
3.主流的浏览器：谷歌 火狐（2016年停止了维护和更新） safari浏览器  百度  360 qq ...
4.特点：内核基本上都是  webkit  或者 blink  兼容  -webkit-
5.使用H5的api 或者说使用一个 叫做： zepto.js 的库（基于高版本浏览器开发）
```

##### 7. box-sizing

```
/*防止内容溢出  不出现滚动条  提供用户体验*/
box-sizing: border-box;

1. 移动端以流式布局为主
2. 百分比布局
3. 非固定像素布局
4. 无法准确计算容器的尺寸
```

##### 8. 点击高亮

```
/*轻击 轻触*/
-webkit-tap-highlight-color: red;
```

##### 9. 防止出现图片下间隙

 本质是有文字导致的，比如：agfy, g比a在格式上下沉，这样的文字和图片在一起，就会出现图片下间隙。

```
<style>
    body{
        /*font-size: 0px;*/
    }
    或
    img{
        display: block;
    }
    或
    img{
        vertical-align: middle;
    }
</style>
```

##### 10.移动端滑动事件：

​	touchstart touchmove touchend

##### 11.click在移动端有300ms的延时：

​	避免300ms的方案：

 1.tap事件

​	2.fastclick插件使用

##### 12.区域滚动效果：

​	插件 isScroll

##### 13.媒体查询

​	1、使用媒体查询能针对不同屏幕区间设置不同的布局和样式

​	2、怎么使用媒体查询：关于媒体查询 @media

​	3、语法： @media screen and (max-width: 768px) and (min-width: 320px){属性样式}

​	eg:

```
@media screen and (max-width: 768px) {
    /*1. 在超小屏设备的时候 768px以下      当前容器的宽度100%     背景蓝色*/
    .container{
        width: 100%;
        background: blue;
    }
}
@media screen and (min-width: 768px) and (max-width: 992px){
    /*2. 在小屏设备的时候   768px-992px    当前容器的宽度750px    背景绿色*/
    .container{
        width: 750px;
        background: green;
    }
}
@media screen and (min-width: 992px) and (max-width: 1200px){
    /*3. 在中屏设备的时候   992px-1200px   当前容器的宽度970px    背景红色*/
    .container{
        width: 970px;
        background: red;
    }
}
@media screen and (min-width: 1200px){
    /*4. 在大屏设备的时候   1200px以上      当前容器的宽度1170px   背景黄色*/
    .container{
        width: 1170px;
        background: yellow;
    }
}
```

##### 14.reset.css 和normalize.css区别

```
共同点：都是重置样式库，增强浏览器的表现一致性
不同点：都是增强浏览器的表现一致性，但是normalize不会重置已经一致的元素
```

##### 15.响应式布局

```
<!--响应式布局容器-->
<div class="container">
    <!--栅格系统：其实就是行和列的布局，网格状布局-->
    <!--行：row-->
    <!-- .container容器默认有15px的左右内间距  .row 填充父容器的15px的左右内间距   margin-left,margin-right -15px拉伸 -->
    <div class="row">
        <!--列：col-*-*  *不确定（参数） -->
        <!--
            col：列样式
            第一个*：屏幕的大小
            大屏设备     lg   大屏设备以上生效包含本身
            中屏设备     md   中屏设备以上生效包含本身
            小屏设备     sm   小屏设备以上生效包含本身
            超小屏设备   xs   超小屏设备以上生效包含本身
            <!--
                响应式容器：
                1. 在超小屏设备的时候 768px以下      当前容器的宽度100%     
                2. 在小屏设备的时候   768px-992px    当前容器的宽度750px   
                3. 在中屏设备的时候   992px-1200px   当前容器的宽度970px    
                4. 在大屏设备的时候   1200px以上      当前容器的宽度1170px 
                -->
                
            第二个*：每一行的分等份，默认分成12等份 ，数字代表的是占多少份
        -->
        <div class="col-xs-4"></div>
        <div class="col-xs-4"></div>
        <div class="col-xs-4"></div>
    </div>
</div>	

<!--
大屏设备    显示
中屏设备    隐藏
小屏设备    显示
超小屏设备  隐藏
visible-lg     大屏显示其他隐藏
visible-md
visible-sm
visible-xs
3.2版本以后  建议使用hidden
hidden-lg
hidden-md
hidden-sm
hidden-xs
-->
```

##### 16.自定义字体图标

```
/*自定义字体图标*/
/*1.通过@font-face定义自己的字体*/
@font-face {
    /*2.申明自己的字体名称*/
    font-family: 'wjs';
    /*3.引入字体文件（约束某一段字符代码什么图案）*/
    src: url(../fonts/MiFie-Web-Font.svg) format('svg'),
    url(../fonts/MiFie-Web-Font.eot) format('embedded-opentype'),
    url(../fonts/MiFie-Web-Font.ttf) format('truetype'),
    url(../fonts/MiFie-Web-Font.woff) format('woff');
}
```

##### 17.boostrap实现导航条

```
 <!--切换按钮-->
            <!--
            类名：collapsed  样式
            属性：
            data-toggle="collapse"  申明是什么组件=折叠组件
            data-target="#bs-example-navbar-collapse-1" 控制的目标元素=选择器
            其他：
            aria-expanded="false"  aria-* 代表提供给屏幕阅读器使用的（盲人阅读器）
            class="sr-only" screen read only  代表提供给屏幕阅读器使用的（盲人阅读器）
            -->
```

##### 18.模板引擎

```
<!--使用模版引擎-->
<script type="text/template" id="pointTemplate">
    <% for(var i = 0 ; i < list.length ; i ++){ %>
        <li data-target="#carousel-example-generic" data-slide-to="<%=i%>" class="					<%=i==0?'active':''%>"></li>
    <% } %>
</script>
<script type="text/template" id="imageTemplate">
    <% for(var i = 0 ; i < list.length ; i ++){ %>
    <div class="item <%=i==0?'active':''%>">
        <% if(isMobile){ %>
        	<a href="#" class="m_imgBox"><img src="<%=list[i].mUrl%>" alt=""></a>
        <% }else{ %>
        	<a href="#" class="pc_imgBox" style="background-image: 
        	    url(<%=list[i].pcUrl%>)"></a>
        <% } %>
    </div>
    <% } %>
</script>
```

##### 补充：

1. a.offsetWidth; 获取控件的宽度

2. document.querySelector('.class_name); 获取标签

3. box.querySelectorAll('li'); 获取li标签数组

4. imageBox.style.transition = 'all 0.2s';

   imageBox.style.webkitTransition = 'all 0.2s'; css3属性要做兼容

5. 浮动的元素因为脱离标准文档流没有高度，当设置touch事件的时候会无法响应事件，所以要清除浮动。

6. dom.addEventListener('touchmove',function(e){

   ​	var moveX = e.touches[0].clientX;

   })

7. 实现两栏，左侧一栏设定：width：100px，float:left ; 右侧一栏设置overflow:hidden。可实现右侧占满屏幕，从而实现两栏。

8. ```
   1、css代码
   /*  +,~选择器   + 紧邻的下一个兄弟元素  ~ 后面所有的兄弟元素*/
   .wjs_topBar > .container > .row > div {
       height: 40px;
       line-height: 40px;
       text-align: center;
   }
   
   .wjs_topBar > .container > .row > div ~ div {
       border-left: 1px solid #ccc;
   }
   
   2、html代码
   <header class="wjs_topBar hidden-sm hidden-xs">
       <div class="container">
           <div class="row">
               <div class="col-md-2"></div>
               <div class="col-md-5"></div>
               <div class="col-md-2"></div>
               <div class="col-md-3"></div>
           </div>
       </div>
   </header>
   ```

##### 注意事项：

1. js方法调用一定不能忘记加小括号“()”，因为不会报错，eg: e.prevertDefault();
2. tranform:translateX(1px);  能提高层级