.
# Canvas

## 坐标系

在 Android 里，每个 View 都有一个自己的坐标系，彼此之间是不影响的。这个坐标系的原点是 View 左上角的那个点；水平方向是 x 轴，右正左负；竖直方向是 y 轴。

<div align="center"> <img src="../pictures//坐标系.png" width="400"/> </div><br>


## 绘制颜色

这类颜色填充方法一般用于在绘制之前设置底色，或者在绘制之后为界面设置半透明蒙版。

Canvas.drawColor(@ColorInt int color)

Canvas.drawRGB(int r, int g, int b)

Canvas.drawARGB(int a, int r, int g, int b) 

## 绘制图形

以下 RectF 皆可用矩形坐标（左上右下）代替。

### 圆

Canvas.drawCircle(float centerX, float centerY, float radius, Paint paint) 

### 矩形

Canvas.drawRect(RectF rect, Paint paint) 

### 点

画一个点

Canvas.drawPoint(float x, float y, Paint paint)

画多个点

Canvas.drawPoints(float[] pts, int offset, int count, Paint paint) 

```java
float[] points = {0, 0, 50, 50, 50, 100, 100, 50, 100, 100, 150, 50, 150, 100};  

canvas.drawPoints(points, 2 /* 跳过两个数，即前两个 0 */,  
          8 /* 一共绘制 8 个数（4 个点）*/, paint);
```

### 椭圆

Canvas.drawOval(RectF rect, Paint paint) 

### 线

由于直线不是封闭图形，所以 setStyle(style) 对直线没有影响

画一条线

Canvas.drawLine(float startX, float startY, float stopX, float stopY, Paint paint) 

画多条线（四个数一条线）

Canvas.drawLines(float[] pts, int offset, int count, Paint paint)

### 圆角矩形

Canvas.drawRoundRect(RectF rect, float rx, float ry, Paint paint) 

### 弧形或扇形

Canvas.drawArc(RectF rect, float startAngle, float sweepAngle, boolean useCenter, Paint paint)

startAngle 是弧形的起始角度（x 轴的正向，即正右的方向，是 0 度的位置；顺时针为正角度，逆时针为负角度），sweepAngle 是弧形划过的角度；useCenter 表示是否连接到圆心，如果不连接到圆心，就是弧形，如果连接到圆心，就是扇形。

### 自定义图形

drawPath(Path path, Paint paint) 

#### Path.Direction



#### Path.addXxx()

添加子图形一般，Xxx为绘图图形章节的图形，这里以addCircle举例。

addCircle(float x, float y, float radius, Direction dir) 

x, y, radius 这三个参数是圆的基本信息，最后一个参数 dir 是画圆的路径的方向。

path.AddCircle(x, y, radius, dir) + canvas.drawPath(path, paint) 和canvas.drawCircle(x, y, radius, paint) 的效果是一样的。

#### Path.xxxTo()

画直线或曲线。

##### 直线

lineTo(float x, float y) / rLineTo(float x, float y) 

```java
path.lineTo(100, 100); // 由当前位置 (0, 0) 向 (100, 100) 画一条直线  
path.rLineTo(100, 0); // 由当前位置 (100, 100) 向正右方 100 像素的位置画一条直线  
```

##### 二次贝塞尔曲线

quadTo(float x1, float y1, float x2, float y2) / rQuadTo(float dx1, float dy1, float dx2, float dy2)

x1, y1 和 x2, y2 则分别是控制点和终点的坐标。

##### 三次贝塞尔曲线

cubicTo(float x1, float y1, float x2, float y2, float x3, float y3) / rCubicTo(float x1, float y1, float x2, float y2, float x3, float y3)

##### 移动到目标位置

moveTo(float x, float y) / rMoveTo(float x, float y) 

##### 弧形

arcTo(RectF rect, float startAngle, float sweepAngle, boolean forceMoveTo) 

这个方法只用来来画弧形，所以不再需要 useCenter 参数；而多出来的这个 forceMoveTo 参数的意思是，绘制是要「抬一下笔移动过去」（true），还是「直接拖着笔过去」（false），区别在于是否留下移动的痕迹。

addArc()为forceMoveTo = true 的简化版 arcTo()。









####

