---
layout:     post   				    # 使用的布局（不需要改）
title:      从锁和RTE ring与KFIFO说起
subtitle:   不能不学习啊 #副标题
date:       2021-01-03 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# 从锁和RTE ring与KFIFO说起

## 前言
为什么写这篇博客？本质上是昨天大师和我讨论RTE ring的无锁操作，外加我在看KFIFO和《C++并发编程》，这个整理完成，我就可以转而去看《计算机体系结构xxxxx》的内容了。毕竟不能说东西整完了无所得。

## 对关键数据/流程的保护
代码进入临界区，对关键数据做竞争修改，是导致恶性条件竞争的原因。三种解决方法：
+ 第一种很常见，就是锁，锁关键数据，保证不会同时多个进行操作。锁完了刷新所有CPU的缓存。
+ 还有一种解决方法就是所谓的无锁编程，本质上就是追求修改的一系列操作不可细分的，有点原子操作那味儿了。
+ 另一种就是将一系列的操作交给一个统一的操作者进行。由操作者判断能不能执行成功。有点类似操作数据库的味道了

下面分开来谈，先说锁，再说无锁，最后事务操作不说了，

### 锁的保护
无论是C/C++操作对于关键数据的操作都是需要加锁的，因此必须小心某个函数在未获得锁的情况下，返回关键数据的引用或者指针。对于C++而言常见的关键数据共享还有个问题，函数调用传入了关键参数，但此时不在锁的保护范围。但是并不是说对关键数据加了个锁就会避免所有的竞争条件--这里面藏着一个所谓“条件竞争”的问题，举个简单的例子，删除链表一个元素的时候需要同时获得三个指针，被删除指针+被删除之前指针+被删除之后指针，即使获取了其中单一一个指针的锁，仍然不能避免竞争的产生。这实际上就是常说的锁粒度问题，锁粒度太细就保护不力，锁粒度太粗性能有一塌糊涂。因此有人采用了多个互斥量竞争的情况，然后就产生了死锁。

常见的死锁的条件每个人都清楚，互相等待，互相需求，都不会释放。似乎只要避免这三者问题就不会死锁。实际并非如此，前面条件成立有个前提：互相等待的锁是不同的锁！举个简单的例子，有一个函数swap(a, b)，每次都先获取a的锁，再获取b的锁，然后交换a，b。看上去似乎没什么问题？现在一个线程执行swap(obj_1, obj_2)，另一个线程执行swap(obj_2, obj_1)。照样GG，而且这东西还挺无解的。

C++ 对于这东西的解决方案是一次锁两个对象，保证不会有第二个线程还能锁成功。至于C，我第一反应是对函数做个保护，然后转念反应过来锁应该保护数据，而不是保护函数。因此只需要第二次获得的时候如果失败就放弃。

所以最终，如何避免死锁的结论是：
+ 要么就每次线程就只有一个锁，别同时上来申请二三四个锁。猜猜这里我想到了什么？mm_struct的读写信号量+页表的自旋锁。
+ 锁的生命周期务必要短，别拿着锁的时候还来干一大堆脏活累活，这是一个避免出现问题的很有效的方法。
+ 正确的申请锁的顺序，每个都一样，如果swap函数每次都是先申请小地址的锁，然后申请大地址的锁，那也没啥问题。这个申请锁顺序就需要参数顺序无关了
+ 正确的使用分层锁，

OK，那么如何设计一个有锁的并发数据结构呢？

实际上就是，
+ 锁尽量少，锁尽量短。
+ 对C++而言，不能再构造函数或者析构函数之后访问，对C而言就是不能访问已经释放的内存。这个让我想起来前几天别人问我的一个free地址出错，coredump失败的问题。因为那个函数是我写的，每个地方如果malloc内存失败，就会把指针改成0x0。最后定位到一个socket获取地址那里，应该是获取失败最后导致失败。
+ 避免函数里面出现锁的争用，类比rw_sem的实现，读可以多多，写尽量少。




## 无锁或者说原子操作 

C++ 提供了std::atomic来进行无锁操作或者说原子操作，atomic除了进行原子修改操作之外，还要求明确的赋值必须完成之后才能发生，也就是说同时提供了内存屏障的操作。因此原子操作数常常作为flag进行操作。这里面伴随着两个概念先行(happens-before)和同发(synchronizes-with)。

在《内存模型和缓存一致性》里，写原子性，指写完了其他核立刻可见，和这里的同步发生是一个理论，指的是线程A的原子写操作从执行的结果来看在线程B的原子读之前(这里的原子读之前，可能A的原子写操作已经被其他的线程所read_and_set了)，这里需要注意的点是同发，同步发生是个跨线程的概念。

而先行，也就是“先行发生”是同一个线程里的事情，在源代码里A发生在B之前。操作同步发生的话，通常就没有先行关系了。这里需要引入一个线程之间的“先行发生”。可以理解为，线程里面的顺序在线程之间传递

这里所谓的同发就是个蛋疼的概念，实际上就是一个先对原子变量写，另一个对原子变量读，虽然叫做同步发生关系，但是本质上是一先一后。还有一个先行关系，包括同一个线程的先行，线程间的先行。这里，C++整进来一个非常复杂的东西memory_order，我个人觉得就是闲的蛋疼。这里实际上就是原子操作的依赖性。这块说实话我觉得《C++并发实战》翻译的很糟糕，建议要么直接看英文，建议看[C++内存模型](https://paul.pub/cpp-memory-model/) 这篇文章。

这时候就得说说内存模型的问题了，总共有六种内存模型：顺序一致性(sequentially consistent)，获取-释放序(memory_order_consume, memory_order_acquire, memory_order_release和memory_order_acq_rel)和自由序(memory_order_relaxed)。详细的内容看[cpp reference](https://zh.cppreference.com/w/cpp/atomic/memory_order)

那么C++中的内存序和我们平时看到的内存序有什么关系呢？C++的内存序提供了一种高于指令的抽象，但是这种抽象在我看来也比较失败。顺序序，我个人觉得更简单对于SEQ_CST的看法，不如理解为两个原子变量之间的操作是稳定的，不会发生重排序的。自由序，或者直接说弱内存序和《内存模型和缓存一致性入门》里不同的地方是，《XXX入门》里面，结果是固定的，即两个不同地址的（不同）对象A和B，看到的值变化的先后顺序是可能变化的，即可以是先A后B，或者先B后A，但是总能是一个固定的结果。而C++提供的弱内存序则不保证看到，每个线程看到的都是固定的结果，可能线程1看到A后B，线程2看到先B后A。这个东西实际上我觉得可以将之理解为不同核共享CACHE导致的，比方说共享L2级CACHE的写缓冲区导致两个核先看到A最后涉及到的获取-释放序，如果是分布在不同线程里的变量：A在线程1，B在线程2，那么结果观察到的和自由序是没有区别的。如果是在同一个线程里执行的操作，那么就不同了。

这里最好按照书里面提供的例子，用公式的观点来进行观察，这样子能够简化思考。

为了方便理解，在观察结果的线程里，可以认为atomic类型的变量观察的顺序是不会释放的。



顺序一致性值得是两个原子操作先A后B，在其他线程看到的也是先A后B，这实际上就是当前线程跟新了其他线程的缓存。
非顺序一致性内存-自由序，看到这里大部分人都会虎躯一震，线程1原子操作是先A后B，线程2里看到的B已经做了，但A未必做。这tm也叫原子操作？按照《C++并发实战》解释来说是：唯一的要求是在访问同一线程中的单个原子变量不能重排序，当给定线程看到原子变量的值时，随后线程的读操作就不会去检索较早的那个值。实际上就是线程1没跟新线程2的缓存，后面跟了一个给数字的例子解释自由序，可以给你旧的缓存值，因此很可能看到和赋值顺序不一致的原子值。只保证线程里面看到新东西之不会在看到旧东西，不能保证线程外面看到新东西，另一个东西看到的也是新的。
非顺序一致性内存-获取/释放序，操作没有顺序，但是有同步。原子加载就是获取，原子存储就是释放，获取释放必然是成对的，也就是说，获取时发现已经释放了。那么释放之前的代码在获取时已经完成了。实际上，锁住锁就是获取，解开锁就是释放，这就保证了锁必然是线性执行的。

OK，那么如何设计一个无锁的并发数据结构呢？

## RTE_ring的无锁实现

首先我们要清楚的明白，多消费，多生产的竞争来自两个方面。一方面是生产和生产的竞争，另一方面是生产和消费的竞争。RTE_ring怎么解决生产和生产的竞争？

简单总结，RTE_ring的无所实现是先占一段距离，令head和tail来开距离。操作完成了，再把head和tail改成一个值。当tail和head一起的时候说明要么现在没人enqueue，要么已经enqueue完成了。

仔细想想实际上这里面依靠的就是一个“圈地的原子性”。而对tail的操作则是不需要原子性的，所以按照C11的写法来说。我们只需要保证read是atomic，顺序一致即可。

mp_enqueue的代码本质还是先占坑，后拉屎的行为。函数`__rte_ring_move_prod_head`中，每次生产者都尝试原子获取并移动`r->prod.head`，因为每个线程都从`r->prod.head`填写数据，这保证了每次都只有一个线程先把坑占好。接着`__rte_ring_enqueue_elems`把内容扔进去。

最后调用`update_tail`函数，实际上update_tail是加了一个对前面操作的写屏障，然后尝试等待生产队列位到位置。

```
static __rte_always_inline unsigned int
__rte_ring_do_enqueue_elem(struct rte_ring *r, const void *obj_table,
		unsigned int esize, unsigned int n,
		enum rte_ring_queue_behavior behavior, unsigned int is_sp,
		unsigned int *free_space)
{
	uint32_t prod_head, prod_next;
	uint32_t free_entries;

	n = __rte_ring_move_prod_head(r, is_sp, n, behavior,
			&prod_head, &prod_next, &free_entries);
	if (n == 0)
		goto end;

	__rte_ring_enqueue_elems(r, prod_head, obj_table, esize, n);

	update_tail(&r->prod, prod_head, prod_next, is_sp, 1);
end:
	if (free_space != NULL)
		*free_space = free_entries - n;
	return n;
}


static __rte_always_inline unsigned int
__rte_ring_move_prod_head(struct rte_ring *r, unsigned int is_sp,
		unsigned int n, enum rte_ring_queue_behavior behavior,
		uint32_t *old_head, uint32_t *new_head,
		uint32_t *free_entries)
{
	const uint32_t capacity = r->capacity;
	unsigned int max = n;
	int success;

	do {
		/* Reset n to the initial burst count */
		n = max;

		*old_head = r->prod.head;

		/* add rmb barrier to avoid load/load reorder in weak
		 * memory model. It is noop on x86
		 */
		rte_smp_rmb();

		/*
		 *  The subtraction is done between two unsigned 32bits value
		 * (the result is always modulo 32 bits even if we have
		 * *old_head > cons_tail). So 'free_entries' is always between 0
		 * and capacity (which is < size).
		 */
		*free_entries = (capacity + r->cons.tail - *old_head);

		/* check that we have enough room in ring */
		if (unlikely(n > *free_entries))
			n = (behavior == RTE_RING_QUEUE_FIXED) ?
					0 : *free_entries;

		if (n == 0)
			return 0;

		*new_head = *old_head + n;
		if (is_sp)
			r->prod.head = *new_head, success = 1;
		else
			success = rte_atomic32_cmpset(&r->prod.head,
					*old_head, *new_head);
	} while (unlikely(success == 0));
	return n;
}

static __rte_always_inline void
update_tail(struct rte_ring_headtail *ht, uint32_t old_val, uint32_t new_val,
		uint32_t single, uint32_t enqueue)
{
	if (enqueue)
		rte_smp_wmb();
	else
		rte_smp_rmb();
	/*
	 * If there are other enqueues/dequeues in progress that preceded us,
	 * we need to wait for them to complete
	 */
	if (!single)
		while (unlikely(ht->tail != old_val))
			rte_pause();

	ht->tail = new_val;
}
```



## BOOST::SPSC_QUEUE的实现

```cpp

#ifdef BOOST_NO_CXX11_VARIADIC_TEMPLATES
template <typename T, typename A0, typename A1>
#else
template <typename T, typename ...Options>
#endif
struct make_ringbuffer
{
#ifdef BOOST_NO_CXX11_VARIADIC_TEMPLATES
    typedef typename ringbuffer_signature::bind<A0, A1>::type bound_args;
#else
    typedef typename ringbuffer_signature::bind<Options...>::type bound_args;
#endif
  
...
    /* if_c是一个偏特化模板，如果runtime_sized为false，就使用第二个类型作为type。否则选择第一个类型作为type*/
    /* 具体的意思是buffer的大小到底是运行时确定还是编译时确定，我们关注编译时确定的ringbuffer */
    typedef typename mpl::if_c<runtime_sized,
                               runtime_sized_ringbuffer<T, allocator>,
                               compile_time_sized_ringbuffer<T, capacity>
                              >::type ringbuffer_type;
};



template <typename T, std::size_t MaxSize>
class compile_time_sized_ringbuffer:
    public ringbuffer_base<T>
{
    typedef std::size_t size_type;
    static const std::size_t max_size = MaxSize + 1;

    typedef typename boost::aligned_storage<max_size * sizeof(T),
                                            boost::alignment_of<T>::value
                                           >::type storage_type;

    storage_type storage_;
...
  

/* 而aligned_storage到最后*/

struct aligned_storage_imp
{
    union data_t
    {
        char buf[size_];

        typename ::boost::type_with_alignment<alignment_>::type align_;
    } data_;
    void* address() const { return const_cast<aligned_storage_imp*>(this); }
};
template <std::size_t size>
struct aligned_storage_imp<size, std::size_t(-1)>
{
   union data_t
   {
      char buf[size];
      ::boost::detail::max_align align_;
   } data_;
   void* address() const { return const_cast<aligned_storage_imp*>(this); }
};

template< std::size_t alignment_ >
struct aligned_storage_imp<0u,alignment_>
{
    /* intentionally empty */
    void* address() const { return 0; }
};

}} // namespace detail::aligned_storage

template <
      std::size_t size_
    , std::size_t alignment_ = std::size_t(-1)
>
class aligned_storage : 
#ifndef BOOST_BORLANDC
   private 
#else
   public
#endif
   ::boost::detail::aligned_storage::aligned_storage_imp<size_, alignment_> 
{
 
public: // constants

    typedef ::boost::detail::aligned_storage::aligned_storage_imp<size_, alignment_> type;

    BOOST_STATIC_CONSTANT(
          std::size_t
        , size = size_
        );
    BOOST_STATIC_CONSTANT(
          std::size_t
        , alignment = (
              alignment_ == std::size_t(-1)
            ? ::boost::detail::aligned_storage::alignment_of_max_align
            : alignment_
            )
        );

private: // noncopyable

    aligned_storage(const aligned_storage&);
    aligned_storage& operator=(const aligned_storage&);

public: // structors

    aligned_storage()
    {
    }

    ~aligned_storage()
    {
    }

public: // accessors

    void* address()
    {
        return static_cast<type*>(this)->address();
    }

```





## kfifo的无锁实现

可以看到本质上就是一个先enqueue再挪动指针，保证数据先进去，再稳定挪动的操作。这个过程使用了内存屏障等多种操作。在ARM ustack atcp_rq就会出现问题。

```
	unsigned int kfifo_in(struct kfifo *fifo, const void *from,
				unsigned int len)
{
	len = min(kfifo_avail(fifo), len);

	__kfifo_in_data(fifo, from, len, 0);
	__kfifo_add_in(fifo, len);
	return len;
}

static inline void __kfifo_in_data(struct kfifo *fifo,
		const void *from, unsigned int len, unsigned int off)
{
	unsigned int l;

	/*
	 * Ensure that we sample the fifo->out index -before- we
	 * start putting bytes into the kfifo.
	 */

	smp_mb();

	off = __kfifo_off(fifo, fifo->in + off);

	/* first put the data starting from fifo->in to buffer end */
	l = min(len, fifo->size - off);
	memcpy(fifo->buffer + off, from, l);

	/* then put the rest (if any) at the beginning of the buffer */
	memcpy(fifo->buffer, from + l, len - l);
}

static inline void __kfifo_add_in(struct kfifo *fifo,
				unsigned int off)
{
	smp_wmb();
	fifo->in += off;
}
```



## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
