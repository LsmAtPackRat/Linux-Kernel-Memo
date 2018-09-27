# Linux-Kernel-Memo



#### COW

COW的机制是内核拷贝父进程的页表内容到子进程的页表中（也就是仅拷贝了地址空间，而不拷贝页表所映射的物理页框），并将父子进程的**页表项**标记为只读，无论这个页本身是否可写，都标记为只读。这样才能在写这页的时候触发缺页异常，这样，内核就进入缺页异常处理函数，这个时候会通过检查vm_area_struct中的标志位，得知这页本身是可写的地址区间的页，进而判断出它是COW页，然后会复制此页的副本，并修改相应的页表项。

要点：如果没有缺页异常，那么内存访问是不会有OS介入的，而仅仅只是MMU查询多级查询页表进行地址转换的过程。所以如果想让OS介入，实现一些机制（例如COW），那么必须先在页表项上做手脚，引发缺页异常，然后到内核中进行情况的核查分类，然后才能做出相应的处理。所以可以把页表看成映射关系和事件触发器的功能合集。如果对一个页的write事件进行捕获，那么就修改该页页表项不可写即可。



#### 各种struct

##### thread_struct

thread_struct类型的结构的定义是体系结构相关的，它代表了**执行上下文**中的所有寄存器和其他信息。内核在进程切换时需要保存和恢复进程的内容，thread_struct结构就可以用于此目的。注意：内核中的thread和我们通常理解的thread不完全是一回事，内核中的thread经常用来指进程体系结构相关的部分。



#### 内核线程的active_mm

Linus Torvalds关于引入active_mm的解释：

> List:       linux-kernel
> Subject:    Re: active_mm
> From:       Linus Torvalds <torvalds () transmeta ! com>
> Date:       1999-07-30 21:36:24
>
> Cc'd to linux-kernel, because I don't write explanations all that often,
> and when I do I feel better about more people reading them.
>
> On Fri, 30 Jul 1999, David Mosberger wrote:
> >
> > Is there a brief description someplace on how "mm" vs. "active_mm" in
> > the task_struct are supposed to be used?  (My apologies if this was
> > discussed on the mailing lists---I just returned from vacation and
> > wasn't able to follow linux-kernel for a while).
>
> Basically, the new setup is:
>
>  - we have "real address spaces" and "anonymous address spaces". The
>    difference is that an anonymous address space doesn't care about the
>    user-level page tables at all, so when we do a context switch into an
>    anonymous address space we just leave the previous address space
>    active.
>
>    The obvious use for a "anonymous address space" is any thread that
>    doesn't need any user mappings - all kernel threads basically fall into
>    this category, but even "real" threads can temporarily say that for
>    some amount of time they are not going to be interested in user space,
>    and that the scheduler might as well try to avoid wasting time on
>    switching the VM state around. Currently only the old-style bdflush
>    sync does that.
>
>  - "tsk->mm" points to the "real address space". For an anonymous process,
>    tsk->mm will be NULL, for the logical reason that an anonymous process
>    really doesn't _have_ a real address space at all.
>
>  - however, we obviously need to keep track of which address space we
>    "stole" for such an anonymous user. For that, we have "tsk->active_mm",
>    which shows what the currently active address space is.
>
>    The rule is that for a process with a real address space (ie tsk->mm is
>    non-NULL) the active_mm obviously always has to be the same as the real
>    one.
>
>    For a anonymous process, tsk->mm == NULL, and tsk->active_mm is the
>    "borrowed" mm while the anonymous process is running. When the
>    anonymous process gets scheduled away, the borrowed address space is
>    returned and cleared.
>
> To support all that, the "struct mm_struct" now has two counters: a
> "mm_users" counter that is how many "real address space users" there are,
> and a "mm_count" counter that is the number of "lazy" users (ie anonymous
> users) plus one if there are any real users.
>
> Usually there is at least one real user, but it could be that the real
> user exited on another CPU while a lazy user was still active, so you do
> actually get cases where you have a address space that is _only_ used by
> lazy users. That is often a short-lived state, because once that thread
> gets scheduled away in favour of a real thread, the "zombie" mm gets
> released because "mm_users" becomes zero.
>
> Also, a new rule is that _nobody_ ever has "init_mm" as a real MM any
> more. "init_mm" should be considered just a "lazy context when no other
> context is available", and in fact it is mainly used just at bootup when
> no real VM has yet been created. So code that used to check
>
> 	if (current->mm == &init_mm)
>
> should generally just do
>
> 	if (!current->mm)
>
> instead (which makes more sense anyway - the test is basically one of "do
> we have a user context", and is generally done by the page fault handler
> and things like that).
>
> Anyway, I put a pre-patch-2.3.13-1 on ftp.kernel.org just a moment ago,
> because it slightly changes the interfaces to accommodate the alpha (who
> would have thought it, but the alpha actually ends up having one of the
> ugliest context switch codes - unlike the other architectures where the MM
> and register state is separate, the alpha PALcode joins the two, and you
> need to switch both together).

从以上解释可以看出：active_mm可以区分内核线程和普通的用户进程。并且内核线程可以借用任意用户进程的内核空间的部分，因为它在所有进程中的内容都是一样的。而同时引入了mm_users和mm_count两个计数器来处理各种使用情况，可以安全地回收mm_struct。



以下是Robert Love的解释：

> On Wed, 2003-04-23 at 16:41, Bryan K. wrote:
>
> > Reading the kernel source I have noticed that a kernel thread must always 
> > have a valid mm_struct pointer at task_struct->active_mm. This prevents the 
> > mm_struct to be deallocated if a kernel thread uses it. I was wandering why 
> > a kernel thread need a mm_struct if it is not supposed to access addresses 
> > below TASK_SIZE?
>
> It does not need (or have) an mm_struct. task->mm is NULL in a kernel
> thread (task->mm == NULL is what a kernel thread is, by definition, in
> fact).
>
> task->active_mm is just the current address space that is mapped in. It
> is an optimization, so we can have something like:
>
> task A is running and has an mm_struct of foo, thus
> A->mm == A->active_mm == foo
>
> task A is preempted by a kernel thread, B. Thus,
> B->mm == NULL and B->active_mm == foo,
> since the mm_struct was never replaced.
>
> Now, if task A is rescheduled the mm need not be reloaded.
>
> Make sense?
>
> Robert Love

从以上解释可以看出：让内核线程借用前一个进程的mm_struct是一种优化手段。如果task A被kernel thread B抢占，然后再重新调度到A，那么此时mm不需要被重新加载（无需调用switch_mm()），TLB也都不用刷新（enter_lazy_tlb()函数），属于一种Lazy优化的技术。



> Hello Roy
>
> > I fail to understand the difference between task->mm and
> > task->active_mm. I've noticed that upon forking a task, both mm and
> > active_mm get the same memory descriptor.
>
> Well, here is my understanding. task_struct->mm points to memory 
> descriptor which is unique to each process (unless they are on the same 
> thread group, forked with CLONE_VM). active_mm points to the *actual* 
> memory descriptor used by the process when it is executed.
>
> So why it is separated? IMHO the reason is to identify which process is 
> kernel thread (doesn't own a process address space) and which one is 
> normal process (owns a process address space). As you can see on 
> functions related with context switching, by checking task_struct->mm, 
> the scheduler can decide whether it is going to switch onto kernel 
> thread or not. if it is NULL, then the process doesn't have process address space, in 
> other word this is a kernel thread. But you also aware that even kernel 
> threads don't acess user space memory, it still needs to access kernel 
> space. because kernel space is 100% identical for every process, kernel 
> thread can freely use memory descriptor (mm) owned by previously 
> running process. All the kernel thread needed is page tables 
> referencing toward virtual address bigger than PAGE_OFFSET, other are 
> simply ignored by it is assumed that kernel thread doesn't need to 
> access user space (perhaps it is somehow can be abused?)
>
> Hope it helps answering your question
>
> regards
>
> Mulyadi

以上的意思是：kernel thread需要之前的mm_struct的页表的内核部分，虽然mm_struct是用户地址空间的结构，但是页表包含了内核空间的映射。

上面的说法有待看代码进行验证。

[一篇帖子](http://www.phpfans.net/ask/fansa1/4044766844.html)：

```c
if (unlikely(!mm)) {
	next->active_mm = oldmm;
	atomic_inc(&oldmm->mm_count);
	enter_lazy_tlb(oldmm, next, smp_processor_id());
} else
	switch_mm(oldmm, mm, next, smp_processor_id());

if (unlikely(!prev->mm)) {
	prev->active_mm = NULL;
	BUG_ON(rq->prev_mm);
	rq->prev_mm = oldmm;
}  
```

以上的代码部分印证了Linus和Robert Love的说法。



#### sysfs

与proc伪文件系统类似的一种虚拟文件系统，但是要设计得更加清晰。挂载点为/sys/，可以将系统的各种属性描述导出到用户空间。



#### rlimit

rlimit是一种限制进程使用资源的机制。每一个进程都有对应的资源使用限制，具体内容可以查看/proc/pid/rlimits文件。init进程的限制在系统启动时即生效，定义在include/asm-generic-resource.h的INIT_RLIMITS中。