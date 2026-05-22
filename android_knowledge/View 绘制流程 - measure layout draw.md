# View 绘制流程 - measure layout draw

> View 绘制是 Android UI 的核心，面试必问！

---

## 一、整体流程

```
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
    public static final int EXACTLY = 1 << MODE_SHIFT;     // 精确值（match_parent 或具体 dp）
    public static final int AT_MOST = 2 << MODE_SHIFT;     // 最大值（wrap_content）
    
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
```

### 2. View 的 measure

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
```

### 3. ViewGroup 的 measure

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
    
    // 遍历子 View
    for (int i = 0; i < count; ++i) {
        final View child = getChildAt(i);
        
        // 测量子 View
        measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, totalHeight);
        
        totalHeight += child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin;
        maxWidth = Math.max(maxWidth, child.getMeasuredWidth());
    }
    
    // 设置自己的尺寸
    setMeasuredDimension(resolveSize(maxWidth, widthMeasureSpec),
                         resolveSize(totalHeight, heightMeasureSpec));
}
```

### 4. 自定义 View 的 measure

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
            width = Math.min(width, widthSize);
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
    setFrame(l, t, r, b);
    
    // 回调 onLayout
    onLayout(changed, l, t, r, b);
}

protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    // View 没有子 View，空实现
}
```

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
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        
        // 根据 gravity 计算位置
        int childLeft, childTop;
        
        switch (gravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
            case Gravity.CENTER_HORIZONTAL:
                childLeft = parentLeft + (parentRight - parentLeft - width) / 2;
                break;
            case Gravity.RIGHT:
                childLeft = parentRight - width;
                break;
            default: // LEFT
                childLeft = parentLeft;
        }
        
        // 设置子 View 位置
        child.layout(childLeft, childTop, childLeft + width, childTop + height);
    }
}
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

### 2. View 的 onDraw

```java
// 自定义 View
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    
    // 绘制内容
    canvas.drawRect(0, 0, getWidth(), getHeight(), mPaint);
    canvas.drawText("Hello", x, y, mTextPaint);
}
```

### 3. ViewGroup 的 dispatchDraw

```java
// ViewGroup.java
@Override
protected void dispatchDraw(Canvas canvas) {
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        
        // 绘制子 View
        drawChild(canvas, child, drawingTime);
    }
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
    return child.draw(canvas, this, drawingTime);
}
```

---

## 五、自定义 View

### 1. 继承 View

```java
public class CircleView extends View {
    private Paint mPaint;
    
    public CircleView(Context context) {
        this(context, null);
    }
    
    public CircleView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }
    
    private void init() {
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(Color.RED);
    }
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 处理 wrap_content
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        
        int width;
        if (widthMode == MeasureSpec.EXACTLY) {
            width = widthSize;
        } else {
            width = 200; // 默认大小
            if (widthMode == MeasureSpec.AT_MOST) {
                width = Math.min(width, widthSize);
            }
        }
        
        setMeasuredDimension(width, width);
    }
    
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        int radius = getWidth() / 2;
        canvas.drawCircle(radius, radius, radius, mPaint);
    }
}
```

### 2. 继承 ViewGroup

```java
public class CustomLayout extends ViewGroup {
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        measureChildren(widthMeasureSpec, heightMeasureSpec);
        
        int width = calculateWidth();
        int height = calculateHeight();
        
        setMeasuredDimension(
            resolveSize(width, widthMeasureSpec),
            resolveSize(height, heightMeasureSpec)
        );
    }
    
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int currentX = 0;
        int currentY = 0;
        
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            
            if (currentX + child.getMeasuredWidth() > getWidth()) {
                currentX = 0;
                currentY += child.getMeasuredHeight();
            }
            
            child.layout(
                currentX,
                currentY,
                currentX + child.getMeasuredWidth(),
                currentY + child.getMeasuredHeight()
            );
            
            currentX += child.getMeasuredWidth();
        }
    }
}
```

---

## 六、面试高频问题

### Q1: View 的绘制流程？
- measure → 确定尺寸
- layout → 确定位置
- draw → 绘制内容

### Q2: invalidate 和 requestLayout 的区别？
| 方法 | 作用 | 触发 |
|------|------|------|
| invalidate() | 重绘 | 只调用 draw |
| requestLayout() | 重新布局 | 调用 measure + layout + draw |

### Q3: wrap_content 不生效的原因？
- 默认情况下，View 的 onMeasure 对 wrap_content 和 match_parent 处理一样
- 解决：在 onMeasure 中处理 AT_MOST 模式

### Q4: View.post 和 Handler.post 的区别？
- View.post：如果 View 已经 attach 到 Window，直接发到 Handler
- 如果还没 attach，会保存到 RunQueue，等 attach 后执行
- Handler.post：直接发到 MessageQueue

### Q5: 如何获取 View 的宽高？
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
```

---

## 七、最佳实践

1. **避免在 onDraw 中创建对象**：Paint、Path 等在构造函数中创建
2. **处理 padding 和 margin**：自定义 View 要考虑 padding，ViewGroup 要考虑 margin
3. **处理 wrap_content**：自定义 View 必须处理 AT_MOST 模式
4. **支持自定义属性**：通过 TypedArray 获取自定义属性
5. **处理触摸事件**：如果需要交互，重写 onTouchEvent
