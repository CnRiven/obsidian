# View 绘制流程 - measure layout draw

> View 绘制是 Android UI 的核心，面试必问！

---

## 一、整体流程

```java
Activity.onCreate()
    ↓
setContentView()
    ↓
PhoneWindow.setContentView()
    ↓
DecorView 添加到 Window
    ↓
ViewRootImpl.performTraversals()
    ↓
┌─────────────────────────────────────┐
│  performMeasure() → measure()       │
│  performLayout()  → layout()        │
│  performDraw()    → draw()          │
└─────────────────────────────────────┘
```java

### 1. ViewRootImpl.performTraversals()

```java
// ViewRootImpl.java
private void performTraversals() {
    // 1. 测量
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

    // 2. 布局
    performLayout(lp, mWidth, mHeight);

    // 3. 绘制
    performDraw();
}
```java

### 2. 绘制的触发时机

```java
// 1. Activity 创建时
Activity.onCreate() → setContentView() → ViewRootImpl.performTraversals()

// 2. View.invalidate()
View.invalidate() → ViewRootImpl.scheduleTraversals() → performTraversals()

// 3. View.requestLayout()
View.requestLayout() → ViewRootImpl.scheduleTraversals() → performTraversals()
```

---

## 二、Measure 流程

### 1. MeasureSpec

```java
// 32 位 int，高 2 位是模式，低 30 位是大小
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK = 0x3 << MODE_SHIFT;

    // 三种模式
    public static final int UNSPECIFIED = 0 << MODE_SHIFT; // 不限制
    public static final int EXACTLY = 1 << MODE_SHIFT;     // 精确值
    public static final int AT_MOST = 2 << MODE_SHIFT;     // 最大值

    // 创建 MeasureSpec
    public static int makeMeasureSpec(int size, int mode) {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }

    // 获取模式
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }

    // 获取大小
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```java

### 2. MeasureSpec 与 LayoutParams 的关系

```java
// ViewGroup.getChildMeasureSpec()
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                // 子 View 指定了具体尺寸
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // 子 View 想要和 Parent 一样大
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // 子 View 想要自己决定大小
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                resultSize = 0;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
    }

    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

**关系总结：**

| Parent SpecMode | Child LayoutParams | Child SpecMode |
|-----------------|-------------------|----------------|
| EXACTLY | 具体 dp | EXACTLY |
| EXACTLY | match_parent | EXACTLY |
| EXACTLY | wrap_content | AT_MOST |
| AT_MOST | 具体 dp | EXACTLY |
| AT_MOST | match_parent | AT_MOST |
| AT_MOST | wrap_content | AT_MOST |
| UNSPECIFIED | 具体 dp | EXACTLY |
| UNSPECIFIED | match_parent | UNSPECIFIED |
| UNSPECIFIED | wrap_content | UNSPECIFIED |

### 3. View 的 measure

```java
// View.java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // 如果规格没变，直接返回（优化）
    if (mOldWidthMeasureSpec == widthMeasureSpec && 
        mOldHeightMeasureSpec == heightMeasureSpec) {
        return;
    }

    // 回调 onMeasure
    onMeasure(widthMeasureSpec, heightMeasureSpec);
}

protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(
        getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec)
    );
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
    }
    return result;
}
```java

### 4. ViewGroup 的 measure

```java
// ViewGroup 没有 onMeasure，由子类实现
// 以 LinearLayout 为例
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}

void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    int totalHeight = 0;
    int maxWidth = 0;
    int totalWeight = 0;

    // 遍历子 View
    for (int i = 0; i < count; ++i) {
        final View child = getChildAt(i);

        // 测量子 View
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, totalHeight);

        totalHeight += child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin;
        maxWidth = Math.max(maxWidth, child.getMeasuredWidth());
        totalWeight += lp.weight;
    }

    // 设置自己的尺寸
    setMeasuredDimension(resolveSize(maxWidth, widthMeasureSpec),
                         resolveSize(totalHeight, heightMeasureSpec));
}

protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    // 计算子 View 的 MeasureSpec
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```java

### 5. 自定义 View 的 measure

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    int width, height;

    // 计算宽度
    if (widthMode == MeasureSpec.EXACTLY) {
        width = widthSize; // match_parent 或具体值
    } else {
        width = calculateWidth(); // wrap_content，自己计算
        if (widthMode == MeasureSpec.AT_MOST) {
            width = Math.min(width, widthSize); // 不能超过父容器
        }
    }

    // 计算高度（同理）
    if (heightMode == MeasureSpec.EXACTLY) {
        height = heightSize;
    } else {
        height = calculateHeight();
        if (heightMode == MeasureSpec.AT_MOST) {
            height = Math.min(height, heightSize);
        }
    }

    setMeasuredDimension(width, height);
}
```

---

## 三、Layout 流程

### 1. View 的 layout

```java
// View.java
public void layout(int l, int t, int r, int b) {
    // 设置位置
    boolean changed = setFrame(l, t, r, b);

    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        // 回调 onLayout
        onLayout(changed, l, t, r, b);
    }
}

protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    // View 没有子 View，空实现
}
```java

### 2. ViewGroup 的 layout

```java
// ViewGroup.java
@Override
public final void layout(int l, int t, int r, int b) {
    // 调用 View.layout()
    super.layout(l, t, r, b);

    // 布局子 View
    onLayout(changed, l, t, r, b);
}

// 以 FrameLayout 为例
@Override
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    layoutChildren(left, top, right, bottom, false);
}

void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
    final int count = getChildCount();

    final int parentLeft = getPaddingLeftWithForeground();
    final int parentRight = right - left - getPaddingRightWithForeground();
    final int parentTop = getPaddingTopWithForeground();
    final int parentBottom = bottom - top - getPaddingBottomWithForeground();

    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final int width = child.getMeasuredWidth();
            final int height = child.getMeasuredHeight();

            int childLeft;
            int childTop;

            int gravity = lp.gravity;
            if (gravity == -1) {
                gravity = mForegroundGravity;
            }

            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

            // 计算水平位置
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                            lp.leftMargin - lp.rightMargin;
                    break;
                case Gravity.RIGHT:
                    if (!forceLeftGravity) {
                        childLeft = parentRight - width - lp.rightMargin;
                        break;
                    }
                case Gravity.LEFT:
                default:
                    childLeft = parentLeft + lp.leftMargin;
            }

            // 计算垂直位置
            switch (verticalGravity) {
                case Gravity.TOP:
                    childTop = parentTop + lp.topMargin;
                    break;
                case Gravity.CENTER_VERTICAL:
                    childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                            lp.topMargin - lp.bottomMargin;
                    break;
                case Gravity.BOTTOM:
                    childTop = parentBottom - height - lp.bottomMargin;
                    break;
                default:
                    childTop = parentTop + lp.topMargin;
            }

            // 设置子 View 位置
            child.layout(childLeft, childTop, childLeft + width, childTop + height);
        }
    }
}
```

### 3. Layout 与 onLayout 的区别

```java
// layout()：设置 View 的位置（mLeft, mTop, mRight, mBottom）
// onLayout()：布局子 View（ViewGroup 重写）

// View 的位置：
mLeft    // 左边界
mTop     // 上边界
mRight   // 右边界
mBottom  // 下边界

// 获取位置：
view.getLeft()   // mLeft
view.getTop()    // mTop
view.getRight()  // mRight
view.getBottom() // mBottom
```

---

## 四、Draw 流程

### 1. 绘制步骤

```java
// View.java
public void draw(Canvas canvas) {
    // 1. 绘制背景
    drawBackground(canvas);

    // 2. 保存 canvas 层（如果需要 fading edge）
    // ...

    // 3. 绘制自身内容
    onDraw(canvas);

    // 4. 绘制子 View
    dispatchDraw(canvas);

    // 5. 绘制装饰（滚动条等）
    onDrawForeground(canvas);

    // 6. 绘制默认焦点高亮
    drawDefaultFocusHighlight(canvas);
}
```

### 2. 绘制顺序

```text
1. drawBackground(canvas)      // 绘制背景
2. onDraw(canvas)              // 绘制自身内容
3. dispatchDraw(canvas)        // 绘制子 View
4. onDrawForeground(canvas)    // 绘制前景（滚动条等）
```java

### 3. View 的 onDraw

```java
// 自定义 View
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    // 绘制内容
    canvas.drawRect(0, 0, getWidth(), getHeight(), mPaint);
    canvas.drawText("Hello", x, y, mTextPaint);
    canvas.drawCircle(cx, cy, radius, mCirclePaint);
}
```java

### 4. ViewGroup 的 dispatchDraw

```java
// ViewGroup.java
@Override
protected void dispatchDraw(Canvas canvas) {
    final int count = mChildrenCount;
    final View[] children = mChildren;

    for (int i = 0; i < count; i++) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
            // 绘制子 View
            drawChild(canvas, child, drawingTime);
        }
    }
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```java

### 5. Canvas 和 Paint

```java
// Canvas：画布，提供绘制方法
canvas.drawRect(rect, paint);
canvas.drawCircle(cx, cy, radius, paint);
canvas.drawText(text, x, y, paint);
canvas.drawBitmap(bitmap, left, top, paint);
canvas.drawPath(path, paint);

// Paint：画笔，设置绘制样式
Paint paint = new Paint();
paint.setColor(Color.RED);           // 颜色
paint.setStrokeWidth(2f);            // 线宽
paint.setStyle(Paint.Style.FILL);    // 填充样式
paint.setTextSize(16f);              // 文字大小
paint.setAntiAlias(true);            // 抗锯齿
```java

---

## 五、自定义 View

### 1. 继承 View

```java
public class CircleView extends View {
    private Paint mPaint;
    private int mRadius;

    public CircleView(Context context) {
        this(context, null);
    }

    public CircleView(Context context, AttributeSet attrs) {
        super(context, attrs);

        // 获取自定义属性
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.CircleView);
        mRadius = ta.getDimensionPixelSize(R.styleable.CircleView_radius, 100);
        ta.recycle();

        init();
    }

    private void init() {
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(Color.RED);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width, height;

        // 处理 wrap_content
        if (widthMode == MeasureSpec.EXACTLY) {
            width = widthSize;
        } else {
            width = mRadius * 2 + getPaddingLeft() + getPaddingRight();
            if (widthMode == MeasureSpec.AT_MOST) {
                width = Math.min(width, widthSize);
            }
        }

        if (heightMode == MeasureSpec.EXACTLY) {
            height = heightSize;
        } else {
            height = mRadius * 2 + getPaddingTop() + getPaddingBottom();
            if (heightMode == MeasureSpec.AT_MOST) {
                height = Math.min(height, heightSize);
            }
        }

        setMeasuredDimension(width, height);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        int cx = getWidth() / 2;
        int cy = getHeight() / 2;
        canvas.drawCircle(cx, cy, mRadius, mPaint);
    }
}
```java

### 2. 继承 ViewGroup

```java
public class CustomLayout extends ViewGroup {
    public CustomLayout(Context context) {
        super(context);
    }

    public CustomLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width = 0;
        int height = 0;

        // 测量所有子 View
        measureChildren(widthMeasureSpec, heightMeasureSpec);

        // 计算自己的尺寸
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                width += child.getMeasuredWidth();
                height = Math.max(height, child.getMeasuredHeight());
            }
        }

        // 处理 padding
        width += getPaddingLeft() + getPaddingRight();
        height += getPaddingTop() + getPaddingBottom();

        // 设置尺寸
        setMeasuredDimension(
            resolveSize(width, widthMeasureSpec),
            resolveSize(height, heightMeasureSpec)
        );
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int currentX = getPaddingLeft();
        int currentY = getPaddingTop();

        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                int childWidth = child.getMeasuredWidth();
                int childHeight = child.getMeasuredHeight();

                // 检查是否需要换行
                if (currentX + childWidth > getWidth() - getPaddingRight()) {
                    currentX = getPaddingLeft();
                    currentY += childHeight;
                }

                // 布局子 View
                child.layout(currentX, currentY, currentX + childWidth, currentY + childHeight);

                currentX += childWidth;
            }
        }
    }
}
```

### 3. 自定义属性

```xml
<!-- res/values/attrs.xml -->
<declare-styleable name="CircleView">
    <attr name="radius" format="dimension" />
    <attr name="circleColor" format="color" />
</declare-styleable>
```java

```java
// 获取自定义属性
TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.CircleView);
int radius = ta.getDimensionPixelSize(R.styleable.CircleView_radius, 100);
int color = ta.getColor(R.styleable.CircleView_circleColor, Color.RED);
ta.recycle();
```

---

## 六、invalidate 和 requestLayout

### 1. invalidate()

```java
// View.java
public void invalidate() {
    invalidate(true);
}

void invalidate(boolean invalidateCache) {
    if (invalidateCache) {
        mPrivateFlags |= PFLAG_INVALIDATED;
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    }

    // 通知父 View 重绘
    final ViewParent p = mParent;
    if (p != null) {
        p.invalidateChild(this, null);
    }
}

// 最终调用 ViewRootImpl.scheduleTraversals()
// 只会触发 draw，不会触发 measure 和 layout
```java

### 2. requestLayout()

```java
// View.java
public void requestLayout() {
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;

    // 通知父 View 重新布局
    final ViewParent p = mParent;
    if (p != null) {
        p.requestLayout();
    }
}

// 最终调用 ViewRootImpl.scheduleTraversals()
// 会触发 measure、layout 和 draw
```

### 3. 区别

| 方法 | 触发 | 使用场景 |
|------|------|----------|
| invalidate() | 只触发 draw | 外观变化（颜色、透明度等） |
| requestLayout() | 触发 measure + layout + draw | 尺寸或位置变化 |

---

## 七、常见问题

### Q1: wrap_content 不生效的原因？
- 默认情况下，View 的 onMeasure 对 wrap_content 和 match_parent 处理一样
- 解决：在 onMeasure 中处理 AT_MOST 模式

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);

    int width;
    if (widthMode == MeasureSpec.EXACTLY) {
        width = widthSize;
    } else {
        width = defaultWidth; // 设置默认值
        if (widthMode == MeasureSpec.AT_MOST) {
            width = Math.min(width, widthSize);
        }
    }

    setMeasuredDimension(width, height);
}
```java

### Q2: 如何获取 View 的宽高？

```java
// 方法一：addOnGlobalLayoutListener
view.getViewTreeObserver().addOnGlobalLayoutListener(new OnGlobalLayoutListener() {
    @Override
    public void onGlobalLayout() {
        int width = view.getWidth();
        int height = view.getHeight();
        view.getViewTreeObserver().removeOnGlobalLayoutListener(this);
    }
});

// 方法二：post
view.post(() -> {
    int width = view.getWidth();
    int height = view.getHeight();
});

// 方法三：addOnLayoutChangeListener
view.addOnLayoutChangeListener((v, left, top, right, bottom, oldLeft, oldTop, oldRight, oldBottom) -> {
    int width = v.getWidth();
    int height = v.getHeight();
});
```

### Q3: View.post 和 Handler.post 的区别？
- View.post：如果 View 已经 attach 到 Window，直接发到 Handler
- 如果还没 attach，会保存到 RunQueue，等 attach 后执行
- Handler.post：直接发到 MessageQueue

### Q4: getWidth() 和 getMeasuredWidth() 的区别？
- getWidth()：layout 后的宽度（mRight - mLeft）
- getMeasuredWidth()：measure 后的宽度（mMeasuredWidth）
- 通常相等，但在某些情况下可能不同

---

## 八、最佳实践

### 1. 避免在 onDraw 中创建对象
```java
// 错误示例
@Override
protected void onDraw(Canvas canvas) {
    Paint paint = new Paint(); // 每次绘制都创建
    canvas.drawRect(0, 0, getWidth(), getHeight(), paint);
}

// 正确示例
private Paint mPaint = new Paint(); // 构造函数中创建

@Override
protected void onDraw(Canvas canvas) {
    canvas.drawRect(0, 0, getWidth(), getHeight(), mPaint);
}
```java

### 2. 处理 padding 和 margin
```java
// 自定义 View 要考虑 padding
@Override
protected void onDraw(Canvas canvas) {
    int left = getPaddingLeft();
    int top = getPaddingTop();
    int right = getWidth() - getPaddingRight();
    int bottom = getHeight() - getPaddingBottom();

    canvas.drawRect(left, top, right, bottom, mPaint);
}

// ViewGroup 要考虑 margin
MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
int left = childLeft + lp.leftMargin;
int top = childTop + lp.topMargin;
```java

### 3. 处理 wrap_content
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);

    int width;
    if (widthMode == MeasureSpec.EXACTLY) {
        width = widthSize;
    } else {
        width = calculateDefaultWidth();
        if (widthMode == MeasureSpec.AT_MOST) {
            width = Math.min(width, widthSize);
        }
    }

    setMeasuredDimension(width, height);
}
```java

### 4. 支持自定义属性
```java
public CircleView(Context context, AttributeSet attrs) {
    super(context, attrs);

    TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.CircleView);
    mRadius = ta.getDimensionPixelSize(R.styleable.CircleView_radius, 100);
    ta.recycle();
}
```java

### 5. 处理触摸事件
```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            // 处理按下
            return true;
        case MotionEvent.ACTION_MOVE:
            // 处理移动
            return true;
        case MotionEvent.ACTION_UP:
            // 处理抬起
            return true;
    }
    return super.onTouchEvent(event);
}
```
