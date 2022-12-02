---
title: Android ImageView图片自适应
date: 2017-04-21 10:28:30
tags: Android
---
----------
## 前言
目中使用Picasso加载图片，结果发现后台给的图片太小，无法填充ImageView，在Picasso加载图片中加上fit()后图片直接不显示了，便想到前面图片太大的解决办法–给ImageView设置自适应，直接加上:`android:adjustViewBounds="true"`就行了。

## ImageView属性说明
```
android:adjustViewBounds
```
是否保持宽高比，是否调整ImageView的界限来保持图像纵横比不变。（可以用来网络上下载下来的图片自适应）
<!--more-->

XML定义里的`android:adjustViewBounds="true"`会将这个ImageView的scaleType设为fitCenter。不过这个fitCenter会被后面定义的scaleType属性覆盖（如果定义了的话），除非在Java代码里再次显示调用`setAdjustViewBounds(true)`。

它的效果还根据layout_width与layout_height的不同而不同：
1. layout_width和layout_height都是定值，那么设置adjustViewBounds是没有效果的，ImageView将是设置的宽高。
2. layout_width和layout_height都是wrap_content，那么设置adjustViewBounds也是没有意义的，用因为ImageView将会始终与图片拥有相同的宽高比。
3. 两者中一个是定值或者match_content，一个是wrap_content，比如layout_width=”200px” layout_height=”wrap_content”时，ImageView的宽将始终是200px，而高则分两种情况：

   - 图片的宽度小于200px时，layout_height将与图片的高相同，图片不会缩放，完整显示在ImageView中，ImageView高度与图片实际高度相同。图片没有占满ImageView，ImageView横向上有空白。
   - 图片的宽度大于或等于200px时，此时ImageView的宽高比与图片相同，因此ImageView的layout_height值为：200除以图片的宽高比。比如图片是500X500的，那么layout_height是200/1。图片将保持宽高比缩放，完整显示在ImageView中，并且完全占满ImageView。

```
android:scaleType
```
设置图片的填充方式。

ImageView的scaleType的属性有好几种，分别是matrix（默认）、center、centerCrop、centerInside、fitCenter、fitEnd、fitStart、fitXY.
- matrix: 不改变原图的大小，从ImageView的左上角开始绘制原图，原图超过ImageView的部分直接剪裁。
- center: 保持原图的大小，显示在ImageView的中心，原图超过ImageView的部分剪裁。
- centerCrop: 等比例放大原图，将原图显示在ImageView的中心，直到填满ImageView位置，超出部分剪裁。
- centerInside: 将图片的内容完整居中显示，当原图宽高小于或等于ImageView的宽高时，按原图大小居中显示；反之将原图等比例缩放至ImageView的宽高并居中显示。
- fitCenter: 按比例拉伸图片，拉伸后图片的宽度为ImageView的宽度，且显示在ImageView的中间。
- fitEnd: 按比例拉伸图片，拉伸后图片的宽度为ImageView的宽度，且显示在ImageView的下边。
- fitStart: 按比例拉伸图片，拉伸后图片的宽度为ImageView的宽度，且显示在ImageView的上边。
- fitXY: 拉伸图片（不按比例）以填充ImageView的宽高。



