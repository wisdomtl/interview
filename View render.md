# 绘制流程
1. 画多大？（测量measure） 
2. 画在哪？（定位layout） 
3. 画什么？（绘制draw）
- 测量、定位、绘制都是从View树的根结点开始自顶向下进行地，即都是由父控件驱动子控件进行地。父控件的测量在子控件件测量之后，但父控件的定位和绘制都在子控件之前。
- 父控件测量过程中`ViewGroup.onMeasure()`，会遍历所有子控件并驱动它们测量自己`View.measure()`。父控件还会将父控件的布局模式与子控件布局参数相结合形成一个`MeasureSpec`对象传递给子控件以指导其测量自己。
- 子控件的MeasureSpec有九种情况：3*3的表格，布局模式有三种取值EXACTLY,AT_MOST,UNSPECIFIED,孩子布局参数也有三种情况，具体数值，match_parent,wrap_content.如果孩子是wrapcontent，根据measureChildWithMargin,则孩子的模式是AtMost）。`View.setMeasuredDimension()`是测量过程的终点，它表示`View`大小有了确定值。
- 第一次加载onMeasure()至少调用两次，最多调用5次（都是有ViewRootImpl调用performMeasure()），如果添加的窗口
- 通过 MEASURED_DIMENSION_SET 强制指定需要为 measuredDimension赋值，否则抛异常
- 父控件在完成自己定位之后，会调用`ViewGroup.onLayout()`遍历所有子控件并驱动它们定位自己`View.layout()`。子控件总是相对于父控件左上角定位。`View.setFrame()`是定位过程的终点，它表示视图矩形区域以及相对于父控件的位置已经确定。
- 控件按照绘制背景，绘制自身，绘制孩子的顺序进行。重写onDraw()定义绘制自身的逻辑，父控件在完成绘制自身之后,会调用`ViewGroup.dispatchDraw()`遍历所有子控件并驱动他们绘制自己`View.draw()`
- 为什么只能在主线程绘制界面：因为重绘指令会有子视图一层层传递给父视图，最终传递到ViewRootImpl，它在每次触发View树遍历时都会调用ViewRootImpl.checkThread（）

## requestLayout()和invalidate()
- 其实两个函数都会自底向上传递到顶层视图ViewRootImpl中
- requestLayout()会添加两个标记位PFLAG_FORCE_LAYOUT，PFLAG_INVALIDATED，而invalidate()只会添加PFLAG_INVALIDATED（所以不会测量和布局）
- invalidate(),会检查主线程消息队列中是否已经有遍历view树任务，通过ViewRootImpl.mWillDrawSoon是否为true，若有则不再抛
- invalidate表示当前控件需要重绘，会标记PFLAG_INVALIDATED，重绘请求会逐级上传到根视图（但只有这个view会被重绘，因为其他的父类没有PFLAG_INVALIDATED，并且携带脏区域.初始脏区是发起view的上下左右，逐级向上传递后每次都会换到父亲的坐标系（平移 left，top）。
- View.measure()和View.layout()会先检查是否有PFLAG_FORCE_LAYOUT标记，如果有则进行测量和定位
- View.requestLayout()中设置了PFLAG_FORCE_LAYOUT标记，所以测量，布局，有可能触发onDraw是因为在在layout过程中发现上下左右和之前不一样，那就会触发一次invalidate，所以触发了onDraw。
- postInvalidate 向主线程发送了一个INVALIDATE的消息

## 绘制模型
- 1.软件绘制 2.硬件加速绘制
- 软件绘制就是调用ViewRootImpl持有的 surface.lockCanvas(),获取和SurfaceFlinger共享的匿名内存，并往里面填充像素数据。这些操作发生在主线程。

- 硬件加速和软件绘制的分歧点是在ViewRootImpl中，如果开启了硬件加速则调用mHardwareRenderer.draw，否则drawSoftware。
- 硬件加速绘制分为两个步骤
    1. 在主线程构建DrawOp树：从根视图触发，自顶向下地为每一个视图构建DrawOp。（使用DisplayListCanvas缓存DrawOp，最终存储到RenderNode）
    2. 向RenderThread发一个DrawFrameTask，唤醒它进行渲染
        1. 进行DrawOp合并
        2. 调用gpu命令进行绘制，gpu向匿名共享内存写内容
        2. 最终将填充好的raw Buffer提交给SurfaceFlinger合成显示。
- RenderThread 是单例，每个进程只有一个。
- cpu 不擅长图形计算，硬件加速即是把这些计算交由 GPU 实现（图形计算转成 gpu指令），由此加入了RenderNode（相当于View），和 DisplayList(相当于view的绘制内容)。当重绘请求发起时，只更新 displyList

# DisplayList
- 原先的 drawXXX 都被包装成一个DrawOp，即将执行的绘制命令序列保存在displayList。每一个view有一个displayList，父亲包含孩子的displayList
- 调用各种draw方法时，即是将绘制命令缓存在displayList中（构建 displayList，在主线程发生），等待下一个vsync信号到来时，命令被转换成 gpu指令（渲染displayList在Render thread进行），让gpu执行
- 当view不需要更新时，可以沿用上一次的 displayList，无需调用onDraw，当view只是简单的属性变化时，也无需重建 displayList，只需修改就好
- 硬件绘制是将绘制命令记录在displayList，软件绘制将ui绘制到surface的图形缓冲区（java层即是 canvas中的bitmap）

## 自定义view注意事项
2. 处理padding和margin，在画子控件的时候需要考虑到上下左右的padding值，实际绘画区域会更小，自定义ViewGroup的时候需要考虑margin值
3. 如果开启线程或者动画需要在onDetachedFromWindow中停止
4. 自定义ViewGroup时为了让Margin生效需要自定义一个LayoutParam并继承MarginLayoutParam
5. 自定义View的时候要重写onMeasure(),因为默认的getDefaultSize()对AT_MOST和EXACTLY 做了同样的处理，导致wrap_content和match_parent效果相同，使用resolveSize(),这个方法结合了父亲的约束和自己的诉求。

## 自定义自动换行控件
1. 调用MeasureChildren确定所有孩子的尺寸
2. 如果换行控件测量模式是EXACTLY时，则高度和宽度都是measureSpec的值，否则遍历孩子将它们的宽高累加
3. 遍历孩子布局每个孩子的上下左右

## MeasureSpec
MeasureSpec用于在View测量过程中描述尺寸，它是一个包含了布局模式和布局尺寸的int值(32位)，其中最高的2位代表布局模式，后30位代表布局尺寸。它包含三种布局模式分别是
1. UNSPECIFIED：父亲没有规定你多大
2. EXACTLY：父亲给你设定了一个死尺寸
3. AT_MOST：父亲规定了你的最大尺寸

# View
- mDrawingCache是一个bitmap用来缓存上一次绘制的样子，当这一帧不需要重绘时，直接用
- 如果该值为null会创建一个bitmap并保存在该字段中，然后根据bitmap对象创建canvas对象

# Canvas
- 硬件加速Canvas叫DisplayListCanvas和非硬件级加速是不同对象
- 父控件的Canvas 和子控件不是一个对象
- 控件的Canvas 是在updateDisplayListIfDirty 中创建出来的
- Canvas 使用池来管理，避免每次重复创建。所有的控件共用一个Canvas池

# 动画
1. view animation 
    - 也称为补间动画：定义关键帧，其余帧由系统补齐
    - 局限于view 局限于位移 旋转 透明度 缩放
    - 在View重绘时,View会从父控件Transform中获取动画矩阵,并根据这个矩阵计算出View在屏幕中的位置和大小,然后进行绘制，每次绘制的时候会检查动画是否完成，若没完成则调用invalidate(),ViewRootImpl向编舞者跑了一个遍历View树的任务，会有同步消息屏障
    - View Animation实际上是通过不断重绘View(onDraw())并更新其transform属性来产生动画效果的(没有执行measure 和 layout)
    - 不能监听动画变化的过程
    - 还能响应原有位置的触摸事件，是因为会将 matrix 反向运算
2. property animation
    - 构建值变化的完整序列 并将其运用到视图属性上，直接改变控件属性，而不是父控件的transform矩阵
    - 不仅仅适用于view 可用于任何提供了getter和setter的对象的任何属性上
    - 通过向 Choreographer 不停post 一个动画类型任务（在绘制任务之前执行）, 当前帧完毕后若动画未结束继续post 没有同步消息屏障
    - 可监听动画变化过程
    - Interpolator决定值变化的速度（根据时间流逝的百分比计算出当前属性值改变的百分比）
    
# 修改View绘制顺序
自定义ViewGroup时可以重写setChildrenDrawingOrderEnabled()来开启自定义孩子绘制顺序（即通过getChildDrawingOrder()确定孩子绘制顺序）

# Scroller
- 使用方法：Scroller.startScroll()开始计算+computeScrollOffset()计算当前偏移量+getScrollX()获取当前偏移量+invalidate()触发重绘

# ConstraintLayout
- 额外的封装，会将容器控件包装成ConstraintWidgetContainer，子控件包装成ConstraintWidget
- 在测量布局的时候，会想把所有的ConstraintWidget移除，然后遍历所有子控件并重新添加ConstraintWidget，如果子控件有约束则将其连接到锚点，构建依赖图，深度遍历依赖图，进行求解（Cassowary 算法），最终得到相对于父亲的上下左右。