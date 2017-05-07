---
title: 【Android】如何在View中绘制一个“洞”
date: 2017-05-07
---
有时，我们需要在界面上绘制一个”洞“，如进行图片裁剪的时候，需要显示一个裁剪的范围，如图：

![图片裁剪](/imgs/hole_views_crop.png)

下面介绍几种实现方法，以在一个View中间画一个圆洞为例

## 1. 直接绘制外部

最直观的方法，用一个Path填充”圆洞”的外部

关键代码如下：

在Size变化的时候，计算Path
``` java
// 计算中间圆 mCircleRect
// ......

mOutsidePath.reset();

// Move到顶端中点
mOutsidePath.moveTo(centerX, 0);
mOutsidePath.lineTo(centerX, centerY);

// 添加一个圆
mOutsidePath.addArc(mCircleRect, 270, 360);
mOutsidePath.lineTo(centerX, 0);
mOutsidePath.lineTo(0, 0);
mOutsidePath.lineTo(0, h);
mOutsidePath.lineTo(w, h);
mOutsidePath.lineTo(w, 0);
mOutsidePath.lineTo(centerX, 0);
mOutsidePath.close();
```

onDraw的时候，直接绘制
``` java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    mPaint.setColor(mOutsideColor);
    mPaint.setStyle(Paint.Style.FILL);
    canvas.drawPath(mOutsidePath, mPaint);
}
```

这样，通过绘制圆的外部区域，就实现了一个圆洞

## 2. Path DIFFERENCE 操作

通过
``` java
Path.op(Path p, Op op)
```
可以进行Path的相应操作，其中DIFFERENCE可以进行“减”操作：

``` java
// 计算中间圆 mCircleRect
// ......

mCirclePath.reset();
mCirclePath.addOval(mCircleRect, Path.Direction.CW);
mCirclePath.close();

mFillPath.reset();
mFillPath.addRect(0, 0, w, h, Path.Direction.CW);
mFillPath.close();

// isInEditMode是因为IDE还不支持Path.op操作
if (!isInEditMode())
    mFillPath.op(mCirclePath, Path.Op.DIFFERENCE);
```

由一个大的Path，减去一个圆的Path，也就得到了一个”洞”

## 3. PATH EVEN_ODD
此方法与方法2类似，只是将op操作替换为：
``` java
mFillPath.addPath(mCirclePath);
mFillPath.setFillType(Path.FillType.EVEN_ODD);
```

Path的FillType还有WINDING, REVERSE_WINDING, REVERSE_EVEN_ODD等，其中的区别可自行研究。

**此方法需要API19以上才支持**

## 4. XferMode
XferMode是一个神奇的玩意儿，可以大致理解为“混合”模式
使用XferMode的Clear Mode，可以将界面上相应的区域”清除”，因此也是可以用来绘制“洞”的，关键代码如下

在初始化的时候，设置Hardware Accelerate
``` java
private void initView() {
    mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    mCircleRect = new RectF();
    mCirclePath = new Path();
    mFillPath = new Path();
    mClearMode = new PorterDuffXfermode(PorterDuff.Mode.CLEAR);

    setLayerType(View.LAYER_TYPE_HARDWARE, mPaint);
}
```

在绘制的时候，使用XferMode
``` java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    canvas.drawColor(mOutsideColor);

    mPaint.setXfermode(mClearMode);
    mPaint.setColor(Color.TRANSPARENT);
    mPaint.setStyle(Paint.Style.FILL);
    canvas.drawPath(mCirclePath, mPaint);
}
```
记得不要在onDraw方法里面去创建对象。

**此方法需要支持硬件加速**

## 运行效果
![运行效果](/imgs/hole_views_result.jpg)

具体代码见github: [Github](https://github.com/dogold/andemos/tree/master/widgets/src/main/java/com/dogold/andemos/widgets/customwidgets/holeviews)
