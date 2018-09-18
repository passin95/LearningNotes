.
# Canvas

## 坐标系

在 Android 里，每个 View 都有一个自己的坐标系，彼此之间是不影响的。这个坐标系的原点是 View 左上角的那个点；水平方向是 x 轴，右正左负；竖直方向是 y 轴。

<div align="center"> <img src="../pictures//坐标系.png" width="400"/> </div><br>


## 绘制颜色

这类颜色填充方法一般用于在绘制之前设置底色，或者在绘制之后为界面设置半透明蒙版。

drawColor(@ColorInt int color)

drawRGB(int r, int g, int b)

drawARGB(int a, int r, int g, int b) 

## 绘制图形

### 圆

drawCircle(float centerX, float centerY, float radius, Paint paint) 

### 矩形

drawRect(float left, float top, float right, float bottom, Paint paint) 

### 点

drawPoint(float x, float y, Paint paint)

###
