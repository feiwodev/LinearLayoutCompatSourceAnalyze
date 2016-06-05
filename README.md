#LinearLayoutCompat源码分析

###简介
在编写android程序的时候 ， 大多数人只知道使用系统内置的控件 ， 忽略了系统为我们打的补丁控件 ，可以大大的减少我们重复编码的次数 ，很多功能需要我们自己去实现的 ， google通过v4 , v7 包 ，大部分都打了补丁 ， 方便我们的开发人员  ， 但我们使用里面的控件 ， 却少之又少 ， 一般常用 `ViewPager` , `SwipeRefresh` ，`Fragment`等等 。其实在V7包中 ， google为我们打了很多传统控件的补丁 ， 比如`AppCompatButton` , `AppCompatTextView`等等 。今天我来分析一个比较实用的布局控件 ， `LinearLayoutCompat` ， 这个类 ， 可以很方便的做出设置界面的效果 ， 如

![](https://raw.githubusercontent.com/zhuyongit/LinearLayoutCompatSourceAnalyze/master/LinearLayoutCompat.png)

那么下面我们就来分析分析这个类 。

当我们要分析一个Android类的时候 ， 我们需要明确以下几点 ：

	1. 这个类做什么的 ，是控件还是组件，还是工具类或其他
	2. 找入口 ，一般为构造函数 ，看其在初始化中做了什么
	3. 找关键方法 ，如控件的onDraw ，组件的生命周期 ，工具类的主要处理方法或类

通过看源码 ， 我们知道 ， LinearLayoutCompat是一个ViewGroup子类 ， 是一个自定义控件  。写过自定义控件的都知道 ， 我们一般在构造方法中 ， 将控件属性加载进来 ，而我们的LinearLayoutCompat也不例外 ， 他加载的是LinearLayoutCompat这个styleable

``` java

	TintTypedArray a = TintTypedArray.obtainStyledAttributes(context, attrs,
                R.styleable.LinearLayoutCompat, defStyleAttr, 0);
```

styleable如下：

	<declare-styleable name="LinearLayoutCompat">
        <attr name="android:orientation" />
        <attr name="android:gravity" />
        <attr name="android:baselineAligned" />
        <attr name="android:baselineAlignedChildIndex" />
        <attr name="android:weightSum" />
        <attr name="measureWithLargestChild" format="boolean" />
        <attr name="divider" />
        <attr name="showDividers">
            <flag name="none" value="0" />
            <flag name="beginning" value="1" />
            <flag name="middle" value="2" />
            <flag name="end" value="4" />
        </attr>
        <attr name="dividerPadding" format="dimension" />
    </declare-styleable>

了解了属性 ， 接下来我们找关键方法 ， 我们知道 ，自定义控件 ， 最核心的地方在绘制 ， 接下来 ， 我们找到onDraw方法

	@Override
    protected void onDraw(Canvas canvas) {
        if (mDivider == null) {
            return;
        }
        if (mOrientation == VERTICAL) {
            drawDividersVertical(canvas);
        } else {
            drawDividersHorizontal(canvas);
        }
    }
	
我们看到 ， onDraw两个判断语句就搞定了 

 	1. 判断有没有divider也就是LinearLayoutCompat子控件的分割线 ， 是Drawable类型的 。
 	2. 判断画线的方向 ， 默认是画水平分割线 ，并且画线

最核心的方法也就是drawDividerVertical和drawDividerHorizotal ， 主要负责画线 。

我们首先来看drawDividerVertical方法 

### 画水平线

	void drawDividersVertical(Canvas canvas) {
        final int count = getVirtualChildCount();
        for (int i = 0; i < count; i++) {
            // 得到LinearLayoutCompat下的子元素个数
            final View child = getVirtualChildAt(i);
            // 判断是否存在子元素，并且子元素不为空
            if (child != null && child.getVisibility() != GONE) {
                if (hasDividerBeforeChildAt(i)) {
                    // 得到子元素的布局参数
                    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                    // 得到子元素距离父元素顶部的距离
                    final int top = child.getTop() - lp.topMargin - mDividerHeight;
                    // 画水平线
                    drawHorizontalDivider(canvas, top);
                }
            }
        }
        
        // 判断showDivider属性
        if (hasDividerBeforeChildAt(count)) {
            final View child = getVirtualChildAt(count - 1);
            int bottom = 0;
            if (child == null) {
                bottom = getHeight() - getPaddingBottom() - mDividerHeight;
            } else {
               // 子元素如果设置了marginBottom ， 则将divider画在距离子元素marginBottom处
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                bottom = child.getBottom() + lp.bottomMargin;
            }
            drawHorizontalDivider(canvas, bottom);
        }
    }

### 画线

	void drawHorizontalDivider(Canvas canvas, int top) {
       // 画线 ，计算左右两边的边距 ，顶部距离
        mDivider.setBounds(getPaddingLeft() + mDividerPadding, top,
                getWidth() - getPaddingRight() - mDividerPadding, top + mDividerHeight);
        mDivider.draw(canvas);
    }

接下来看一下 ，showDividers属性

	protected boolean hasDividerBeforeChildAt(int childIndex) {
        if (childIndex == 0) {
            return (mShowDividers & SHOW_DIVIDER_BEGINNING) != 0;
        } else if (childIndex == getChildCount()) {
            return (mShowDividers & SHOW_DIVIDER_END) != 0;
        } else if ((mShowDividers & SHOW_DIVIDER_MIDDLE) != 0) {
            boolean hasVisibleViewBefore = false;
            for (int i = childIndex - 1; i >= 0; i--) {
                if (getChildAt(i).getVisibility() != GONE) {
                    hasVisibleViewBefore = true;
                    break;
                }
            }
            return hasVisibleViewBefore;
        }
        return false;
    }

此方法就做了些判断 ， 判断是否需要绘制头部和底部中间的divider

如此，我们就分析完了画水平Divider， 垂直Divider原理其实都差不多  ， 计算Divider的宽度 ，和子元素的左边右边据 ，再进行计算。

这里只进行Divider的绘制原理进行分析 ，其他的像测量 ，和布局计算 ，就没做分析 ， 和linearLayout相似 。