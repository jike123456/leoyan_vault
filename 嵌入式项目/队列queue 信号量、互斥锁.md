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

1. **二值信号量**
    
    - 状态从 0 → 1
    - 会唤醒一个正在 `xSemaphoreTake()` 等待该信号量的任务
2. **计数信号量**
    
    - 计数值 +1
    - 计数值达到最大值时再 give 会失败

### 获取信号量（等信号）

`xSemaphoreTake()`

```c
// 任务A：释放信号量 
xSemaphoreGive(xBinarySem);

// 任务B：等待信号量 
if(xSemaphoreTake(xBinarySem, portMAX_DELAY) == pdTRUE) {
	// 被唤醒后执行 
}
```

## 1.为什么需要互斥锁

多个任务可能会访问同一个共享资源：

串口
LCD
全局结构体
文件系统
I2C/SPI 总线

如果同时访问，就会冲突

## 3. 互斥锁的核心函数

### 创建互斥锁

`xSemaphoreCreateMutex()`

注意：虽然名字里有 `Semaphore`，但它创建的是 **Mutex**

### 获取锁

`xSemaphoreTake(xMutex, portMAX_DELAY);`

### 释放锁

`xSemaphoreGive(xMutex);`

```c
SemaphoreHandle_t xMutex; // 互斥量句柄

// 1. 创建互斥量（在初始化函数中执行）
void vCreateMutex( void )
{
    xMutex = xSemaphoreCreateMutex();
    if( xMutex == NULL )
    {
        // 处理创建失败（如打印日志、触发告警）
    }
}

// 2. 任务中使用互斥量保护共享资源
void vTaskFunction( void * pvParameters )
{
    for( ;; )
    {
        // 3. 获取互斥量，无限等待
        if( xSemaphoreTake( xMutex, portMAX_DELAY ) == pdTRUE )
        {
            // 4. 临界区：访问共享资源（如串口、传感器、共享变量）
            // ...

            // 5. 释放互斥量（必须执行，否则死锁）
            xSemaphoreGive( xMutex );
        }
    }
}
```

- **队列是用来传数据的**
- **信号量是用来发通知的**
- **互斥锁是用来保护共享资源的**
- **队列里的数据是复制进去的**
- **任务没事干时应该阻塞等待，而不是死循环空转**
- **中断里少做事，通常只发信号，具体处理放到任务里**

生产者任务  --->  队列  --->  消费者任务

- ADC任务 → 队列 → 滤波任务
- 串口接收任务 → 队列 → 协议解析任务
- 按键扫描任务 → 队列 → 控制任务