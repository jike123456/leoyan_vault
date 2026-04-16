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

Blocked -> Ready -> Running

```c
void TaskA(void *argument)
{
    for (;;)
    {
        printf("A\r\n");
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void TaskB(void *argument)
{
    for (;;)
    {
        printf("B\r\n");
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

xTaskCreate(TaskA, "TaskA", 128, NULL, 1, NULL);
xTaskCreate(TaskB, "TaskB", 128, NULL, 2, NULL);
```