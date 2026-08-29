---
title: FreeRTOS 队列：任务之间怎么安全地"递东西"（信号量竟然也是队列）
tags: [freertos, rtos, queue, 嵌入式]
categories: [嵌入式]
---

上一篇 [《FreeRTOS 核心原理》](/2026/08/24/freertos-core-principles.html) 结尾留了句话：中断里要通知任务，用"置标志位 / 放队列 / 唤醒一个任务"。前一个你已经会了--全局变量加标志位，裸机时代人人写过。但先看一个几乎每个 RTOS 新手都踩过的坑：

两个任务，一个生产数据，一个消费数据。最直觉的写法：一个全局数组，加一个 `data_ready` 标志：

```c
uint8_t buf[64];
volatile int data_ready = 0;

/* 生产者 */
void producer(void) {
    fill(buf);
    data_ready = 1;
}

/* 消费者 */
while (!data_ready) { }
data_ready = 0;
process(buf);
```

这段代码在演示板上跑得好好的，直到某天现场出现一种死法：消费者刚把 `data_ready` 清零、还没开始读 `buf`，调度器一个切换，生产者又写了一半进去，消费者读到的半是新半是旧。你加 `volatile`、调优先级、加延时，问题像幽灵一样时有时无。

根因不在于哪行写错了，而在于"**写数据**"和"宣布数据好了"是两个动作，中间任何时刻都可能被切开。你需要一个原子性由内核担保的东西：既存数据，又管"什么时候给你"。这就是队列（Queue）。

<!--more-->

## L1：先用起来--三个函数构成的生产-消费

你在第一层。目标：把上一节的竞态代码换成队列版，顺便记住一组接口。

```c
#include "FreeRTOS.h"
#include "queue.h"

QueueHandle_t xQueue;

int main(void)
{
    /* 长度 10，每条消息是一个 uint32_t */
    xQueue = xQueueCreate(10, sizeof(uint32_t));
    configASSERT(xQueue != NULL);      /* 堆不够时返回 NULL，别当没看见 */

    xTaskCreate(vProducer, "prod", 128, NULL, 2, NULL);
    xTaskCreate(vConsumer, "cons", 128, NULL, 1, NULL);
    vTaskStartScheduler();
    for (;;) { }
}

void vProducer(void *pv)
{
    uint32_t seq = 0;
    for (;;) {
        /* 第 4 个参数：队列满时最多等多久（tick 数） */
        if (xQueueSend(xQueue, &seq, 100) != pdPASS) {
            /* 等 100 个 tick 还满着 -- 丢数据？报警？自己决定 */
            LED_ERROR_ON();
        }
        seq++;
        vTaskDelay(10 / portTICK_PERIOD_MS);
    }
}

void vConsumer(void *pv)
{
    uint32_t got;
    for (;;) {
        /* 队列空就阻塞，等到天荒地老 */
        xQueueReceive(xQueue, &got, portMAX_DELAY);
        printf("got %lu\n", (unsigned long)got);
    }
}
```

对照裸机版，有三个质变：

1. **数据进队列了**。`xQueueSend` 把 `seq` 的**内容**拷进队列自己的存储区，消费者 `xQueueReceive` 再拷出来。生产者写完就可以随便改局部变量，不存在"读一半被覆盖"。
2. **消费者不用轮询了**。队列空时 `xQueueReceive` 直接把任务挂起（阻塞），CPU 一个周期都不花在 `while (!data_ready)` 上。注意 `portMAX_DELAY` 那个参数就是干这个的--上一篇讲过的阻塞语义，这里队列替你用了。
3. **满了会等**。`xQueueSend` 的第 4 个参数和 receive 对称：队列满时生产者阻塞多久。给个有限值（如 100 tick）意味着"满太久就算失败"，返回 `pdFAIL`，你可以选择报警或丢帧；给 `portMAX_DELAY` 就是死等。

内存账也简单：队列总开销 ≈ **长度 × 单条大小 + 一个七八十字节的管理结构**（`Queue_t`，内含指针和链表头）。这钱从 FreeRTOS 的堆（`heap_4` 那个）里出。想要完全静态分配，用 `xQueueCreateStatic`。

**中断要用另一套门**。ISR 里没有任务上下文，不能"阻塞"，必须走 `FromISR` 版本：

```c
void EXTI0_IRQHandler(void)
{
    BaseType_t xWoken = pdFALSE;
    uint32_t event = 0x1000;

    xQueueSendFromISR(xQueue, &event, &xWoken);
    /* 若唤醒的任务优先级更高，退出 ISR 时立刻切过去，别等下个 tick */
    portYIELD_FROM_ISR(xWoken);

    EXTI->PR = 0x1000;     /* 清挂起位 */
}
```

那个 `xWoken` 出参就是 [《中断》](/2026/08/24/interrupt-cortex-m4.html) 那篇说的"ISR 的唯一正经工作：安全地高优先级唤醒"的落点。忘了 `portYIELD_FROM_ISR` 不会死机，只是唤醒推迟到下个 tick--毫秒级迟滞，极难查。

到这里你已经能把任何"裸机标志位"改成队列了。但几个问题悬着：队列内部到底长什么样？空时"阻塞"了，谁在数据到达的那一刻叫醒它？

## L2：懂原理--环形缓冲加两条等待链表

到这一层，把队列拆开看。FreeRTOS 的队列在 `queue.c` 里，本体是一个结构体加一块连续存储：

```c
/* 简化自 queue.c 的 Queue_t，字段名保留 */
typedef struct QueueDefinition {
    int8_t *pcHead;                /* 存储区起点 */
    int8_t *pcWriteTo;             /* 下一次 send 写到哪 */
    int8_t *pcReadFrom;            /* 下一次 receive 从哪读 */
    volatile UBaseType_t uxMessagesWaiting;   /* 现在里面有几条 */
    UBaseType_t uxLength;          /* 总容量 */
    UBaseType_t uxItemSize;        /* 每条多少字节 */
    volatile int8_t cRxLock, cTxLock;          /* 中断安全用的锁计数，L3 讲 */
    List_t xTasksWaitingToSend;    /* 队列满了、排队等着塞的发送者 */
    List_t xTasksWaitingToReceive; /* 队列空了、排队等着取的接收者 */
} Queue_t;
```

存储区就是一个**环形缓冲**：`pcWriteTo` 往前走，走到尾部绕回 `pcHead`；`pcReadFrom` 在后面追。加法和绕回都在 `vTaskSuspendAll()` 保护下做--所以"塞一条"这个动作天然互斥。

真正的主角是那两条**等待链表**。上一篇讲过：任务阻塞 = 从就绪链表摘下来，挂到某个"等事件"的链表上，TCB 里的状态改成 blocked。队列的阻塞就挂在这两条链表上，方向相反：

**接收方阻塞的全程：**

1. 消费者调 `xQueueReceive`，发现 `uxMessagesWaiting == 0`。
2. 内核把它挂到 `xTasksWaitingToReceive`，记录"我等到 tick N"，然后切换走。此刻消费者**不消耗任何 CPU**。
3. 生产者调 `xQueueSend`：`memcpy` 一条进环形缓冲，`uxMessagesWaiting` 加一。
4. 发现有任务在 `xTasksWaitingToReceive` 里--取**优先级最高**的那个（同优先级取等得最久的），把它摘下链表、放回就绪表。
5. 如果被唤醒者优先级高于当前正在跑的任务，直接触发 PendSV 立刻换人。这一步在 `xQueueSendFromISR` 里就是那个 `xWoken` 出参干的事。

发送方满时阻塞是对称的：`xQueueSend` 发现队列满，挂到 `xTasksWaitingToSend`；任何一次 `xQueueReceive` 取走数据后会唤醒它。所以队列是个双向阀门，两头都能等。

注意第 3 步那个 `memcpy`：**FreeRTOS 队列传的是值，不是引用。** `xQueueSend(xQueue, &seq, ...)` 传的是 `&seq`，但内核做的是"把 `seq` 指向的内容拷进队列"；`xQueueReceive` 再拷出来。传地址只是"把货搬进仓库"的动作需要，不是把仓库里放了个地址。这一个设计决定消灭了整整一类 bug--下面 L3 专门说为什么。

还有个容易漏看的细节：`uxMessagesWaiting` 的读、写指针的挪动，全部发生在关调度/临界区里，**但只保护指针记账那几条指令**，`memcpy` 大块数据时也持有锁吗？老版本里 send/receive 的拷贝在挂起调度期间完成，拷贝大结构体会拉长这段不可抢占窗口--这是队列传大数据的真实代价，收尾地图会给出路。

现在你能回答"谁在数据到达那一刻叫醒它"了：**生产者的 `xQueueSend` 自己就是闹钟**。没有后台线程，没有轮询，通知成本折进了每次 send 里。但更深一层的问题是：为什么这个设计要长成"拷贝值"？以及标题埋的那个雷--信号量凭什么也是队列？

## L3：想得透--拷贝、特例与那把"锁"

**为什么按值拷贝。** 设想按引用：`xQueueSend` 只存指针。那么生产者 `xQueueSend(&seq)` 返回后，`seq` 是个局部变量，函数返回栈就回收，或者下一轮循环 `seq++` 把它改了。消费者拿到的指针指向什么？不是悬空就是脏数据。你得自己管理生命周期：静态缓冲、引用计数、用完通知生产者回收……等于把内核该管的事全背回自己肩上。按值拷贝的潜台词是：**数据所有权随 send 移交，之后你随便改你的，我拿我的**。内存安全换一点拷贝开销，对嵌入式这种"消息几十字节"的量级，几乎总是划算的。

那要传大数据怎么办？两条正路：传**指向堆/静态缓冲的指针**（消息本身只有 4 字节，但生命周期由你担保--谁 free、何时 free，得约定好，常用"接收方用完归还空闲链表"的池子模式）；或改用 v10 之后的 **StreamBuffer / MessageBuffer**（变长、零拷贝语义更贴合）。传 1KB 的结构体按值塞队列，RAM 和禁调度窗口都会替你付账。

**信号量是队列的特例。** 打开 `semphr.h` 看一眼就穿帮了：

```c
#define xSemaphoreCreateBinary() \
    xQueueGenericCreate( ( UBaseType_t ) 1,   /* 长度 1   */ \
                         semSEMAPHORE_QUEUE_ITEM_LENGTH, /* = 0 */ \
                         queueQUEUE_TYPE_BINARY_SEMAPHORE )
```

一个**长度 1、每条 0 字节**的队列。`xSemaphoreGive` 就是往里 send 一团 0 字节的"空气"，`xSemaphoreTake` 就是 receive--空了就阻塞。这就是为什么二进制信号量能当"事件通知"用：队列的"有/无"被压缩成一比特，数据本体被抽掉了。计数信号量同理，长度放宽到 N 而已。

互斥锁（`xSemaphoreCreateMutex`）稍进一步：底层还是这个队列，但 `queueQUEUE_TYPE_MUTEX` 让内核多干一件事--**优先级继承**。低优先级任务持锁时被临时提到等锁任务里最高者的优先级，防上一篇收尾地图提过的"优先级反转"死锁。同一条血脉，加了一个只在"锁"语义下才说得通的功能。

（顺带把容易混的近亲排个座：**事件组**和**任务通知**不是队列变体--事件组是独立的一排位；任务通知直接写对方 TCB 里的字段，不走任何队列结构，所以更快，但只能点对点。）

**那把"锁"到底锁什么。** L2 说指针记账在临界区里，但 ISR 也会调 `xQueueSendFromISR`，中断随时可能劈进任务版的临界区中间--链表操作又不是可重入的，怎么办？看 `cRxLock` / `cTxLock` 这对计数字段：任务侧进入队列操作前先把队列"锁上"（suspend 调度器），这期间若 ISR 来操作队列，内核不碰等待链表，只把"该唤醒/该记账几次"**攒**在这两个计数里；等任务侧退出临界区，再一次性补做链表动作（`prvUnlockQueue` 干的活）。也就是"中断不能等，那就先记账后算账"。这是整个队列实现里最精巧的一段，也是为什么 `FromISR` 接口绝不能用任务版替代--任务版的"挂起调度等锁"在 ISR 里就是死机。

## 动手环节：UART 接收的经典套路

把三篇串起来：[《UART》](/2026/08/25/uart-serial-cortex-m4.html) 那篇的接收，加上队列和中断，就是嵌入式里最经典的"ISR 只递条、任务干活"模式。在 STM32F4 上（HAL 库）：

```c
#define QLEN 16
static QueueHandle_t xUartRxQ;
static uint8_t rxb;

void uart_task_init(void)
{
    xUartRxQ = xQueueCreate(QLEN, sizeof(uint8_t));
    HAL_UART_Receive_IT(&huart1, &rxb, 1);   /* 挂第一次中断接收 */
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    BaseType_t w = pdFALSE;
    BaseType_t in_isr = (__get_IPSR() != 0);   /* 保险：确认真在 ISR 里 */

    if (in_isr)
        xQueueSendFromISR(xUartRxQ, &rxb, &w);
    else
        xQueueSend(xUartRxQ, &rxb, 0);          /* HAL 也可能任务态回调 */

    portYIELD_FROM_ISR(w);
    HAL_UART_Receive_IT(&huart1, &rxb, 1);     /* 再挂一次 */
}

void vUartParser(void *pv)
{
    uint8_t ch;
    char line[80];
    int i = 0;

    for (;;) {
        xQueueReceive(xUartRxQ, &ch, portMAX_DELAY);   /* 没字符就一直睡 */
        if (ch == '\n' || i >= sizeof(line) - 1) {
            line[i] = '\0';
            handle_line(line);
            i = 0;
        } else {
            line[i++] = ch;
        }
    }
}
```

值得盯住的三个点：

- **ISR 里只有 4 行**：塞队列、请求切换、重挂接收。解析、拼行、回包全在任务里，随便阻塞、随便慢，不占中断延迟预算。
- `w` 用 `portYIELD_FROM_ISR` 收尾，解析任务被立刻调度，吞吐和响应兼得。
- 队列长度 16 = 你给突发的容忍度。解析任务卡死 17 个字符后，新数据从 `xQueueSendFromISR` 返回 `errQUEUE_FULL` 静默丢--**所以生产环境必须查这个返回值**，丢包计数器是最好的心跳指标。

调试期给自己加两个探针：`uxQueueMessagesWaiting(xUartRxQ)` 随时看积压（持续逼近 QLEN 说明消费太慢）；`uxQueueSpacesAvailable` 看剩余。这两个函数中断里也能调（各有 FromISR 版本）。

## 收尾地图

| 你要干什么 | 用什么 | 一句话理由 |
|---|---|---|
| 任务间传数据（定长消息） | 队列 | 拷贝即所有权移交，阻塞唤醒内建 |
| 中断向任务递数据 | 队列 + `FromISR` + `portYIELD_FROM_ISR` | ISR 只递条不干活 |
| 传大数据（>几十字节） | 指针入队 + 缓冲池，或 MessageBuffer | 省拷贝和禁调度窗口 |
| 只要"事件发生了"这一比特 | 二进制信号量 | 长度 1、条目 0 字节的队列 |
| 计数资源（N 个坑位） | 计数信号量 | 长度 N 的"空气"队列 |
| 保护共享资源 | 互斥锁（**永远别在 ISR 里 take**） | 队列特例 + 优先级继承 |
| 一对一点对点、极致性能 | 任务通知（`xTaskNotify`） | 直写 TCB，不走队列 |
| 一个任务等多个来源 | 队列集（`xQueueCreateSet`） | 把多条队列汇成一个等待点 |

四个最常见的死法，收进一张便签：

1. **ISR 里调了任务版接口**（`xQueueSend` 而非 `xQueueSendFromISR`）--内部"挂起调度再等锁"，中断里等于自杀。`configASSERT` 开着的话会当场断言，没开就是玄学死机。
2. **塞指针指向栈变量**--拷贝语义被你亲手废掉，出队时栈帧早没了。
3. **不看 send 的返回值**--队列满时数据是**丢**的，不是等的（除非你给了阻塞时长）。
4. **互斥锁跨 ISR**--ISR 里不能阻塞等锁，`xSemaphoreTake` 的互斥锁版本直接被断言拦下；中断里只许用二进制信号量发"事件"。

还有一条经验值：队列深度别拍脑袋设大。它赌的是"消费速度跟得上"这个假设--深度 16 掩盖不了的积压，深度 160 只是让它晚 10 倍爆。先用小队列逼出消费端瓶颈，再用 `uxQueueMessagesWaiting` 的曲线说话。

下一站还是那两条路：往深处走，去读 `queue.c` 的 `prvUnlockQueue` 和 `xQueueGenericSend`，对照 L2/L3 的伪代码看真实实现怎么处理"锁着的时候来了中断"；往宽处走，去看任务通知（`taskNOTIFICATION`）为什么在点对点场景能把队列甩开一个数量级，以及队列集怎么让一个任务优雅地同时等 UART、SPI 和一条网络命令。到那一步，"任务间通信"在你眼里就不再是 API 列表，而是同一套"存储 + 等待链表 + 记账"的三件套，变着花样组合。
