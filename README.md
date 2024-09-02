# libevent源码分析

未完成

## **Reactor模式**

Reactor模式是一种事件驱动机制. 应用程序不是主动调用某个API完成处理, 而是逆置了事件处理流程, 应用程序需要提供相应的接口并注册到Reactor上, 如果相应事件发生, Reactor将主动调用应用程序注册的接口(回调函数), 使用libevent也是向libevent框架注册相应的事件和回调函数, libevent会调用这些回调函数处理相应的事件(IO读写, 定时和信号)

使用Reactor模式必备的几个组件: 事件源, Reactor框架, 多路复用机制和事件处理程序

![image-20240830150301128](D:\ueSim\cjw\md_pics\Reactor结构.png)

- 事件源: fd, 程序在指定的句柄上注册关心的事件, 比如IO事件

- event demultiplexer: 事件多路分发机制, 由操作系统提供的IO多路复用机制, 比如select和epoll. 程序首先将其关心的句柄(事件源)及其事件注册到event demultiplexer上, 当有事件到达时, event demultiplexer会发出通知"在已注册的句柄集中, 一个或多个句柄的世家已经就绪"; 程序收到通知后, 就可以在非阻塞的情况下对事件进行处理了. 在libevent中, 依然是select, poll, epoll等, 但是libevent用结构体`eventop`进行了封装, 以统一的接口来支持这些IO多路复用机制, 达到了对外隐藏底层系统机制的目的.

- Reactor: 是事件管理的接口, 内部使用event demultiplexer注册, 注销事件, 并运行事件循环, 当有事件进入"就绪状态"时, 调用注册事件的回调函数处理事件. 在libevent中, 对应`event_base`结构体

  ```cpp
  // 一种典型的Reactor声明方式
  class Reactor{
      public:
          int register_handler(Event_Handler *pHandler, int event);
          int remove_handler(Event_Handler *pHandler, int event);
          void handle_events(timeval *ptv);
          // ...
  };
  ```

- Event handler: 事件处理程序, 其提供了一组接口, 每个接口对应了一种类型的事件, 供Reactor在相应的事件发生时调用, 执行相应的事件处理. 通常其会绑定一个有效的句柄. 对应到libevent中, 就是`event`结构体. 

  ```cpp
  // 两种典型的event_handler声明方式
  class Event_Handler{
      public:
          virtual void handle_read() = 0;
          virtual void handle_write() = 0;
          virtual void handle_timeout() = 0;
          virtual void handle_close() = 0;
          virtual HANDLE get_handle() = 0;
          // ...
  };
  class Event_Handler{
      public:
          // events maybe read/write/timeout/close .etc
          virtual void handle_events(int events) = 0;
          virtual HANDLE get_handle() = 0;
          // ...
  };
  ```

Reactor事件处理流程:

![image-20240830152357967](D:\ueSim\cjw\Reactor事件处理流程)

## libevent基础流程

先看一段代码

```c
struct event ev;
struct timeval tv;
void time_cb(int fd, short event, void *argc){
    printf("timer wakeup/n");
    event_add(&ev, &tv); // reschedule timer
}

int main(){
    // 1.初始化libevent, 保存返回的指针.
    // 这一步相当于初始化一个reactor实例, 在初始化以后就可以注册事件了
    struct event_base *base = event_init();
    tv.tv_sec = 10; // 10s period
    tv.tv_usec = 0;
    // 2.初始化事件event, 设置回调函数(timer_cb)和关注的事件
    evtimer_set(&ev, time_cb, NULL);
    // 3.设置event从属的event_base,指明event要注册到哪个event_base实例上
    event_base_set(base, &ev);
    // 4.添加事件. 这里是定时事件
    event_add(&ev, &tv);
    // 5.程序进入无限循环, 等待就绪事件并执行事件处理
    event_base_dispatch(base);
}
```

当应用程序向libevent注册一个事件后，libevent内部是怎么样进行处理的呢？

1）首先应用程序准备并初始化event，设置好事件类型和回调函数；这对应于前面第步骤2和3； 

2）向libevent添加该事件event。对于定时事件，libevent使用一个小根堆管理，key为超时时间；对于Signal和I/O事件，libevent将其放入到等待链表（wait list）中，这是一个双向链表结构； 

3）程序调用**event_base_dispatch()**系列函数进入无限循环，等待事件，以select()函数为例；每次循环前libevent会检查定时事件的最小超时时间tv，根据tv设置select()的最大等待时间，以便于后面及时处理超时事件； 当select()返回后，首先检查超时事件，然后检查I/O事件； Libevent将所有的就绪事件，放入到激活链表中； 然后对激活链表中的事件，调用事件的回调函数执行事件处理；

## libevent基础数据结构

```c
// 这个结构体将不同平台上的事件循环的实现方法统一表示为这个结构体的五个方法
struct eventop {
	/** The name of this backend. */
	const char *name;
	/** Function to set up an event_base to use this backend.  It should
	 * create a new structure holding whatever information is needed to
	 * run the backend, and return it.  The returned pointer will get
	 * stored by event_init into the event_base.evbase field.  On failure,
	 * this function should return NULL. 
	 * 
	 * 初始化event_base
	 */
	void *(*init)(struct event_base *);
	/** Enable reading/writing on a given fd or signal.  
	 * 
	 * @ 'events' will be the events that we're trying to enable: one or more of EV_READ,
	 * EV_WRITE, EV_SIGNAL, and EV_ET.  事件类型
	 * @ 'old' will be those events that were enabled on this fd previously. 先前的事件类型
	 * @ 'fdinfo' will be a structure associated with the fd by the evmap; 
	 * its size is defined by the
	 * fdinfo field below.  It will be set to 0 the first time the fd is
	 * added.  The function should return 0 on success and -1 on error.
	 * 添加/删除事件
	 */
	int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
	/** As "add", except 'events' contains the events we mean to disable. */
	int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
	
	/** Function to implement the core of an event loop.  It must see which
	    added events are ready, and cause event_active to be called for each
	    active event (usually via event_io_active or such).  It should
	    return 0 on success and -1 on error.
		实现事件循环的核心, 检查哪些事件已经准备好, 并调用相应的事件处理函数
	 */
	int (*dispatch)(struct event_base *, struct timeval *);

	/** Function to clean up and free our data from the event_base. 清理内存*/
	void (*dealloc)(struct event_base *);
	/** Flag: set if we need to reinitialize the event base after we fork.
	 */
	int need_reinit;
	/** Bit-array of supported event_method_features that this backend can
	 * provide. */
	enum event_method_feature features;
	/** Length of the extra information we should record for each fd that
	    has one or more active events.  This information is recorded
	    as part of the evmap entry for each fd, and passed as an argument
	    to the add and del functions above.
	 */
	size_t fdinfo_len;
};
```

```c
// 使用libevent之前先调用event_init()初始化这个结构体
struct event_base {
	
	/** Function pointers and other data to describe this event_base's
	 * backend. */
	// 
	const struct eventop *evsel;

	/** Pointer to backend-specific data. */
	// 具体的平台相关的事件总线，例如当backend是epoll的时候，evbase就是epollop
	void *evbase;

	/** List of changes to tell backend about at next dispatch.  Only used
	 * by the O(1) backends. */
	struct event_changelist changelist;

	/** Function pointers used to describe the backend that this event_base
	 * uses for signals */
	const struct eventop *evsigsel;

	/** Data to implement the common signal handler code. */
	struct evsig_info sig;

	/** Number of virtual events */
	int virtual_event_count;
	/** Maximum number of virtual events active */
	int virtual_event_count_max;
	/** Number of total events added to this event_base */
	int event_count;
	/** Maximum number of total events added to this event_base */
	int event_count_max;
	
	/** Number of total events active in this event_base */
	int event_count_active;

	/** Maximum number of total events active in this event_base */
	int event_count_active_max;

	/** Set if we should terminate the loop once we're done processing
	 * events. */
	int event_gotterm;

	/** Set if we should terminate the loop immediately */
	int event_break;

	/** Set if we should start a new instance of the loop immediately. */
	int event_continue;

	/** The currently running priority of events */
	int event_running_priority;

	/** Set if we're running the event_base_loop function, to prevent
	 * reentrant invocation. */
	int running_loop;

	/** Set to the number of deferred_cbs we've made 'active' in the
	 * loop.  This is a hack to prevent starvation; it would be smarter
	 * to just use event_config_set_max_dispatch_interval's max_callbacks
	 * feature */
	int n_deferreds_queued;

	/* Active event management. */
	/** An array of nactivequeues queues for active event_callbacks (ones
	 * that have triggered, and whose callbacks need to be called).  Low
	 * priority numbers are more important, and stall higher ones.
	 */
	struct evcallback_list *activequeues;
	/** The length of the activequeues array */
	int nactivequeues;
	/** A list of event_callbacks that should become active the next time
	 * we process events, but not this time. */
	struct evcallback_list active_later_queue;

	/* common timeout logic */

	/** An array of common_timeout_list* for all of the common timeout
	 * values we know. */
	struct common_timeout_list **common_timeout_queues;
	/** The number of entries used in common_timeout_queues */
	int n_common_timeouts;
	/** The total size of common_timeout_queues. */
	int n_common_timeouts_allocated;

	/** Mapping from file descriptors to enabled (added) events */
	struct event_io_map io;

	/** Mapping from signal numbers to enabled (added) events. */
	struct event_signal_map sigmap;

	/** Priority queue of events with timeouts. */
	// 使用 最小堆 管理超时时间的队列
	struct min_heap timeheap;

	/** Stored timeval: used to avoid calling gettimeofday/clock_gettime
	 * too often. */
	struct timeval tv_cache;

	struct evutil_monotonic_timer monotonic_timer;

	/** Difference between internal time (maybe from clock_gettime) and
	 * gettimeofday. */
	struct timeval tv_clock_diff;
	/** Second in which we last updated tv_clock_diff, in monotonic time. */
	time_t last_updated_clock_diff;

#ifndef EVENT__DISABLE_THREAD_SUPPORT
	/* threading support */
	/** The thread currently running the event_loop for this base */
	unsigned long th_owner_id;
	/** A lock to prevent conflicting accesses to this event_base */
	void *th_base_lock;
	/** A condition that gets signalled when we're done processing an
	 * event with waiters on it. */
	void *current_event_cond;
	/** Number of threads blocking on current_event_cond. */
	int current_event_waiters;
#endif
	/** The event whose callback is executing right now */
	struct event_callback *current_event;

#ifdef _WIN32
	/** IOCP support structure, if IOCP is enabled. */
	struct event_iocp_port *iocp;
#endif

	/** Flags that this base was configured with */
	enum event_base_config_flag flags;

	struct timeval max_dispatch_time;
	int max_dispatch_callbacks;
	int limit_callbacks_after_prio;

	/* Notify main thread to wake up break, etc. */
	/** True if the base already has a pending notify, and we don't need
	 * to add any more. */
	int is_notify_pending;
	
	/** A socketpair used by some th_notify functions to wake up the main
	 * thread. */
	// 用于线程通知的一对管道的fd
	evutil_socket_t th_notify_fd[2];

	/** An event used by some th_notify functions to wake up the main
	 * thread. */
	struct event th_notify;
	/** A function used to wake up the main thread from another thread. */
	int (*th_notify_fn)(struct event_base *base);

	/** Saved seed for weak random number generator. Some backends use
	 * this to produce fairness among sockets. Protected by th_base_lock. */
	struct evutil_weakrand_state weakrand_seed;

	/** List of event_onces that have not yet fired. */
	LIST_HEAD(once_event_list, event_once) once_events;
};
```

```c
// libevent的核心
struct event {
	struct event_callback ev_evcallback;

	/* for managing timeouts */
	union {
		TAILQ_ENTRY(event) ev_next_with_common_timeout;
		int min_heap_idx; // 在最小堆中的索引值
	} ev_timeout_pos;
	evutil_socket_t ev_fd;

    // event关注的事件类型
    // 1.IO事件, EV_WRITE和EV_READ定时事件
    /* #define EV_TIMEOUT 0x01

       #define EV_READ  0x02

       #define EV_WRITE 0x04

       #define EV_SIGNAL 0x08

       #define EV_PERSIST 0x10 /* Persistant event */
    */
	short ev_events;
	short ev_res;		/* result passed to event callback */

	struct event_base *ev_base;

	union {
		/* used for io events */
		struct {
			LIST_ENTRY (event) ev_io_next;
			struct timeval ev_timeout;
		} ev_io;

		/* used by signal events */
		struct {
			LIST_ENTRY (event) ev_signal_next;
			short ev_ncalls;                 
			/* Allows deletes in callback */
			short *ev_pncalls;
		} ev_signal;
	} ev_;

	struct timeval ev_timeout;
};
```

```c
struct event_callback {
	TAILQ_ENTRY(event_callback) evcb_active_next;
	short evcb_flags;       // 表示event当前的状态
	ev_uint8_t evcb_pri;	/* 事件的权重 smaller numbers are higher priority */ 
	ev_uint8_t evcb_closure; // 终止的方式
	/* allows us to adopt for different types of events */
        union {
		void (*evcb_callback)(evutil_socket_t, short, void *);
		void (*evcb_selfcb)(struct event_callback *, void *);
		void (*evcb_evfinalize)(struct event *, void *);
		void (*evcb_cbfinalize)(struct event_callback *, void *);
	} evcb_cb_union;
	void *evcb_arg;
};
```

## 事件处理框架

### 创建和初始化event_base

创建一个event_base对象也既是创建了一个新的libevent实例，程序需要通过调用event_init()（内部调用event_base_new函数执行具体操作）函数来创建，该函数同时还对新生成的libevent实例进行了初始化。

- 该函数首先为event_base实例申请空间，
- 然后初始化timer mini-heap，选择并初始化合适的系统I/O 的demultiplexer机制，初始化各事件链表；

函数还检测了系统的时间设置，为后面的时间管理打下基础。

### 接口函数

Reactor框架的作用就是提供事件的注册、注销接口；根据系统提供的事件多路分发机制执行事件循环，当有事件进入“就绪”状态时，调用注册事件的回调函数来处理事件。 Libevent中对应的接口函数主要就是：

```c
int  event_add(struct event *ev, const struct timeval *timeout);
int  event_del(struct event *ev);
int  event_base_loop(struct event_base *base, int loops);
void event_active(struct event *event, int res, short events);
void event_process_active(struct event_base *base);
```

- 对于**定时事件**，这些函数将调用timer heap管理接口执行插入和删除操作；
- 对于**I/O和Signal事件**将调用eventopadd和delete接口函数执行插入和删除操作（eventop会对Signal事件调用Signal处理接口执行操作）；

#### 注册事件函数原型: 

```c
int event_add(struct event *ev, const struct timeval *tv)
```

参数：ev：指向要注册的事件； tv：超时时间；

e函数将ev注册到ev->ev_base上，事件类型由ev->ev_events指明，

- 如果注册成功，v将被插入到已注册链表中；
- 如果tv不是NULL，则会同时注册定时事件，将ev添加到timer堆上；

如果其中有一步操作失败，那么函数保证没有事件会被注册，可以讲这相当于一个原子操作

```c
int event_add(struct event *ev, const struct timeval *tv) {
    struct event_base *base = ev->ev_base;
    // 要注册到的event_base
    const struct eventop *evsel = base->evsel;
    void *evbase = base->evbase;
    // base使用的系统I/O策略
    // 新的timer事件，调用timer heap接口在堆上预留一个位置
    // 注：这样能保证该操作的原子性：
    // 向系统I/O机制注册可能会失败，而当在堆上预留成功后，
    // 定时事件的添加将肯定不会失败；
    // 而预留位置的可能结果是堆扩充，但是内部元素并不会改变
    if (tv != NULL && !(ev->ev_flags & EVLIST_TIMEOUT)) {
        if (min_heap_reserve(&base->timeheap, 1 + min_heap_size(&base->timeheap)) == -1)
                    return (-1);
        /* ENOMEM == errno */
    }
    // 如果事件ev不在已注册或者激活链表中，则调用evbase注册事件
    if ((ev->ev_events & (EV_READ|EV_WRITE|EV_SIGNAL)) && !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE))) {
        res = evsel->add(evbase, ev);
        if (res != -1) // 注册成功，插入event到已注册链表中
        event_queue_insert(base, ev, EVLIST_INSERTED);
    }
    // 准备添加定时事件
    if (res != -1 && tv != NULL) {
        struct timeval now;
        // EVLIST_TIMEOUT表明event已经在定时器堆中了，删除旧的
        if (ev->ev_flags & EVLIST_TIMEOUT)
                    event_queue_remove(base, ev, EVLIST_TIMEOUT);
        // 如果事件已经是就绪状态则从激活链表中删除
        if ((ev->ev_flags & EVLIST_ACTIVE) &&
                (ev->ev_res & EV_TIMEOUT)) {
            // 将ev_callback调用次数设置为0
            if (ev->ev_ncalls && ev->ev_pncalls) {
                *ev->ev_pncalls = 0;
            }
            event_queue_remove(base, ev, EVLIST_ACTIVE);
        }
        // 计算时间，并插入到timer小根堆中
        gettime(base, &now);
        evutil_timeradd(&now, tv, &ev->ev_timeout);
        event_queue_insert(base, ev, EVLIST_TIMEOUT);
    }
    return (res);
}
```

- event_queue_insert()负责将事件插入到对应的链表中
- event_queue_remove()负责将事件从对应的链表中删除

#### 删除事件函数原型: 

```c
int event_del(struct event *ev);
```

该函数将删除事件ev

- 对于I/O事件，从I/O 的demultiplexer上将事件注销；
- 对于Signal事件，将从Signal事件链表中删除；
- 对于定时事件，将从堆上删除；

同样删除事件的操作则不一定是原子的，比如删除时间事件之后，有可能从系统I/O机制中注销会失败

```c
int event_del(struct event *ev) {
    struct event_base *base;
    const struct eventop *evsel;
    void *evbase;
    // ev_base为NULL，表明ev没有被注册
    if (ev->ev_base == NULL)
      return (-1);
    // 取得ev注册的event_base和eventop指针
    base = ev->ev_base;
    evsel = base->evsel;
    evbase = base->evbase;
    // 将ev_callback调用次数设置为
    if (ev->ev_ncalls && ev->ev_pncalls) {
        *ev->ev_pncalls = 0;
    }
    // 从对应的链表中删除
    if (ev->ev_flags & EVLIST_TIMEOUT)
      event_queue_remove(base, ev, EVLIST_TIMEOUT);
    if (ev->ev_flags & EVLIST_ACTIVE)
      event_queue_remove(base, ev, EVLIST_ACTIVE);
    if (ev->ev_flags & EVLIST_INSERTED) {
        event_queue_remove(base, ev, EVLIST_INSERTED);
        // EVLIST_INSERTED表明是I/O或者Signal事件，
        // 需要调用I/O demultiplexer注销事件
        return (evsel->del(evbase, ev));
    }
    return (0);
}
```

#### 事件主循环

libevent的事件主循环主要是通过**event_base_loop ()**函数完成的，其主要操作如下面的流程图所示，**event_base_loop**所作的就是持续执行下面的循环

![image-20240902113449070](./libevent源码分析/image-20240902113449070.png)

```c
int event_base_loop(struct event_base *base, int flags){
    const struct eventop *evsel = base->evsel;
    void *evbase = base->evbase;
    struct timeval tv;
    struct timeval *tv_p;
    int res, done;
    // 清空时间缓存
    base->tv_cache.tv_sec = 0;
    // evsignal_base是全局变量，在处理signal时，用于指名signal所属的event_base实例
    if (base->sig.ev_signal_added)
        evsignal_base = base;
    done = 0;
    while (!done) { // 事件主循环
        // 查看是否需要跳出循环，程序可以调用event_loopexit_cb()设置event_gotterm标记
        // 调用event_base_loopbreak()设置event_break标记
        if (base->event_gotterm) {
            base->event_gotterm = 0;
            break;
        }
        if (base->event_break) {
            base->event_break = 0;
            break;
        }
        // 校正系统时间，如果系统使用的是非MONOTONIC时间，用户可能会向后调整了系统时间
        // 在timeout_correct函数里，比较last wait time和当前时间，如果当前时间< last wait time
        // 表明时间有问题，这是需要更新timer_heap中所有定时事件的超时时间。
        timeout_correct(base, &tv);
        // 根据timer heap中事件的最小超时时间，计算系统I/O demultiplexer的最大等待时间
        tv_p = &tv;
        if (!base->event_count_active && !(flags & EVLOOP_NONBLOCK)) {
            timeout_next(base, &tv_p);
        } else {
            // 依然有未处理的就绪时间，就让I/O demultiplexer立即返回，不必等待
            // 下面会提到，在libevent中，低优先级的就绪事件可能不能立即被处理
            evutil_timerclear(&tv);
        }
        // 如果当前没有注册事件，就退出
        if (!event_haveevents(base)) {
            event_debug(("%s: no events registered.", __func__));
            return (1);
        }
        // 更新last wait time，并清空time cache
        gettime(base, &base->event_tv);
        base->tv_cache.tv_sec = 0;
        // 调用系统I/O demultiplexer等待就绪I/O events，可能是epoll_wait，或者select等；
        // 在evsel->dispatch()中，会把就绪signal event、I/O event插入到激活链表中
        res = evsel->dispatch(base, evbase, tv_p);
        if (res == -1)
            return (-1);
        // 将time cache赋值为当前系统时间
        gettime(base, &base->tv_cache);
        // 检查heap中的timer events，将就绪的timer event从heap上删除，并插入到激活链表中
        timeout_process(base);
        // 调用event_process_active()处理激活链表中的就绪event，调用其回调函数执行事件处理
        // 该函数会寻找最高优先级（priority值越小优先级越高）的激活事件链表，
        // 然后处理链表中的所有就绪事件；
        // 因此低优先级的就绪事件可能得不到及时处理；
        if (base->event_count_active) {
            event_process_active(base);
            if (!base->event_count_active && (flags & EVLOOP_ONCE))
                done = 1;
        } else if (flags & EVLOOP_NONBLOCK)
            done = 1;
    }
    // 循环结束，清空时间缓存
    base->tv_cache.tv_sec = 0;
    event_debug(("%s: asked to terminate loop.", __func__));
    return (0);
}
```

#### epoll为例分析IO多路复用

实现在源文件**epoll.c**中，eventops对象epollops定义如下：

```c
const struct eventop epollops = {
    "epoll",
    epoll_init,
    epoll_add,
    epoll_del,
    epoll_dispatch,
    epoll_dealloc,
    1 /* need reinit */
};
struct epollop {
	struct epoll_event *events;
	int nevents;
	int epfd;
#ifdef USING_TIMERFD
	int timerfd;
#endif
};
```

```c
static void *epoll_init    (struct event_base *);
static int epoll_add    (void *, struct event *);
static int epoll_del    (void *, struct event *);
static int epoll_dispatch(struct event_base *, void *, struct timeval *);
static void epoll_dealloc    (struct event_base *, void *);
```

