## 1. 通信：把“数据”传过去

例如：

- 传感器任务采集温度 `25`
- 显示任务需要把 `25` 显示出来

这时候传递的是**具体数据**  
所以适合用：

- **Queue 队列**

---

## 2. 同步：告诉别人“你现在可以干活了”

例如：

- 按键中断来了
- 通知某个任务：“有按键事件发生了”

这里不一定要传复杂数据，只是发一个“信号”  
所以适合用：

- **Semaphore 信号量**

---

## 3. 互斥：共享资源不能同时乱用

例如：

- 任务A打印串口
- 任务B也打印串口

如果两个任务同时往串口发数据，就可能变成乱码：
```c
TaskA: Hello
TaskB: World
```

`HeWo lrllod`
所以需要一个“锁”：

- 谁先拿到锁，谁先用
- 用完再释放
- 另一个任务再用

## 3. 队列的核心函数

### 创建队列

`xQueueCreate(队列长度, 每个元素大小)`

例如：

```c
QueueHandle_t xTempQueue;  
xTempQueue = xQueueCreate(5, sizeof(int));
```

意思是：

- 这个队列最多放 5 个元素
- 每个元素大小是 `int`

---

### 发送数据到队列

```c
xQueueSend(xTempQueue, &temp, portMAX_DELAY);
```

意思是：

- 把 `temp` 这个数据送入队列
- portMAX_DELAY如果队列满了，就一直等

---

### 从队列接收数据

```c
xQueueReceive(xTempQueue, &recvTemp, portMAX_DELAY);
```

意思是：

- 从队列取一个数据出来
- 如果队列空了，就一直等

## 二值信号量 Binary Semaphore

所谓二值，就是只有两种状态：

- 有信号
- 没信号

就像一个开关：

- 0：没有事件
- 1：有事件

## 4. 关键函数

### 创建二值信号量

`xSemaphoreCreateBinary()`

### 释放信号量（给信号）

`xSemaphoreGive()`

### 获取信号量（等信号）

`xSemaphoreTake()`