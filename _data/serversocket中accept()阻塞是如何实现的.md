``` java
public class Server {

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(9090);
        serverSocket.accept();
    }
}
```

accept()执行后，会阻塞等待连接。底层是怎么实现阻塞的?

accept()在DualStackPlainSocketImpl.java(windows)中调用的是native方法，accept0()。

> linux为PlainSocketImpl.java

![image-20240510170528097](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/image-20240510170528097.png)

这里调用了accept函数。显然它定义在头文件中。

![img](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/1575054-20200611200737522-533661763.png)](https://img2020.

在系统中找到了头文件winsock2.h。

![img](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/1575054-20200611200746437-1932606643.png)

函数声明在这里。但是找不到对应的实现。想到linux中应该有类似开源的源码。

![img](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/1575054-20200611200752953-684949313.png)

在一个博客上找到了linux中TCP accept的实现，这里我没那么严谨，取自己找到源码来读（暂时能力和精力有限哈）。https://blog.csdn.net/mrpre/article/details/82655834
其中重要的是这段代码。我以为底层是通过for(;;)实现的阻塞，但是对于每个进程for(;;)只会执行一次，那为什么这么写，主要是历史遗留原因，博主也有说明。那么阻塞是怎么实现的呢？最核心是schedule_timeout函数。于是又找到了源码。

![img](https://raw.githubusercontent.com/candyboyou/imgs/master/imgs/1575054-20200611200800221-1850689149.png)

> 注意上图中首先将进程加入到等待队列里面，并设置为可中断。schedule_timeout是理想的延迟方法。会让需要延迟的任务睡眠指定的时间。最核心的是schedule函数。它的前后是设置定时器和删除定时器。定时器到指定时间会唤醒进程，重新加入就绪队列。

看来是schedule实现了阻塞。找来了它的源码。

```java
signed long __sched schedule_timeout(signed long timeout)
{
    struct timer_list timer;
    unsigned long expire;

    switch (timeout)
    {
    case MAX_SCHEDULE_TIMEOUT: //睡眠时间无限大，则不需要设置定时器。
        /*
         * These two special cases are useful to be comfortable
         * in the caller. Nothing more. We could take
         * MAX_SCHEDULE_TIMEOUT from one of the negative value
         * but I' d like to return a valid offset (>=0) to allow
         * the caller to do everything it want with the retval.
         */
        schedule();
        goto out;
    default:
        /*
         * Another bit of PARANOID. Note that the retval will be
         * 0 since no piece of kernel is supposed to do a check
         * for a negative retval of schedule_timeout() (since it
         * should never happens anyway). You just have the printk()
         * that will tell you if something is gone wrong and where.
         */
        if (timeout < 0) {
            printk(KERN_ERR "schedule_timeout: wrong timeout "
                "value %lx\n", timeout);
            dump_stack();
            current->state = TASK_RUNNING;
            goto out;
        }
    }

    expire = timeout + jiffies;

    setup_timer_on_stack(&timer, process_timeout, (unsigned long)current);
    __mod_timer(&timer, expire, false, TIMER_NOT_PINNED);
    schedule();
    del_singleshot_timer_sync(&timer);

    /* Remove the timer from the object tracker */
    destroy_timer_on_stack(&timer);

    timeout = expire - jiffies;

 out:
    return timeout < 0 ? 0 : timeout;
}
```

内核版本2.6.39。schedule主要实现了进程调度。即让出当前进程的CPU，切换上下文。既然切换到其它进程执行了，而当前进程又进入了等待队列，不再会被调度。直到时间结束，或者中断来。这时候会从schedule返回。删除定时器。根据返回的timeout值判断是否大于0判断是睡眠到时间被唤醒，还是因为有连接了提前唤醒。



```java
jav/*
 * schedule() is the main scheduler function.
 */
asmlinkage void __sched schedule(void)
{
 struct task_struct *prev, *next;
 unsigned long *switch_count;
 struct rq *rq;
 int cpu;
 
need_resched:
 preempt_disable(); //禁止内核抢占
 cpu = smp_processor_id(); //获取当前CPU
 rq = cpu_rq(cpu); //获取该CPU维护的运行队列（run queue)
 rcu_note_context_switch(cpu); //更新全局状态，标识当前CPU发生上下文的切换。
 prev = rq->curr; //运行队列中的curr指针赋予prev。
 
 schedule_debug(prev); 
 
 if (sched_feat(HRTICK))
  hrtick_clear(rq);
 
 raw_spin_lock_irq(&rq->lock); //锁住该队列
 
 switch_count = &prev->nivcsw; //记录当前进程的切换次数
 if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) { //是否同时满足以下条件：1该进程处于停止状态，2该进程没有在内核态被抢占。
  if (unlikely(signal_pending_state(prev->state, prev))) { //若不是非挂起信号，则将该进程状态设置成TASK_RUNNING
   prev->state = TASK_RUNNING;
  } else { //若为非挂起信号则将其从队列中移出
   /*
    * If a worker is going to sleep, notify and
    * ask workqueue whether it wants to wake up a
    * task to maintain concurrency. If so, wake
    * up the task.
    */
   if (prev->flags & PF_WQ_WORKER) {     
    struct task_struct *to_wakeup;
 
    to_wakeup = wq_worker_sleeping(prev, cpu);
    if (to_wakeup)
     try_to_wake_up_local(to_wakeup);
   }
   deactivate_task(rq, prev, DEQUEUE_SLEEP); //从运行队列中移出
 
   /*
    * If we are going to sleep and we have plugged IO queued, make
    * sure to submit it to avoid deadlocks.
    */
   if (blk_needs_flush_plug(prev)) {
    raw_spin_unlock(&rq->lock);
    blk_schedule_flush_plug(prev);
    raw_spin_lock(&rq->lock);
   }
  }
  switch_count = &prev->nvcsw; //切换次数记录
 }
 
 pre_schedule(rq, prev); 
 
 if (unlikely(!rq->nr_running))
  idle_balance(cpu, rq);
 
 put_prev_task(rq, prev);  
 next = pick_next_task(rq); //挑选一个优先级最高的任务将其排进队列。
 clear_tsk_need_resched(prev); //清除pre的TIF_NEED_RESCHED标志。
 rq->skip_clock_update = 0;
 
 if (likely(prev != next)) { //如果prev和next非同一个进程
  rq->nr_switches++; //队列切换次数更新
  rq->curr = next;
  ++*switch_count; //进程切换次数更新
 
  context_switch(rq, prev, next); /* unlocks the rq */ //进程之间上下文切换
  /*
   * The context switch have flipped the stack from under us
   * and restored the local variables which were saved when
   * this task called schedule() in the past. prev == current
   * is still correct, but it can be moved to another cpu/rq.
   */
  cpu = smp_processor_id();
  rq = cpu_rq(cpu);
 } else //如果prev和next为同一进程，则不进行进程切换。
  raw_spin_unlock_irq(&rq->lock);  
 
 post_schedule(rq);
 
 preempt_enable_no_resched();
 if (need_resched()) //如果该进程被其他进程设置了TIF_NEED_RESCHED标志，则函数重新执行进行调度
  goto need_resched;
}
```