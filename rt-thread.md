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
    
    1. 将创建的线程加入线程就绪队列，执行调度
        ```C
        rt_err_t rt_thread_startup(rt_thread_t thread) //input是线程控制块的指针
        ```
    

# Simple Led Project
<!-- // 1.png   -->
- <img src="https://raw.githubusercontent.com/YupengHan/RTOS-Learning/main/imgs/1.png" width = "600" height = "200" alt="图片名称" align=center />
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
        1. 把调度区锁住，不再进行线程切换
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
#### 信号量
- 
    - 线程 通过 信号量的
### 信号量工作机制
- 
    - 线程之间同步的问题
    - thread之间通过 获取 释放 信号量
    - 达到 同步 或 互斥 的目的
- 每个信号量对象有
    1. 信号量值：
        - 对应信号量对象数量（资源数）
    1. 线程等待队列
    ```C
        struct rt_semaphore {
            struct rt_ipc_object_parent;
            rt_uint16_t value;
        }

        // 静态信号量
        struct rt_semaphore static_sem；
        // 动态信号量， 指向信号量的指针
        rt_sem_t dynamic_sem； 
    ```
### 信号量的操作
- 初始化和脱离
    ```C
    // 现在代码中定义出来 
    struct rt_semaphore static_sem;
    
    
    
    /*针对静态信号量*/
    rt_err_t rt_sem_init(rt_sem_t sem,   // 信号量的指针，将静态信号量的地址传入
                        const char*name, //名称
                        rt_uint32_t value, //初始值，几个改信号实例数目
                        rt_uint8_t flag)  // 信号量的标志 

    /*
    FLAG types
        RT_IPC_FLAG_FIFO //先进先出
        RT_IPC_FLAG_PRIO //按照优先级的方式排队等候
    flag 的作用
        当信号不好用的时候，等待进程的等待方式
    */
    re_err_t rt_sem_detach(rt_sem_t sem)
    //从管理器中移除
    ```
- 创建与删除
    ```C
    /*
    因为create函数需要申请信号量的内存空间，所以有成功和失败两种情况
    对于返回值，需要判断是否等于null！
    如果返回值为0，申请失败
    */
    rt_sem_t rt_sem_create(const char* name,
                            rt_uint32_t value,
                            rt_uint8_t flag)
    rt_sem_t rt_sem_delete(rt_sem_t sem)
    //自动释放信号实例内存资源
    ```

- 获取信号量
    ```C
    /*
        因为这个函数会使得的thread被挂起
        我们只在thread中调用
        不能在中断 ISR 中调用
    */

    rt_err_t rt_sem_take(rt_sem_t sem, rt_int32_t time) {};
    /*
    return value > 0  means signal is available
    return value == 0, means cannnot use signal, 申请信号量的thread申请失败
    如果不可用，该thread会根据tick参数进行等待（如果为0，立刻返回）
    tick是根据嘀嗒时钟进行等待的
    如果time<0，RT_WAITING_FOREVER,永远等待
    */
    rt_err_t rt_sem_trytake(rt_sem_t sem)
    /*
        如果没等到，返回一个rt_etimeout
    */
    ```
- 释放
    ```C
    /*
    可以在thread 和 ips 中调用，因为不会导致thread挂起
    */
    rt_err_t rt_sem_release(rt_sem_t sem)
    //释放一个

    ```
#### RT_EOK
    #define 	RT_EOK   0 //There is no error
    The error code is defined to identify which kind of error occurs. When some bad things happen, the current thread's errno will be set. see _rt_errno
    
    #define RT_ERROR   1 // A generic error happens

# 生产者消费者问题
- 
    -
    大小为n的缓冲区  
    生产者 写 缓冲间  
    消费者 取 缓冲区  

    核心：
        1. 生产者在 缓冲区 满 的时候 不写
        1. 消费者 在 缓冲区 空 的时候 不读
- 互斥关系
    - 缓冲区， 一时刻 一个thread
    - 0/1 信号量
- 同步关系
    - 生产后才能消费， 消费后才能生产
    - 缓冲区满信号量（init 0） 
    - <img src="https://raw.githubusercontent.com/YupengHan/RTOS-Learning/main/imgs/2.PNG" width = "480" height = "360" alt="图片名称" align=center />


<!-- ![avatar](https://raw.githubusercontent.com/YupengHan/RTOS-Learning/main/imgs/2.PNG = 100x100
) -->

- Yupeng Notes
    - 这个rt_semaphore 在 rt_sem_init 的时候没有自动的分配内存
    - rt_uint32_t value 这个值仅代表初始值的大小（意味着 后面的值可以比这个大，没有所谓最大限制的概念）

    ```C++
    rt_err_t rt_sem_init(rt_sem_t sem,   // 信号量的指针，将静态信号量的地址传入
                        const char*name, //名称
                        rt_uint32_t value, //初始值，几个改信号实例数目
                        rt_uint8_t flag)  // 信号量的标志
    
    
    struct rt_semaphore
    {
        struct rt_ipc_object parent;                        /**< inherit from ipc_object */

        rt_uint16_t          value;                         /**< value of semaphore. */
    };
    typedef struct rt_semaphore *rt_sem_t;

    struct rt_ipc_object
    {
        struct rt_object parent;                            /**< inherit from rt_object */

        rt_list_t        suspend_thread;                    /**< threads pended on this resource 是个双向链表*/
    };

    struct rt_list_node
    {
        struct rt_list_node *next;                          /**< point to next node. */
        struct rt_list_node *prev;                          /**< point to prev node. */
    };
    typedef struct rt_list_node rt_list_t;   

    ```

    ```C++
    // about the rt_sem_init
    rt_err_t rt_sem_init(rt_sem_t    sem,
                     const char *name,
                     rt_uint32_t value,
                     rt_uint8_t  flag)
    {
        RT_ASSERT(sem != RT_NULL);

        /* init object */
        rt_object_init(&(sem->parent.parent), RT_Object_Class_Semaphore, name);

        /* init ipc object */
        rt_ipc_object_init(&(sem->parent)); //初始化了一个double link list（用来存储对象）

        /* set init value */
        sem->value = value;

        /* set parent */
        sem->parent.parent.flag = flag;

        return RT_EOK;
    }
    ```

# 互斥量
#### semlock 0-1 量?
- 互斥锁 特殊的二值信号量
- 两种状态 locked unlocked
```C++
    struct rt_mutex
    {
        struct rt_ipc_object parent;                        /**< inherit from ipc_object */

        rt_uint16_t          value;                   //只有两种状态      /**< value of mutex */

        rt_uint8_t           original_priority;      //上次拥有这个互斥量的priority       /**< priority of last thread hold the mutex */
        rt_uint8_t           hold;                  //？？？线程持有互斥量的次数？         /**< numbers of thread hold the mutex */

        struct rt_thread    *owner;                     //当前持有这个mutex的线程控制块    /**< current owner of mutex */
    };

    struct_rt_mutex static_mutex;
    rt_mutex_t dynamic_mutex;
```

- <img src="https://raw.githubusercontent.com/YupengHan/RTOS-Learning/main/imgs/3.png" width = "800" height = "444" alt="图片名称" align=center />
```C++
    rt_err_t rt_mutex_detach //从内核管理中剔除
    rt_err_t rt_mutex_take //加锁，如果被使用了，目前线程就suspend
    /*
    和普通的信号量不同
    互斥量mutex 支持重复的take，
    Eg thread 1 using mutex
        thread2 take (suspend, hold)
        thread3 take (suspend, 加入hold的队列)
    */
    rt_err_t rt_mutex_release //加锁，只有这个thread take 了 当前mutex，才能使用release
    // take release 都只能在thread中操作，不可以在中断之中操作！
    // 信号量的release可以在中断中使用， mutex不行
```

# 线程的优先级翻转[Not Started]
# 事件集
-
    - 特定时间发生唤醒thread
    - 任意单个事件唤醒thread
    - 多个时间同时发生才能唤醒thread
- 
    - 信号量：1对1的线程同步
    - 事件集合：1对多 多对1 多对多 的同步
    - rt-thread事件集合 32bit uint，每个bit代表一个事件，线程通过 and or 与一个或多个时间链接
        - or 线程与任何一个事件发生同步 独立型同步
        - and 线程与若干事件都发生同步，需要若干事件全部发生 关联型同步
    

- 
    -
    ```C
    // 现在代码中定义出来 
    struct  rt_event {
        struct rt_ipc_object_parent;
        rt_uint32_t set;
    };
    typedef struct rt_event *rt_event_t;
    
    ```


# RT-thread 内核学习
### 内核基础 - Intro
- <img src="https://raw.githubusercontent.com/YupengHan/RTOS-Learning/main/imgs/4.png" width = "600" height = "300" alt="图片名称" align=center />
- 线程调度
    - 线程调度算法是基于优先级的全抢占式多线程调度算法，即在系统中除了中断处理函数、调度器上锁部分的代码和禁止中断的代码是不可抢占的之外，系统的其他部分都是可以抢占的
- 



# Next time start with BILIBILI videos





    

