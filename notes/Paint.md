
# Paint

## 基本设置

### 绘制模式

Paint.setStyle(Style style)

style默认值为FILL。

- Paint.Style.STROKE - 画线模式
- Paint.Style.FILL - 填充模式
- Paint.Style.FILLFILL_AND_STROKE - 两种模式并用

### 颜色

Paint.setColor(int color) 

Paint.setARGB(int a, int r, int g, int b) 

### 线条宽度

Paint.setStrokeWidth(float width) 

### 文字大小

Paint.setTextSize(float textSize) 

### 抗锯齿开关

Paint.setAntiAlias(boolean aa) 

### 线条端点形状

Paint.setStrokeCap(cap)

- Paint.Cap.ROUND - 圆头
- Paint.Cap.BUTT - 平头 
- Paint.Cap.SQUARE - 方头

### 着色器

Paint.setShader(Shader shader)

同时setShader()和setColor/ARGB()和Color时优先使用Shader的颜色。

#### 着色规则 

- Shader.TileMode.CLAMP -  夹子模式（直译），端点之外延续端点处的颜色。
- Shader.TileMode.MIRROR - 镜像模式，以任何一个渐变中间或端点为基准都是堆成的。
- Shader.TileMode.REPEAT - 重复模式。


#### 线性渐变 Shape

LinearGradient(float x0, float y0, float x1, float y1, int color0, int color1, Shader.TileMode tile)

- x0 y0 x1 y1 - 渐变的两个端点的位置。
- color0 color1 - 端点的颜色。
- tile - 端点范围之外的着色规则。



### 


# 参考资料
- [绘制基础 - HenCoder](https://hencoder.com/ui-1-1/)