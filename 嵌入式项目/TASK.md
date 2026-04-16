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

- TaskA：Blocked
- TaskB：Blocked

系统会运行 **Idle Task**

# 练习5：高优先级任务被唤醒时会发生什么

先设定场景：

- `TaskA` 优先级 1，当前正在 Running
- `TaskB` 优先级 3，之前一直因为等待消息而 Blocked
- 现在消息到了，`TaskB` 从 Blocked 变成 Ready

## 问题

这时谁运行？`TaskA` 状态会变什么？

---

## 标准答案

- `TaskB` 会开始运行
- `TaskA` 从 Running 变成 Ready

