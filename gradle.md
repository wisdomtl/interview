# gradle 生命周期
1. Initialization (初始化)
    - 决定构建中包含哪些项目，并构建对应的project实例。
2. Configuration (配置)
    - 生成task的依赖图（DAG,有向无环图），应用插件，注册task
3. Execution (执行)
    - 执行所需的task


