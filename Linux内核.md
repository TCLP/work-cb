[技术简说的头条主页 - 今日头条(www.toutiao.com)](https://www.toutiao.com/c/user/token/MS4wLjABAAAAQea_WW8hzmBfzmL4PZ6Bl8UKsNgxb2G8TMUhg6fBLp8/)

# 1内存管理

## (1)内核空间与用户空间

32位系统内核空间和用户空间地址范围，0x00000000-0xc0000000(3G)用户空间，0xc0000000-0xffffffff(1G)内核空间，可配置；

为什么要划分内核空间和用户空间？

* 处理器模式不同，权限不同

  对于x86体系的cpu，用户空间的代码运行在ring3模式，内核空间的代码运行在ring0模式；

  对于arm体系的cpu，用户空间代码运行在usr模式，内核空间代码运行在svc模式；

* 安全考量

  系统中有各种资源，比如计算资源， 内存资源和外设资源，而linux是多用户，多进程系统， 所以，这些资源必须受限制；

* 设计上解耦

  内核代码侧重与系统管理，解除核心代码与业务逻辑代码的耦合，内核代码偏重于系统管理，用户空间代码偏重于业务逻辑实现；

驱动程序在内核通过copy_from_user和copy_to_user函数在内核空间与用户空间进行数据拷贝；

## (2)x86段页管理

地址转换

* 逻辑地址->线性地址->物理地址

  段页转换：逻辑地址->线性地址
  页表映射：线性地址->物理地址

  在linux中，一般设置段基址为0，所以一般逻辑地址就是线性地址；

页表映射(二级)

* linear address -> phy address(MMU)

  [dir(10bits) | table(10bits) | offset(12bits)]

  dir：page directory，页目录起始地址保存在cr3寄存器中，page directory中每个页目录保存1024个数据，每个数据4bytes，所以页目录一般占用4k大小物理内存空间，即一个物理块大小，```每个进程都有自己的cr3起始地址```；

  table：page table，根据linear address中dir的值找到了page table的起始地址，根据page table中的值就找到了物理页表的起始地址，每个项保存1024条数据，所以每个page table占用4k物理空间，即一个物理块大小；

  offset：通过page directory and page table找到了物理块的起始地址，通过起始地址加上offset就得到了真正的物理地址； 

* e.g

  liear address：0xC1234567

  高10位：0x304

  中10位：0x234

  低12位：0x567

  假设最终从page table中读取到的内容是0x123，说明物理页块的起始地址是0x123，0x123左移12bits + 0x567就是最终的物理地址；

多级页表这种方式进程访问内存时至少两次访存，效率不高，所以后面引入TLB，即页表缓冲(地址变换高速缓存)；

## (3)linux内存管理和分配机制

* 页框

  linux内核使用struct page结构体描述每个物理页，也叫页框；

* 伙伴算法

  内核在很多情况下，需要申请连续的页框，而且数量不定，所以，linux把所有的空闲页框分组在了11个链表中，每个链表上的页框块是固定的，在第i条链表中每个页框块都包含了2的i次方个连续页，所以每次分配的最大的连续物理内存大小就是4K * 2^10 = 4M，注意，系统中的每个页框的物理地址是该块大小的整数倍，例如，大小为16个页框的块，其起始地址是16*2^12的倍数；

* 页框操作

  * alloc_pages()

    static inline struct page *alloc_pages(gfp_t gfp_mask, unsigned int order)

    分配2^order个连续的物理页，并返回一个指针，指向第一个页的page结构体

  * page_address()

    void *page_address(const struct page *page)

    返回page页面所映射的虚拟地址

* slab

  slab实现了内存分配与管理，在驱动开发中，常常分配的内存是很小的，所以slab层把不同的对象划分为所谓的高速缓存(cache)组，其中每个高速缓存都存放不同类型的对象，每种对象类型对应一个高速缓存(cache)，例如一个高速缓存存放task_struct结构体，而另一个高速缓存存放struct inode结构体，`slab由一个或者多个物理上连续的页组成，每个高速缓存由多个slab组成`；

  ```   
  chche_chain--->kmem_cache(obj_size=32B)--->kmem_cache(obj_size=64B)
                 |
                 |
                \|/
  slabs_full   slabs_partial   slabs_free
       |
       |
      \|/
     双向链表(slab1(head)<--->slab2<--->slab3<--->slab4(tail))
                               |
                               |
                              \|/
                连续page(page1-page2-page3-page4)
                                |
                                |
                               \|/
                    双向链表(object1<--->object2<--->object3)
  ```

  slab机制要解决的问题：

  * 减少伙伴算法在分配小块连续内存时所产生的内部碎片；
  * 将频繁使用的对象缓存起来，减少分配，初始化和释放对象的时间开销；
  * 通过着色技术调整对象以更好的使用硬件高速缓存；

* slab扩展

  * slab
  * slob
  * slub

* slab-高速缓存的建立

  ````
  struct kmem_cache *kmem_cache_create(const char *name, size_t size, size_t align, unsigned long flags, void (*ctor)(void *));
  ````

  可以通过命令查看系统已经建立的slab高速缓存：

  `cat /proc/slabinfo`

* slab-高速缓存的使用

  ```
  void *kmem_cache_alloc(struct kmem_cache *cachep,gfp_t flags);
  void *kmem_cache_destory(struct kmem_cache *cachep);
  flag作用：涉及到内核里面不同场景内存分配情况，如在中断上下文里面是可以睡眠的，在进程上下文里面是不可以睡眠的，再比如指定从dma中申请内存，还是从普通内存区域申请内存；
  ```

* kmalloc/vmalloc函数

  ```
  返回一个内存块的指针，其大小至少为size，所分配的内存在物理上是连续的：
  static __always_inline void *kmalloc(size_t size, gfp_t flags);
  static __always_inline void *kzalloc(size_t size, gfp_t flags);//分配并将空间置为0
  返回一个内存块的指针，其大小至少为size，所分配的内存在物理上无需连续的：
  void *vmalloc(unsigned long size);
  void *vzalloc(unsigned long size);//分配并将空间置为0
  ```

* kmalloc源码

  ```
  ......
  ```

## (4)linux进程虚拟地址管理

进程的3G的虚拟地址空间只有被映射为物理地址空间，才能够被使用，进程如何管理和分配它的3G的虚拟地址空间了--->采用分治思想；

按照不同的访问属性和功能划分为不同的内存区域，称为虚拟内存区域(VMA)：

```
3G     --->    代码段
               数据段
               bbs段
               lib库的代码段
               lib库的数据段
               lib库的bbs段
               任何内存映射文件(有名mmap建立)
               任何共享内存段(匿名mmap建立)
               进程栈stack
               进程堆heap
```

内核使用mm_struct来描述一个进程的虚拟地址空间，使用vm_area_struct来描述一个虚拟内存区域(VMA)，进程的虚拟地址空间由多个VMA组成；

```
struct mm_struct {
	struct vm_area_struct *mmap;//指向虚拟内存区域的链表
	struct rb_root mm_rb;		//指向红黑树
	struct vm_area_struct *mmap_cache;//指向最近找到的虚拟内存区域
	pgd_t *pgd;					//指向进程的页目录表,可以实现共享,如同一进程的多个线程
	atomic_t mm_users;			//用户空间有多少用户
	atomic_t mm_count;			//对mm_struct有多少引用
	int map_count;				//虚拟内存的个数
	struct list_head mmlist;	//所有活动的mm的链表
	unsigned long start_code, end_code, start_data, end_data;
	unsigned long start_brk, brk, start_stack;
	unsigned long arg_start, arg_end;
	unsigned long env_start, env_end;
};
strcut vm_area_struct {
	struct mm_struct *vm_mm;	//虚拟内存区域所在的虚拟内存空间
	unsigned long vm_start;		//在虚拟内存空间的起始地址
	unsigned long vm_end;		//在虚拟内存空间的结束地址
	struct vm_area_struct *vm_next, *vm_prev;
	pgport_t vm_page_port;		//虚拟内存区域对应的存取权限
	unsigned long vm_flags;		//虚拟内存区域标志
	struct vm_operations_struct *vm_ops;//操作函数
	struct file *vm_file;		//file we map to(can be null)
	unsigned long vm_pgoff;		//文件中的偏移
};
```

线程之间共享内存地址的实现机制：

在linux中，如果clone()时设置CLONE_VM标记，我们把这样的进程称为线程，线程之间共享同样的虚拟地址区域，fork()函数利用copy_mm()函数复制父进程的mm_struct，也就是current->mm域给其子进程；

```
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk);
```

用户空间的mmap函数实现原理：

```
#include <sys/mmap.h>
void *mmap(void *addr, size_t len, int port, int flags, int fd, off_t offset);
int munmap(void *addr, size_t len);
把一个文件映射到当前进程空间，或者设备，或者内存，这种映射比read/write效率高，因为不用内核空间到用户空间的拷贝，而mmap相当于内核空间和用户空间虚拟地址映射到了一个物理地址；
```

用户空间的mmap()会通过系统调用调用到内核的do_mmap()函数，do_mmap()首先创建一个新的vma区域，然后加入到进程的虚拟地址空间中，然后调用底层的mmap函数建立vma和实际物理地址的联系(建立页表)；

驱动的mmap建立虚拟地址和物理地址的映射，建立vma和物理地址的映射工作有remap_pfn_range来完成， 原型如下：

```
int remap_pfn_range(struct vm_area_struct *vma, unsigned long virt_addr, unsigned long pfn, unsigned long size, pgport_t port);
vma:需要建立映射的vma
virt_addr:需要建立映射的vma起始地址
pfn:页帧号，对应虚拟地址应当被映射的物理地址，这个页帧号简单的理解就是物理地址右移PAGE_SHIFT位
size:需要建立映射vma的大小，以字节
port:使用在vma->vm_page_port中找到的值
```



## (5)页面置换过程

在虚拟内存与物理内存的映射中，进程不一定会把进程的所有虚拟地址空间映射到物理空间，就是说页目录表中的内容可以是0，这时候表示虚拟地址没有映射到物理地址，当需要访问这个虚拟地址的时候，发现页目录项为0，就会产生缺页中断，使用伙伴算法将可用的物理页面映射到进程中；

## (6)OOM问题

当进程使用的内存大于物理内存+swap空间时，就会发生oom问题，由于系统压力，系统会选择保护一些程序，而将其他程序killed，释放内存，如何防止因为oom进程被killed的问题了？

* 禁止使用swap

  可以设置/proc/sys/vm/swappiness=0，禁止所有进程使用swap分区将页面换到磁盘上；

* 使用大页内存

* 可以设置/proc/sys/vm/overcommit_memory=2，在值为2的情况下，可以分配的内存最大大小在swap size + RAM*((/proc/sys/vm/overcommit_ratio) / 100))，如果大于这个直接返回错误；

* 如果发生了oom，可以使用下面的方法调整被killed

  设置/proc/xxx/oom_adj权值在-17-15之间，设置为0表示永远不会被killed，其余情况越大越容易被killed；

  /proc/xxx/oom_score是根据oom_adj计算出来的一个值，根据这个值选择谁被killed；

实验：

```
使用malloc分配大于swap+raw的内存，并一直向该内存中写数据，当swap用完之后就会发生oom现象；
```

## (7)内存使用量统计原理

