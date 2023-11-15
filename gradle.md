# gradle 生命周期
1. Initialization (初始化)
    - 决定构建中包含哪些项目，并构建对应的project实例。
2. Configuration (配置)
    - 生成task的依赖图（DAG,有向无环图），应用插件，注册task
3. Execution (执行)
    - 执行所需的task

# Task
- Task是一个任务，是Gradle中最小的构建单元,Task管理了一个Action的List，你可以在List前面插入一个Action（doFirst），也可以从list后面插入（doLast），Action是最小的执行单元。
- Task的创建是在Project中，一个build.gradle文件对应一个Project对象，所以我们可以直接在build.gradle文件中创建Task。
- 增量构建：是当Task的输入和输出没有变化时，跳过action的执行，当Task输入或输出发生变化时，在action中只对发生变化的输入或输出进行处理，这样就可以避免一个没有变化的Task被反复构建，当Task发生变化时也只处理变化部分，这样可以提高Gradle的构建效率，缩短构建时间。


# 命令
- 展示所有task
`./gradlew tasks`

# Transform
- Gradle Transform是Android 官方提供给开发者在项目构建阶段（class->dex期间）用来修改.class文件的一套标准API