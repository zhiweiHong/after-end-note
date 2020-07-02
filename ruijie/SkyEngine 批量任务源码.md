# `SkyEngine` 序列批处理任务源码解读——任务运行流程

​		由于没有源码文件，笔者是通过idea的反编译功能查看的。代码上可能有些许出入，然而大体的方向不会有太多的偏差。

​		根据文档示例，想要在调用批处理任务，需要在Controller方法中注入`BatchTaskBuilder`。示例如下：

![image-20200616170935324](.\image-20200616170935324.png)

​		`BatchTaskBuilder`是平台对外开放的一个接口，主要用于提供构造`BatchTask`功能。重点看start方法。

​		`BatchTaskBuilder`的默认实现类是`DefaultBatchTaskBuilder`。

​		而`DefaultBatchTaskBuilder`的start()方法如下所示。

```java
public BatchTaskSubmitResult start() throws BusinessException {
    this.checkBatchTask();
    LOGGER.info("开始执行批处理任务, 当前任务身份标识为[{}],任务名[{}],参数名[{}],是否开启并行处理[{}] ", new Object[]{this.identityId, this.taskNameKey, this.taskDescKey, this.enableParallel});
    // 构造任务实体，无外乎是任务名称，任务描述，是否开启并行等属性。这个类后续会被转换为其他的类。
    BatchTaskDefinition batchTaskDefinition = this.createBatchTaskDefinition();
    // 判断任务运行的策略。此处任务有两种运行策略，平行和串行。
    BatchTaskRunner batchTaskRunner = this.determineTaskRunner();
    LOGGER.debug("开始提交任务到批处理任务处理器：[{}]", batchTaskRunner.batchTaskRunnerName());
    // 运行任务。
    BatchTaskSubmitResult defaultBatchTaskSubmitResult = batchTaskRunner.runTask(batchTaskDefinition, this.batchTaskHandler, this.batchTaskNotifier);
    LOGGER.debug("完成任务提交，批处理任务标识：[{}]", defaultBatchTaskSubmitResult.getTaskId());
    return defaultBatchTaskSubmitResult;
}
private BatchTaskRunner determineTaskRunner() {
    return this.enableParallel ? this.parallelBatchTaskRunner : this.serialBatchTaskRunner;
}
```

​		可以看到，这套并行任务支持两种模式，串行和并行，分别对应的实现是`SerialBatchTaskRunner`和`ParallelBatchTaskRunner`

。而这两个类都继承了`AbstractBatchTaskRunner`。`batchTaskRunner.runTask`默认调用的是`AbstractBatchTaskRunner`的方法。

```java
public BatchTaskSubmitResult runTask(BatchTaskDefinition batchTaskDefinition, BatchTaskHandler batchTaskHandler, BatchTaskNotifier batchTaskNotifier) throws BusinessException {
        Assert.notNull(batchTaskDefinition, "batchTaskDefinition can not be null");
        Assert.notNull(batchTaskHandler, "batchTaskHandler can not be null");
        Assert.notNull(batchTaskNotifier, "batchTaskNotifier can not be null");
    	// 此处使用了包装对象，将BatchTaskDefinition转换为BatchTaskEntity。
        // 实际上，batchTaskHandler在包装类里没有被用到。
        BatchTaskWrapper batchTaskWrapper = this.wrap(batchTaskDefinition, batchTaskHandler);
        // 插入数据库，返回数据库主键
        UUID taskId = batchTaskNotifier.beforeBatchTaskSubmit(batchTaskWrapper.getBatchTaskEntity());
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("完成任务准备，生成的任务ID为:{}", taskId);
        }

        LOGGER.info("使用[{}]处理任务[taskId:{}]", this.batchTaskRunnerName(), taskId);
        // 包装产生任务所需要的数据
        TaskDataPackage taskDataPackage = new TaskDataPackage();
        taskDataPackage.setBatchTaskNotifier(batchTaskNotifier);
        taskDataPackage.setBatchTaskHandler(batchTaskHandler);
        taskDataPackage.setBatchTaskWrapper(batchTaskWrapper);
        taskDataPackage.setEnablePerformanceMode(batchTaskDefinition.isEnablePerformanceMode());
        taskDataPackage.setPerformanceModeThreadNum(batchTaskDefinition.getPerformanceModeThreadCount());
        taskDataPackage.setEnableSynchronousMode(batchTaskDefinition.isEnableSynchronousMode());
        taskDataPackage.setTaskId(taskId);
        taskDataPackage.setTaskPoolId(batchTaskDefinition.getTaskPoolId());
        // 执行任务。
        this.doRunTask(taskDataPackage);
        String taskDesc = batchTaskDefinition.getTaskDesc();
        String taskName = batchTaskDefinition.getTaskName();
        BatchTaskStatus taskStatus = batchTaskWrapper.getBatchTaskEntity().getTaskStatus();
        BatchTaskSubmitResultDTO batchTaskSubmitResultDTO = new BatchTaskSubmitResultDTO(taskId, taskName, taskDesc, taskStatus);
        return batchTaskSubmitResultDTO;
    }
```

​		下面是`SerialBatchTaskRunner`的`doRunTask`方法。

```java
protected void doRunTask(TaskDataPackage taskDataPackage) {
        BatchTaskHandler batchTaskHandler = taskDataPackage.getBatchTaskHandler();
        Assert.notNull(batchTaskHandler, "batchTaskHandler must not be null");
        BatchTaskNotifier batchTaskNotifier = taskDataPackage.getBatchTaskNotifier();
        Assert.notNull(batchTaskNotifier, "batchTaskNotifier must not be null");
        BatchTaskWrapper batchTaskWrapper = taskDataPackage.getBatchTaskWrapper();
        Assert.notNull(batchTaskWrapper, "batchTaskWrapper must not be null");
        AtomicInteger successNum = new AtomicInteger(0);
        AtomicInteger totalNum = new AtomicInteger(0);
        BatchTaskEntity batchTaskEntity = batchTaskWrapper.getBatchTaskEntity();
    	// 主要任务执行前置器。其实也是真正执行任务的所在。
        MajorTaskPreProcessor majorTaskPreProcessor = () -> {
            while(batchTaskHandler.hasNext()) {
                BatchTaskItem item = (BatchTaskItem)batchTaskHandler.next();
 				// 执行小任务          
                MinorTask.builder().batchTaskEntity(batchTaskEntity).batchTaskHandler(batchTaskHandler).batchTaskNotifier(batchTaskNotifier).successNum(successNum).totalNum(totalNum).batchTaskItem(item).minorTaskCallback(() -> {
                }).build().run();
            }

        };
        Builder builder = MajorTask.builder();
    	// 性能模式相关的操作
        if (taskDataPackage.isEnablePerformanceMode()) {
            LOGGER.info("批处理任务[{}]开启性能模式, 任务名称:[{}], 准备生成自定义线程池", taskDataPackage.getTaskId(), taskDataPackage.getBatchTaskWrapper().getBatchTaskEntity().getTaskName());
            builder = this.setupPerformanceMode(builder, taskDataPackage.getTaskId(), taskDataPackage.getTaskPoolId(), taskDataPackage.getPerformanceModeThreadNum());
        }
		// 总任务
        MajorTask serialTask = builder.batchTaskEntity(batchTaskEntity).batchTaskHandler(batchTaskHandler).batchTaskNotifier(batchTaskNotifier).successNum(successNum).totalNum(totalNum).majorTaskPreProcessor(majorTaskPreProcessor).build();

        try {
            Runnable serialTask1 = TtlWrapper.wrapRunnable(serialTask);
            if (taskDataPackage.isEnableSynchronousMode()) {
                // 判断是否启用同步模式
                serialTask1.run();
            } else {
                // 否则线程池提交任务。
                this.decideThreadPool(taskDataPackage.isEnablePerformanceMode()).submit(serialTask);
            }

            this.performanceThreadPoolExecutor = null;
        } catch (Throwable var12) {
            this.onSubmitFail(batchTaskEntity, batchTaskNotifier, var12);
            return;
        }

        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("完成批处理任务提交,本次一共提交{}任务", batchTaskWrapper.getItemWrapperList().size());
        }

    }
```

​		这段代码里有两个核心的任务类。`MinorTask`和`MajorTask`。

​		先看`MajorTask`的run方法。

```java
public void run() {
    Object batchTaskItemResult = null;

    try {
        // 插入数据库，记录子任务。
        this.batchTaskNotifier.processingBatchTaskItem(this.batchTaskEntity, this.itemEntity);
        // 用户自定义任务执行相关。
        batchTaskItemResult = this.batchTaskHandler.processItem(this.batchTaskItem);
        Assert.notNull(batchTaskItemResult, "batchTaskItemResult can not be null");
        if (((BatchTaskItemResult)batchTaskItemResult).getItemStatus() == BatchTaskItemStatus.SUCCESS) {
            // 更新成功任务数
            this.successNum.incrementAndGet();
        }
    } catch (Exception var14) {
        Exception e = var14;
        LOGGER.error("任务执行失败,任务信息：" + JSON.toJSONString(this.itemEntity), var14);

        try {
            this.batchTaskHandler.afterException(this.batchTaskItem, e);
            batchTaskItemResult = this.buildExceptionResult(e);
        } catch (Throwable var13) {
            LOGGER.error("afterException error ", var13);
            batchTaskItemResult = this.buildExceptionResult(var13);
        }
    } finally {
        try {
            // 更新任务总数
            this.totalNum.incrementAndGet();
            // 更新子任务状态
            this.batchTaskNotifier.afterBatchTaskItemComplete(this.itemEntity, (BatchTaskItemResult)batchTaskItemResult);
        } catch (Throwable var12) {
            LOGGER.error("afterBatchTaskItemComplete error", var12);
            this.batchTaskNotifier.afterBatchTaskItemException(this.itemEntity, var12);
        }
		// 提供回调接口
        this.minorTaskCallback.after();
    }

}
```

​		这里面有一点需要注意。`totalNum`和`successNum`都是由外部传入的，都是`AtomicInteger`，且默认值为0。由于传入的`BatchTaskHandler`继承了Iterator类，无法计数。只能通过迭代器依次迭代来计数。

​		`MajorTask`主要是增强了任务执行的功能。真正执行任务的是`MajorTaskPreProcessor`类。

```java
public void run() {
    Object batchTaskFinishResult = null;

    try {
        Assert.notNull(this.majorTaskPreProcessor, "majorTaskPreProcessor must not be null");
        LOGGER.debug("开始执行批处理任务[{}]前置处理器", JSON.toJSONString(this.batchTaskEntity));
        this.majorTaskPreProcessor.preProcess();
        int failCount = this.totalNum.get() - this.successNum.get();
        LOGGER.info("完成批处理任务，本次一共提交[{}]条任务，其中成功[{}]条", this.totalNum.get(), this.successNum.get());
        // 回调用户自定义方法
        batchTaskFinishResult = this.batchTaskHandler.onFinish(this.successNum.get(), failCount);
    } catch (Exception var11) {
        LOGGER.error("任务执行异常, 任务信息为：" + JSON.toJSONString(this.batchTaskEntity), var11);
        batchTaskFinishResult = DefaultBatchTaskFinishResult.builder().batchTaskStatus(BatchTaskStatus.FAILURE).msgKey("base-alarm_batch_task_execute_error").msgArgs(new String[]{var11.getLocalizedMessage()}).build();
    } finally {
        try {
            // 更新数据库状态
            this.batchTaskNotifier.afterBatchTaskComplete(this.batchTaskEntity, (BatchTaskFinishResult)batchTaskFinishResult);
            if (this.majorTaskPostProcessor != null) {
                LOGGER.debug("开始执行批处理任务[{}]后置处理器", JSON.toJSONString(this.batchTaskEntity));
                // 后置任务处理器
                this.majorTaskPostProcessor.postProcess();
            }
        } catch (Throwable var10) {
            LOGGER.error("afterBatchTaskComplete error ：batchTaskEntity=" + JSON.toJSONString(this.batchTaskEntity), var10);
            this.batchTaskNotifier.afterBatchTaskException(this.batchTaskEntity, var10);
        }

    }

}
```

​		至于` Runnable serialTask1 = TtlWrapper.wrapRunnable(serialTask);`这也算是一个包装器，增强类的功能。

​		至此，整个`SerialBatchTaskRunner`的运行流程结束。