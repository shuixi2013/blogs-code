title: (转) unlocked_ioctl 和堵塞（waitqueue）读写函数的实现
date: 2015-01-19 10:31:16
updated: 2015-01-19 10:31:16
categories: [Basics Knowledge]
tags: [basics, linux]
---

学习了驱动程序的设计，感觉在学习驱动的同时学习linux内核，也是很不错的过程哦，做了几个实验，该做一些总结，只有不停的作总结才能印象深刻。

我的平台是虚拟机，fedora14，内核版本为2.6.38.1.其中较之前的版本存在较大的差别，具体的实现已经在上一次总结中给出了。今天主要总结的是ioctl和堵塞读写函数的实现。

## 一、ioctl函数的实现

首先说明在2.6.36以后ioctl函数已经不再存在了，而是用`unlocked_ioctl`和`compat_ioctl`两个函数实现以前版本的ioctl函数。同时在参数方面也发生了一定程度的改变，去除了原来ioctl中的struct inode参数，同时改变了返回值。

但是驱动设计过程中存在的问题变化并不是很大，同样在应用程序设计中我们还是采用ioctl实现访问，而并不是`unlocked_ioctl`函数，因此我们还可以称之为ioctl函数的实现。ioctl函数的实现主要是用来实现具体的硬件控制，采用相应的命令控制硬件的具体操作，这样就能使得硬件的操作不再是单调的读写操作。使得硬件的使用更加的方便。ioctl函数实现主要包括两个部分，首先是命令的定义，然后才是ioctl函数的实现，命令的定义是采用一定的规则。

ioctl的命令主要用于应用程序通过该命令操作具体的硬件设备，实现具体的操作，在驱动中主要是对命令进行解析，通过switch-case语句实现不同命令的控制，进而实现不同的硬件操作。

ioctl函数的命令定义方法：

<pre>
int (*unlocked_ioctl)(struct file*filp,unsigned int cmd,unsigned long arg)
</pre>

虽然其中没有指针的参数，但是通常采用arg传递指针参数。cmd是一个命令。**每一个命令由一个整形数据构成（32bits），将一个命令分成四部分，每一部分实现具体的配置，设备类型（幻数）8bits，方向2bits，序号8bits，数据大小13/14bits**。命令的实现实质上就是通过简单的移位操作，将各个部分组合起来而已。

一个命令的分布的大概情况如下：
<pre config="brush:bash;toolbar:false;">
|---方向位(31-30)|----数据长度(29-16)----------------|---------设备类型（15-8）------|----------序号（7-0）----------|
|----------------------------------------------------------------------------------------------------------------------------------------|
</pre>

其中方向位主要是表示对设备的操作，比如读设备，写设备等操作以及读写设备等都具有一定的方向，2个bits只有4种方向。数据长度表示每一次操作（读、写）数据的大小，一般而已每一个命令对应的数据大小都是一个固定的值，不会经常改变，14bits说明可以选择的数据长度最大为16k。

设备类型类似于主设备号（由于8bits，刚好组成一个字节，因此经常采用字符作为幻数，表示某一类设备的命令），用来区别不同的命令类型，也就是特定的设备类型对应特定的设备。序号主要是这一类命令中的具体某一个，类似于次设备号（256个命令），也就是一个设备支持的命令多达256个。

同时在内核中也存在具体的宏用来定义命令以及解析命令。但是大部分的宏都只是定义具体的方向，其他的都需要设计者定义。

主要的宏如下：

<pre config="brush:bash;toolbar:false;">

#include<linux/ioctl.h>

_IO(type,nr)                表示定义一个没有方向的命令，
_IOR(type,nr,size)          表示定义一个类型为type，序号为nr，数据大小为size的读命令
_IOW(type,nr,size)          表示定义一个类型为type，序号为nr，数据大小为size的写命令
_IOWR(type,nr,size)         表示定义一个类型为type，序号为nr，数据大小为size的写读命令

</pre>

通常的type可采用某一个字母或者数字作为设备命令类型。是实际运用中通常采用如下的方法定义一个具体的命令:

```cpp

//头文件
#include<linux/ioctl.h>

/*定义一系列的命令*/
/*幻数，主要用于表示类型*/
#define MAGIC_NUM 'k'
/*打印命令*/
#define MEMDEV_PRINTF _IO(MAGIC_NUM,1)
/*从设备读一个int数据*/
#define MEMDEV_READ _IOR(MAGIC_NUM,2,int)
/*往设备写一个int数据*/
#define MEMDEV_WRITE _IOW(MAGIC_NUM,3,int)

/*最大的序列号*/
#define MEM_MAX_CMD 3

```

还有对命令进行解析的宏，用来确定具体命令的四个部分（方向，大小，类型，序号）具体如下所示：

```cpp

/*确定命令的方向*/
_IOC_DIR(nr)                    
/*确定命令的类型*/
_IOC_TYPE(nr)                     
/*确定命令的序号*/
_IOC_NR(nr)                           
/*确定命令的大小*/
_IOC_SIZE(nr)    

```

上面的几个宏可以用来命令，实现命令正确性的检查。

ioctl的实现过程主要包括如下的过程：

1. 命令的检测
2. 指针参数的检测
3. 命令的控制switch-case语句

1、命令的检测主要包括类型的检查，数据大小，序号的检测，通过结合上面的命令解析宏可以快速的确定。

```cpp

        /*检查类型，幻数是否正确*/
        if(_IOC_TYPE(cmd)!=MAGIC_NUM)
                return -EINVAL;
        /*检测命令序号是否大于允许的最大序号*/
        if(_IOC_NR(cmd)> MEM_MAX_CMD)
                return -EINVAL;

```

2、主要是指针参数的检测。指针参数主要是因为内核空间和用户空间的差异性导致的，因此需要来自用户空间指针的有效性。使用`copy_from_user`,`copy_to_user`,`get_user`,`put_user`之类的函数时，由于函数会实现指针参量的检测，因此可以省略，但是采用`__get_user()`,`__put_user()`之类的函数时一定要进行检测。具体的检测方法如下所示：

```cpp

if(_IOC_DIR(cmd) & _IOC_READ)
        err = !access_ok(VERIFY_WRITE,(void *)args,_IOC_SIZE(cmd));
else if(_IOC_DIR(cmd) & _IOC_WRITE)
        err = !access_ok(VERIFY_READ,(void *)args,_IOC_SIZE(cmd));
if(err)/*返回错误*/
        return -EFAULT;

```

**当方向是读时，说明是从设备读数据到用户空间，因此要检测用户空间的指针是否可写，采用`VERIFY_WRITE`，而当方向是写时，说明是往设备中写数据，因此需要检测用户空间中的指针的可读性`VERIFY_READ`。检查通常采用`access_ok()`实现检测，第一个参数为读写，第二个为检测的指针，第三个为数据的大小**。

3、命名的控制：命令的控制主要是采用switch和case相结合实现的，这于window编程中的检测各种消息的实现方式是相同的。

```cpp

/*根据命令执行相应的操作*/
        switch(cmd)
        {
                case MEMDEV_PRINTF:
                        printk("<--------CMD MEMDEV_PRINTF Done------------>\n\n");
                        ...
                        break;
                case MEMDEV_READ:
                        ioarg = &mem_devp->data;
                        ...
                        ret = __put_user(ioarg,(int *)args);
                        ioarg = 0;
                        ...
                        break;
                case MEMDEV_WRITE:
                        ...
                        ret = __get_user(ioarg,(int *)args);
                        printk("<--------CMD MEMDEV_WRITE Done ioarg = %d--------->\n\n",ioarg); 
                        ioarg = 0;
                        ...
                        break;
                default:
                        ret = -EINVAL;
                        printk("<-------INVAL CMD--------->\n\n");
                        break;
        }

```

这只是基本的框架结构，实际中根据具体的情况进行修改。这样就实现了基本的命令控制。
文件操作支持的集合如下：

```cpp

/*添加该模块的基本文件操作支持*/
static const struct file_operations mem_fops =
{
        /*结尾不是分号，注意其中的差别*/
        .owner = THIS_MODULE,
        .llseek = mem_llseek,
        .read = mem_read,
        .write = mem_write,
        .open = mem_open,
        .release = mem_release,
        /*添加新的操作支持*/
        .unlocked_ioctl = mem_ioctl,
};

```

需要注意不是ioctl,而是unlocked_ioctl。

## 二、设备的堵塞读写方式实现，通常采用等待队列。

设备的堵塞读写方式，默认情况下的读写操作都是堵塞型的，具体的就是如果需要读数据，当设备中没有数据可读的时候应该等待设备中有设备再读，当往设备中写数据时，如果上一次的数据还没有被读完成，则不应该写入数据，就会导致进程的堵塞，等待数据可读写。但是在应用程序中也可以采用非堵塞型的方式进行读写。只要在打开文件的时候添加一个`O_NONBLOCK`,这样在不能读写的时候就会直接返回，而不会等待。

因此我们在实际设计驱动设备的同时需要考虑读写操作的堵塞方式。堵塞方式的设计主要是通过等待队列实现，通常是将等待队列（实质就是一个链表）的头作为设备数据结构的一部分。在设备初始化过程中初始化等待队列的头。最后在设备读写操作的实现添加相应的等待队列节点，并进行相应的控制。

等待队列的操作基本如下：

1、等待队列的头定义并初始化的过程如下：

方法一：
<pre>
struct wait_queue_head_t mywaitqueue;
init_waitqueue_head(&mywaitqueue);
</pre>
方法二：
<pre>
DECLARE_WAIT_QUEUE_HEAD(mywaitqueue);
</pre>
以上的两种都能实现定义和初始化等待队列头。

2、创建、移除一个等待队列的节点，并添加、移除相应的队列。
定义一个等待队列的节点:`DECLARE_WAITQUEUE(wait,tsk)`
其中tsk表示一个进程，可以采用current当前的进程。
添加到定义好的等待队列头中。
<pre>
add_wait_queue(wait_queue_head_t *q,wait_queue_t *wait);
即：add_wait_queue(&mywaitqueue,&wait);
</pre>

移除等待节点
<pre>
remove_wait_queue(wait_queue_head_t *q,wait_queue_t *wait);
即：remove_wait_queue(&mywaitqueue,&wait);
</pre>

3、等待事件
`wait_event(queue,condition)`;当condition为真时，等待队列头queue对应的队列被唤醒，否则继续堵塞。这种情况下不能被信号打断。
`wait_event_interruptible(queue,condition)`;当condition为真时，等待队列头queue对应的队列被唤醒，否则继续堵塞。这种情况下能被信号打断。

4、唤醒等待队列
`wait_up(wait_queue_head_t *q)`,唤醒该等待队列头对应的所有等待。
`wait_up_interruptible(wait_queue_head_t *q)`唤醒处于`TASK_INTERRUPTIBLE`的等待进程。

应该成对的使用。即`wait_event`于`wait_up`,而`wait_event_interruptible`与`wait_up_interruptible`。

```cpp

wait_event和wait_event_interruptible的实现都是采用宏的方式，都是一个重新调度的过程，如下所示：
#define wait_event_interruptible(wq, condition)                \
({                                    \
    int __ret = 0;                            \
    if (!(condition))                        \
        __wait_event_interruptible(wq, condition, __ret);    \
    __ret;                                \
})
#define __wait_event_interruptible(wq, condition, ret)            \
do {                                    \
     /*此处存在一个声明等待队列的语句，因此不需要再重新定义一个等待队列节点*/
    DEFINE_WAIT(__wait);                        \
                                    \
    for (;;) {                            \
        /*此处就相当于add_wait_queue()操作，具体参看代码如下所示*/
        prepare_to_wait(&wq, &__wait, TASK_INTERRUPTIBLE);    \
        if (condition)                        \
            break;                        \
        if (!signal_pending(current)) {                \
            /*此处是调度，丢失CPU，因此需要wake_up函数唤醒当前的进程
		根据定义可知，如果条件不满足，进程就失去CPU,能够跳出for循环的出口只有
                1、当条件满足时2、当signal_pending（current）=1时。
                1、就是满足条件，也就是说wake_up函数只是退出了schedule函数，
                而真正退出函数还需要满足条件	
                2、说明进程可以被信号唤醒。也就是信号可能导致没有满足条件时就唤醒当前的进程。 
               这也是后面的代码采用while判断的原因.防止被信号唤醒。   
	   */
            schedule();                    \
            continue;                    \
        }                            \
        ret = -ERESTARTSYS;                    \
        break;                            \
    }                                \
    finish_wait(&wq, &__wait);                    \
} while (0)

#define DEFINE_WAIT(name) DEFINE_WAIT_FUNC(name, autoremove_wake_function)
#define DEFINE_WAIT_FUNC(name, function)				\
	wait_queue_t name = {						\
		.private	= current,				\
		.func		= function,				\
		.task_list	= LIST_HEAD_INIT((name).task_list),	\
	}

void prepare_to_wait(wait_queue_head_t *q, wait_queue_t *wait, int state)
{
	unsigned long flags;

	wait->flags &= ~WQ_FLAG_EXCLUSIVE;
	spin_lock_irqsave(&q->lock, flags);
	if (list_empty(&wait->task_list))
               /*添加节点到等待队列*/
		__add_wait_queue(q, wait);
	set_current_state(state);
	spin_unlock_irqrestore(&q->lock, flags);
}
唤醒的操作也是类似的。
#define wake_up_interruptible(x)	__wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
 
  void __wake_up(wait_queue_head_t *q, unsigned int mode,
			int nr_exclusive, void *key)
{
	unsigned long flags;

	spin_lock_irqsave(&q->lock, flags);
	__wake_up_common(q, mode, nr_exclusive, 0, key);
	spin_unlock_irqrestore(&q->lock, flags);
}

static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
			int nr_exclusive, int wake_flags, void *key)
{
	wait_queue_t *curr, *next;

	list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
		unsigned flags = curr->flags;

		if (curr->func(curr, mode, wake_flags, key) &&
				(flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
			break;
	}
}

```

等待队列通常用在驱动程序设计中的堵塞读写操作，并不需要手动的添加节点到队列中，直接调用即可实现，具体的实现方法如下：

1、在设备结构体中添加等待队列头，由于读写都需要堵塞，所以添加两个队列头，分别用来堵塞写操作，写操作。

```cpp

#include<linux/wait.h>

struct mem_dev
{
        char *data;
        unsigned long size;
        /*添加一个并行机制*/
        spinlock_t lock;

        /*添加一个等待队列t头*/
        wait_queue_head_t rdqueue;
        wait_queue_head_t wrqueue;
};

```

2、然后在模块初始化中初始化队列头:

```cpp

/*初始化函数*/
static int memdev_init(void)
{
       ....
        for(i = 0; i < MEMDEV_NR_DEVS; i)
        {
                mem_devp[i].size = MEMDEV_SIZE;
                /*对设备的数据空间分配空间*/
                mem_devp[i].data = kmalloc(MEMDEV_SIZE,GFP_KERNEL);
                /*问题，没有进行错误的控制*/
                memset(mem_devp[i].data,0,MEMDEV_SIZE);

                /*初始化定义的互信息量*/
                //初始化定义的自旋锁ua
                spin_lock_init(&(mem_devp[i].lock));
                /*初始化两个等待队列头,需要注意必须用括号包含起来，使得优先级正确*/
                init_waitqueue_head(&(mem_devp[i].rdqueue));
                init_waitqueue_head(&(mem_devp[i].wrqueue));
        }
      ...
}

```

**3、确定一个具体的条件，比如数据有无，具体的条件根据实际的情况设计。**
<pre>
/*等待条件*/
static bool havedata = false;
</pre>

4、在需要堵塞的读函数，写函数中分别实现堵塞，首先定义等待队列的节点，并添加到队列中去，然后等待事件的唤醒进程。但是由于读写操作的两个等待队列都是基于条件havedata的，所以在读完成以后需要唤醒写，写完成以后需要唤醒读操作，同时更新条件havedata，最后还要移除添加的等待队列节点。

```cpp

/*read函数的实现*/
static ssize_t mem_read(struct file *filp,char __user *buf, size_t size,loff_t *ppos)
{
        unsigned long p = *ppos;
        unsigned int count = size;
        int ret = 0;

        struct mem_dev *dev = filp->private_data;

        /*参数的检查，首先判断文件位置*/
        if(p >= MEMDEV_SIZE)
                return 0;
        /*改正文件大小*/
        if(count > MEMDEV_SIZE - p)
                count = MEMDEV_SIZE - p;
         #if 0
        /*添加一个等待队列节点到当前进程中*/
        DECLARE_WAITQUEUE(wait_r,current);

        /*将节点添加到等待队列中*/
        add_wait_queue(&dev->rdqueue,&wait_r);

        /*添加等待队列，本来采用if即可，但是由于信号等可能导致等待队列的唤醒，因此采用循环，确保不会出现误判*/
        #endif

        while(!havedata)
        {
                /*判断用户是否设置为非堵塞模式读,告诉用户再读*/
                if(filp->f_flags & O_NONBLOCK)
                        return -EAGAIN;

                /*依据条件havedata判断队列的状态，防止进程被信号唤醒*/
                wait_event_interruptible(dev->rdqueue,havedata);
        }

        spin_lock(&dev->lock);
        /*从内核读数据到用户空间，实质就通过private_data访问设备*/
        if(copy_to_user(buf,(void *)(dev->data p),count))
        {
                /*出错误*/
                ret = -EFAULT;
        }
        else
        {
                /*移动当前文件光标的位置*/

                *ppos = count;
                ret = count;

                printk(KERN_INFO "read %d bytes(s) from %d\n",count,p);
        }
      
        spin_unlock(&dev->lock);
	 #if 0
        /*将等待队列节点从读等待队列中移除*/
        remove_wait_queue(&dev->rdqueue,&wait_r);
	#endif 

        /*更新条件havedate*/
        havedata = false;
        /*唤醒写等待队列*/
        wake_up_interruptible(&dev->wrqueue);

        return ret;
}

```

```cpp

/*write函数的实现*/
static ssize_t mem_write(struct file *filp,const char __user *buf,size_t size,loff_t *ppos)
{
        unsigned long p = *ppos;
        unsigned int count = size;
        int ret = 0;

        /*获得设备结构体的指针*/
        struct mem_dev *dev = filp->private_data;

        /*检查参数的长度*/
        if(p >= MEMDEV_SIZE)
                return 0;
        if(count > MEMDEV_SIZE - p)
                count = MEMDEV_SIZE - p;
	  #if 0
        /*定义并初始化一个等待队列节点，添加到当前进程中*/
        DECLARE_WAITQUEUE(wait_w,current);
        /*将等待队列节点添加到等待队列中*/
        add_wait_queue(&dev->wrqueue,&wait_w);
        #endif

        /*添加写堵塞判断*/
        /*为何采用循环是为了防止信号等其他原因导致唤醒*/
        while(havedata)
        {
                /*如果是以非堵塞方式*/
                if(filp->f_flags & O_NONBLOCK)
                        return -EAGAIN;
        /*分析源码发现，wait_event_interruptible 中存在DECLARE_WAITQUEUE和add_wait_queue的操作，因此不需要手动添加等待队列节点*/
                wait_event_interruptible(&dev->wrqueue,(!havedata));
        }

        spin_lock(&dev->lock);

        if(copy_from_user(dev->data p,buf,count))
                ret = -EFAULT;
        else
        {
                /*改变文件位置*/
                *ppos = count;
                ret = count;
                printk(KERN_INFO "writted %d bytes(s) from %d\n",count,p);
        }

        spin_unlock(&dev->lock);
	
	#if 0
        /*将该等待节点移除*/
        remove_wait_queue(&dev->wrqueue,&wait_w);
	#endif 

        /*更新条件*/
        havedata = true;
        /*唤醒读等待队列*/
        wake_up_interruptible(&dev->rdqueue);

        return ret;
}

```

5、应用程序采用两个不同的进程分别进行读、写，然后检测顺序是否可以调换，检查等待是否正常。

