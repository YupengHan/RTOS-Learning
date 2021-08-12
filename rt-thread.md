# RT-thread 
1. 线程代码
1. 线程控制模块
    - 一种数据结构
    - 存放线程信息： 优先级，线程名称，线程状态，线程之间的链表关系，线程等待事件
1. 线程堆栈
    - 独立栈（stack）空间
    - 保存上下文：环境，变量，数据
    - 连续的内存空间

1. 创建线程
    1. rt_thread_init
    ``` C
    rt_err_t rt_thread_init(struct rt_thread *thread, //线程控制块
                            const char *name,
                            void(*entry)(void* parameter), //线程代码
                            void *parameter, // 0 for nothing
                            void *stack_start,  //线程栈起始地址
                            rt_uint32_t stack_size,
                            rt_uint8_t priority,
                            rt_uint32_t tick) //时间切片个数
    ```
    静态线程创建
    - 事先定义好，线程控制块和栈空间，所以是静态的
    - 使用环境：使用外部存储空间的时候
    1. rt_thread_create
    ```C
    rt_thread_t rt_thread_creat(const char *name,
                            void(*entry)(void* parameter), //线程代码
                            void *parameter, // 0 for nothing
                            rt_uint32_t stack_size,
                            rt_uint8_t priority,
                            rt_uint32_t tick) //时间切片个数
    ```
    动态线程创建

    - 不需要其实地址
    - 不需要输入线程控制块， 线程控制块为返回值（自动分配的）
    - 创建动态线程之后，还要调用
        ```C
        rt_err_t rt_thread_startup(rt_thread_t thread) //input是线程控制块的指针
        ```
        将创建的线程加入线程就绪队列，执行调度

# Simple Led Project
// 1.png  
初始状态 A;  
就绪状态 B;  
运行状态 C;  
挂起状态 D;   
关闭状态 E;   


A rt_thread_startup -> B
B rt_thread_suspend -> D
C rt_thread_exit    -> E
D rt_thread_detach  -> E
D rt_thread_delete  -> E

### 嘀嗒(心跳)时钟
- 定时器的定时中断
- 系统嘀嗒，时钟节拍
STM32 系统嘀嗒频率 100Hz， 每个嘀嗒10ms

### GPIO 驱动架构 操作IO 
    General purpose I/O?

```C
void rt_pin_mode(rt_base_t pin, //pin脚
                rt_t_mode);     //模式
MODE:
    PIN_MODE_OUTPUT //XX输出
    PIN_MODE_INPUT
    PIN_MODE_INPUT_PULLUP
    PIN_MODE_INPUT_PULLDOWN
    PIN_MODE_OUTPUT_OD //开路输出

// IO写入
void rt_pin_write(rt_base_t pin, rt_base_t value)
                                // PIN_HIGH,PIN_LOW


int rt_pin_read(rt_base_t pin) //读入
```

1. board.h 可以看到io管脚数量 STM32_F10X_PIN_NUMBERS (144)
1. 查看管脚号码：
    drv_gpio.c查看

1. list_thread 指令  
    查看每个thread的状态，stack pointer位置，stack size 和 max_use of stack
    thread stack size 是16 进制的

# Thread manager


- priority 
    - 256个优先级，0为最高优先级，最低级别预留给空闲线程
    - 我们可已通过rt_config.h中的RT_THREAD_PRIORITY_MAX宏来修改最大优先级
    - 针对STM32， 默认设置最大优先级为32
    - Note创建线程总数不限与256，同一优先级可以存在多个线程，能创建多少个线程只受限于stack mem和线程控制块的开销
- time tick 时间片参数
    - 只在相同优先级的线程中，有约束作用
    - 系统对 就绪状态 同优先级 的线程们， 采用时间片轮转的调度方式进行
    - 时间片决定单词运行时长，单位是系统节拍（OS Tick）
- 
    - 优先抢占调度
        - 当有thread的优先级高于当前thread的优先级，且处于就绪状态，就一定会发生thread调度
    - 通过优先级抢占调度，最大限度的满足了系统的实时性
    - 时间片轮询调度
        - 约束thread单次运行时长的调度
        - 同优先级任务能够轮流占有处理器
- !note
    - 经过测试，高优先级的程序，即便是要跑10min，也不会中途停下来跑一下低优先级的程序的
    - 真 尊贵的高优先级
# 空闲线程 及 两个常用的勾子函数
### 系统空闲线程 tidle
- list_thread 打印运行thread的控制命令，看到过rtshell 和 tridle这两个线程
- tshell tidle 系统线程
- 空闲线程 tidle
    - 最低优先级
    - 无其他在运行的线程的时候，调度器才过来
    - 负责资源回收，以及将一些处于关闭状态的线程从线程调度器列表中移除
    - 无线循环结构，永远不会被挂起
    - located at kernel/idle.c // rt-thread/src/idle.c
- tidle 提供了一些钩子函数 hook
    - tidle的勾子让系统在空闲的时候执行一些非紧急的事务
        - Eg： 系统运行指示灯闪烁
        -   CPU使用情况统计
### 钩子函数
- 设置
```C
rt_err_t rt_thread_idle_sethook(void(*hook)(void))

```
- 删除
```C
rt_err_t rt_thread_idle_delhook(void(*hook)(void))
input is a function

```
### 空闲线程的注意事项
- tidle永远处于就绪状态 钩子函数执行中
    相关代码必须保证空闲线程在任何时刻都不会被挂起
    rt_thread_delay()
    rt_sem_take()
    可能会导致线程挂起的阻塞类函数，都不能用于钩子函数
- 空闲线程可以使用多个钩子函数

### 系统调度钩子函数
- 系统的 上下文 切换是系统中常见事件
- 有时用户想知道某一时刻发生了什么样子的线程切换
- 系统调度钩子函数，在系统thread切换时运行，通过这个钩子，我们可以了解系统调度信息

```C
rt_scheduler_sethook(void(*hook)(struct rt_thread*from, struct rt_thread*to))
//这个函数两个输入，from 和 to
```
- 系统调度器的钩子函数，只能设置一个
- tidle的钩子函数，default可以设置4个
# Q 
    - volatile static int hook_times = 0; volatile static这是啥？

# 临界区保护

### 临界资源
- 临界资源 一次只允许一个thread访问的共享资源，可是是一个硬件设备（Eg：打印机），变量，缓冲区
- 多thread互斥的对临界资源进行访问
### 临界区
- 每个线程中 访问资源 的 那段代码 叫 临界区(critical section)
- 每次只允许一个线程进入临界区 

### RT-thread 中 临界区保护的方式
1. 关闭系统调度
    把调度器锁住，不让其进行线程切换，保证当前运行的任务不被换出
    直到把调度器解锁（常用临界区保护方法）
    - 禁止调度
        ```C        
        void thread_entry(void parameter) {
            while(1) {
                /*调度器上锁，上锁后不在切换其他线程，仅中断响应*/
                rt_enter_critical();

                /*临界区代码*/
                ...
                /*调度器解锁*/
                rt_exit_critical();
            }
        }
        ```
    - 关闭中断
        - 调度是建立在中断的基础上，当我们关闭了中断之后，系统将不能再进行调度，线程自身也自然不会被其他线程占用
        ```C
            void thread_entry(void* parameter) {
                // rt_base_t level; 
                rt_uint32_t level;
                while(1) {
                    /*关闭中断*/
                    level = rt_hw_interrupt_disable();
                    /*以下是临界区*/
                    ...
                    /*恢复中断*/
                    rt_hw_interrupt_enable(level);
                }
            }
        ```
        - !Note关闭中断之后，外部中断也会停止响应

1. 互斥特性保护临界区
    - 信号量
    - 互斥量

# 信号量的使用
### IPC Internal Process Communication 线程间通讯
-
    - 嵌入式系统运行的代码，包括
        1. 线程
        1. ISR 中断
    -  thread ISR 之间，需要进行如下通信
        1. 在运行的时候， 有时候步骤需要同步（按照预定的先后次序运行）
        1. 资源需要互斥
        1. 数据交互
        - 操作系统需要提供机制 完成上述三个功能
    - RT-thread 的 IPC
        1. 信号量
        1. 互斥量
        1. 事件
        1. 邮件
        1. 消息队列
        - 通过IPC，我们可以协调多线程（包括ISR）之间工作
### 信号量
- 
    - 线程 通过 信号量的
