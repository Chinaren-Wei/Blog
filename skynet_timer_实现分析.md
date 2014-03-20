# Skynet Timer 实现分析

#### timer 模块入口基本介绍
skynet 启动时，启动单独一个线程处理 timer 事件，每 2.5 毫秒执行一次函数 skynet_updatetime 更新系统时间并处理 timeout  超时事件。  

	create_thread(&pid[1], _timer, m);

	static void *
	_timer(void *p) {
	    struct monitor * m = p;
	    for (;;) {
	        skynet_updatetime();
	        CHECK_ABORT
	        wakeup(m,m->count-1);
	        usleep(2500);
	    }
		...
	}

先看看 skynet_updatetime 函数：  

	void
	skynet_updatetime(void) {
	    uint32_t ct = _gettime();
	    if (ct != TI->current) {
	        int diff = ct>=TI->current?ct-TI->current:(0xffffff+1)*100-TI->current+ct;
	        TI->current = ct;
    	    int i;
    	    for (i=0;i<diff;i++) {
    	        timer_execute(TI);
    	    }
    	}
	} 
_gettime() 调用系统时钟获取最新的系统时间，先算出距离上一次更新时间的 diff，然后通过 for 循环执行 diff 次 timer 事件处理函数 timer_execute。

那个 if 判断和 diff 的计算看起来很诡异，但其实原理很简单。_gettime() 虽然通过系统调用获取时间，但该函数根据当前的时间计算最终返回的却是一个 uint32_t 的整数，这个整数被限定于始终小于 (0xffffff+1)*100，如果超过，则从 0 重新开始，所以才有那个比较诡异的 diff 计算。

_gettime() 函数如下：

	static uint32_t
	_gettime(void) {
		uint32_t t;
		...
	    struct timeval tv;
	    gettimeofday(&tv, NULL);
	    t = (uint32_t)(tv.tv_sec & 0xffffff) * 100;
	    t += tv.tv_usec / 10000;
	    ...
	    return t;
	}

返回的结果，单位：10 毫秒

那个 for 循环含义明了了，每 10 毫秒执行一次事件处理函数 timer_execute。

获取系统时间的目的就在于控制这个 timer_execute 函数的执行频率，后续 timer 实现与时间就没什么关系，是另外一套机制。

##### timer 数据结构分析
	#define TIME_NEAR_SHIFT 8
	#define TIME_NEAR (1 << TIME_NEAR_SHIFT)
	#define TIME_LEVEL_SHIFT 6
	#define TIME_LEVEL (1 << TIME_LEVEL_SHIFT)
	#define TIME_NEAR_MASK (TIME_NEAR-1)
	#define TIME_LEVEL_MASK (TIME_LEVEL-1)

	struct timer_node {
	    struct timer_node *next;
	    int expire;
	};

	struct link_list {
	    struct timer_node head;
	    struct timer_node *tail;
	};

	struct timer {
    	struct link_list near[TIME_NEAR];
    	struct link_list t[4][TIME_LEVEL-1];
    	int lock;
    	int time;
    	uint32_t current;
    	uint32_t starttime;
	};

	static struct timer * TI = NULL;


near 成员是一个 size 为 16 的数组，或者叫 hash 表。用途是将在最近 16(TIME_NEAR) * 10 (毫秒) 内即将发生的事件 hash 到这个数组中，每 10 毫秒(也就是刚才的 for 循环) 执行其中一组。


成员 t 是一个二维数组，用二维数组的目的是为了根据事件的 timeout 给时间由近到远分级存储在这 4 个 hash 表内。目前写死了 4 级事件。超时时间最长的都在 t[4] 这个 hash 表中；给 timeout 分级的目的在于提高事件处理的效率。

time 初始值是 0，每执行函数 timer_execute 时递增一次，根据设计，应该就是每 10 毫秒递增一次。time 的值，就是自 timer 启动以来所经历的时间( time * 10 毫秒)

lock 锁变量

current 用以 cache _gettime 执行的结果.

starttime 字段记录 timer 的启动时间。


near 和 t 的元素都是一个链表，被 hash 到同一个位置的元素用链表串联起来。


##### 如何添加一个事件
添加事件的入口是这个 timer_add 函数:

1. 先 malloc 一个 timer_node，相关的 data 数据紧随其后存储。 
2. 给 timer_node 设置好超时时间： node->expire= time + T->time;
3. 执行实际的添加操作 add_node(T,node);  

		static void
		timer_add(struct timer *T,void *arg,size_t sz,int time)
		{
    		struct timer_node *node = (struct timer_node *)malloc(sizeof(*node)+sz);
    		memcpy(node+1,arg,sz);
	
    		while (__sync_lock_test_and_set(&T->lock,1)) {};
		
    		    node->expire=time+T->time;
    		    add_node(T,node);
	
    		__sync_lock_release(&T->lock);
		}

add_node 函数，比较多位操作，不熟悉的情况，头很晕。  

TIME_NEAR_MASK 的二进制值是 0000 1111  

	static void
	add_node(struct timer *T,struct timer_node *node)
	{
    	int time=node->expire;
    	int current_time=T->time;

		// TIME_NEAR_MASK 的二进制值是 0000 1111 
		// 把最近要发生的事件(timeout 事件)放进 T->near 的 hash 表里面
		// 换一种写法就好理解了，如：if ((time - current_time) <= TIME_NEAR_MASK)
    	if ((time|TIME_NEAR_MASK)==(current_time|TIME_NEAR_MASK)) {
    	    link(&T->near[time&TIME_NEAR_MASK],node);
    	}
    	
    	// 如果不是最近事件，则开始计算该事件的层数，每次移 TIME_LEVEL_SHIFT 进行判断
    	else {
    	    int i;
    	    int mask=TIME_NEAR << TIME_LEVEL_SHIFT;
    	    for (i=0;i<3;i++) {
        	    if ((time|(mask-1))==(current_time|(mask-1))) {
        	        break;
        	    }
        	    mask <<= TIME_LEVEL_SHIFT;
        	}
        	link(&T->t[i][((time>>(TIME_NEAR_SHIFT + i*TIME_LEVEL_SHIFT)) & TIME_LEVEL_MASK)-1],node);
    	}
	}


##### timer 事件逻辑
核心函数：timer_execute

该函数分 2 部分，第一部分，执行一次 near 事件；第二部分，重新计算所有事件所在的层级，如，把新处于 near 的事件转移到 near hash 表中。

先看第一部分，处理 near 事件，或者叫 timeout 事件：

	static void
	timer_execute(struct timer *T)
	{
	    while (__sync_lock_test_and_set(&T->lock,1)) {};
		
		// 计算本次要处理的 near hash 表的位置，这个位置根据 time 来 hash，随着 time 的递增，素有的事件都可以被处理到
		 	    
	    int idx=T->time & TIME_NEAR_MASK;
		...
    	while (T->near[idx].head.next) {
    	    current=link_clear(&T->near[idx]);
			// 处理事件列表
    	    do {
        	    struct timer_event * event = (struct timer_event *)(current+1);
        	    struct skynet_message message;
            	message.source = 0;
            	skynet_context_push(event->handle, &message);
				...
            	struct timer_node * temp = current;
            	current=current->next;
            	free(temp);
        	} while (current);
    	}
    	
		// 自定义时钟 + 1
    	++T->time;

    	mask = TIME_NEAR;
    	time = T->time >> TIME_NEAR_SHIFT;
    	i=0;
		
		// 每 TIME_NEAR 次循环，执行一次重建事件层级的逻辑
		// 每 TIME_LEVEL 次，执行一次下一级事件重建逻辑，不太好描述，还是看代码吧！
    	while ((T->time & (mask-1))==0) {
    	    idx=time & TIME_LEVEL_MASK;
    	    if (idx!=0) {
        	    --idx;
            	current=link_clear(&T->t[i][idx]);
            	while (current) {
            	    struct timer_node *temp=current->next;
            	    add_node(T,current);
            	    current=temp;
            	}
            	break;
        	}
        	mask <<= TIME_LEVEL_SHIFT;
        	time >>= TIME_LEVEL_SHIFT;
        	++i;
    	}
    	...
 


 