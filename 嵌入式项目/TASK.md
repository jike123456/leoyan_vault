|当前状态|触发事件|下一状态|
|---|---|---|
|Ready|被调度器选中|Running|
|Running|调用 `vTaskDelay()`|Blocked|
|Running|等队列/信号量/事件|Blocked|
|Blocked|时间到 / 事件到|Ready|
|Running|被更高优先级任务抢占|Ready|
|Running|被挂起 `vTaskSuspend()`|Suspended|
|Suspended|被恢复 `vTaskResume()`|Ready|
|Running|被删除 `vTaskDelete()`|Deleted|
