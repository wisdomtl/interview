# 渲染
- 渲染的起点是setState() 最终调用到BuildOwner.scheduleBuildFor(){
	1. SchedulerBinding.scheduleFrame()预约下一帧 {
		委托Choreography注册下一个vsync
	}
	2. 将element添加到脏列表_dirtyElements
}
等到信号来到，再通知 Dart 进行渲染，渲染分为build-layout-paint-composite四大阶段
1. build：根据Widget构建Element并挂载到Element tree，然后遍历tree 构建RenderObject
2. Layout（布局的计算）：确定每个子widget大小和在屏幕中的位置。
3. Paint（视图的绘制）：为每个子widget提供canvas，让他们绘制自己。
4. Composite（合成）：所有widget合成到一起，交给GPU处理。

# 视图
Widget+Element+RenderObject 对应有三棵树 widget树，element树，RenderObject树
- Widget是视图的配置信息，当Widget被创建时会通过inflateWidget()创建一个Element并调用mount()挂载到Element tree上，通过遍历Element tree来构建RenderObject
- RenderObject负责布局和绘制
	- layout():布局原则是，约束向下，尺寸向上：父亲将约束传递给孩子，孩子把自己的尺寸传递给父亲
	- relayoutBoundary用于避免子控件重新布局导致其所有父控件都重新布局
	- paint():绘制

# widget生命周期（State生命周期）
1. StatefulWidget.createState()构建State
2. initState()
3. didChangeDependencies()
4. build()构建视图
控件第一次被渲染会调用上面三个生命周期方法
5. 当State依赖关系发生变化时会被调用 didChangeDependencies()--- didUpdateWidget()---build()
当控件被移除时调用下面两个方法
5. deactivate()当state从视图树中移除时
6. dispose() 当State被永久移除

调用setState()之后其实会对从当前节点开始，他的所有子孙节点调用updateChild(Element child, Widget newWidget, dynamic newSlot)

# dart
默认情况下 dart 代码运行在 main isolate，app处于一个无限循环之中，不断从microtask队列和event队列中获取消息并执行。

## dart 异步
1. 使用Future，向event队列尾部发消息
2. scheduleMicrotask() 下个microtask队列尾部发消息


mixin 是一个特殊的类，它的属性和行为可以被其他类复用，而且不需要通过继承。