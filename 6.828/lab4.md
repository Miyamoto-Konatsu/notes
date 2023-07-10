### Implementing Copy-on-Write Fork

You now have the kernel facilities to implement copy-on-write `fork()` entirely in user space.



We have provided a skeleton for your `fork()` in `lib/fork.c`. Like `dumbfork()`, `fork()` should create a new environment, then scan through the parent environment's entire address space and set up corresponding page mappings in the child. The key difference is that, while `dumbfork()` copied *pages*, `fork()` will initially only copy page *mappings*. `fork()` will copy each page only when one of the environments tries to write it.

lib/fork.c里有一个fork函数的框架，和dubmfork类似。fork()首先创建一个新的进程环境，然后扫描父进程的整个地址空间，并且在子进程设置相应的页面映射。关键的不同点时dumbfork()会复制物理页，fork()仅仅初始化页表映射。fork()只会在一个进程尝试地去写页面时复制这个页面。

The basic control flow for `fork()` is as follows:

1. The parent installs `pgfault()` as the C-level page fault handler, using the `set_pgfault_handler()` function you implemented above.

   父进程使用set_pgfault_handler()安装页错误处理函数。

2. The parent calls `sys_exofork()` to create a child environment.

   父进程使用sys_exofork()创建子进程

3. For each writable or copy-on-write page in its address space below UTOP, the parent calls duppage

   , which should map the page copy-on-write into the address space of the child and then remap

    the page copy-on-write in its own address space. [ Note: The ordering here (i.e., marking a page as COW in the child before marking it in the parent) actually matters! Can you see why? Try to think of a specific case where reversing the order could cause trouble. ] duppage sets both PTEs so that the page is not writeable, and to contain

   PTE_COW . in the "avail" field to distinguish copy-on-write pages from genuine read-only pages.

   The exception stack is *not* remapped this way, however. Instead you need to allocate a fresh page in the child for the exception stack. Since the page fault handler will be doing the actual copying and the page fault handler runs on the exception stack, the exception stack cannot be made copy-on-write: who would copy it?

   `fork()` also needs to handle pages that are present, but not writable or copy-on-write.

   对于每一个地址在UTOP下面的可写页和copy-on-write页，父进程调用duppage()把copy-on-write页映射到子进程的地址空间，然后把这个重新映射到自己的空间。【先在子进程中把页标记为COW，然后在父进程中标记。这个顺序很重要】duppage ()把每个pte设置为不可写和PTE_COW 。PTE_COW 在pte的avail字段，用于取分copy-on-write页和只读页。异常栈不用这种方法重映射。代替的是应该从新为子进程分配一页当做异常栈。因为页错误处理函数会执行真正的复制并且它在异常栈上运行。异常栈不能是copy-on-write，那谁来复制他？fork()还需要处理不能写或者不是copy-on-write的但是存在的页。

4. The parent sets the user page fault entrypoint for the child to look like its own.

   父将子进程的用户页面错误入口点设置为其自己的入口点

5. The child is now ready to run, so the parent marks it runnable.

   设置子进程可运行

Each time one of the environments writes a copy-on-write page that it hasn't yet written, it will take a page fault. Here's the control flow for the user page fault handler:

每一次一个进程去写一个还没有写过的copy-on-write页时，会产生页错误。

1. The kernel propagates the page fault to `_pgfault_upcall`, which calls `fork()`'s `pgfault()` handler.

   内核把页错误传给`_pgfault_upcall`，`_pgfault_upcall`然后fork()的调用页错误处理函数。

2. `pgfault()` checks that the fault is a write (check for `FEC_WR` in the error code) and that the PTE for the page is marked `PTE_COW`. If not, panic.

   ​	页错误处理函数检测这个错误类型是不是写错误，并且检测这个页是不是被标记为copy-on-write。 如果没有标记为copy-on-write，说明这页只能读，不能写。

3. `pgfault()` allocates a new page mapped at a temporary location and copies the contents of the faulting page into it. Then the fault handler maps the new page at the appropriate address with read/write permissions, in place of the old read-only mapping.

   ​	页错误处理函数分配一个物理页，并把这个页映射到一个临时的地址，然后把写错误页的内容复制到新分配的页。

   然后把新页映射到正确的虚拟地址并设置正确的权限。

The user-level `lib/fork.c` code must consult the environment's page tables for several of the operations above (e.g., that the PTE for a page is marked `PTE_COW`). The kernel maps the environment's page tables at `UVPT` exactly for this purpose. It uses a [clever mapping trick](https://pdos.csail.mit.edu/6.828/2018/labs/lab4/uvpt.html) to make it to make it easy to lookup PTEs for user code. `lib/entry.S` sets up `uvpt` and `uvpd` so that you can easily lookup page-table information in `lib/fork.c`.

用户级的fork函数要执行以上的操作必须要查看页表获取信息，这就是为什么用户进程的页表映射到UVPT，使用一个巧妙的映射技巧使得用户代码可以容易地查看pte。lab/entry.s设置uvpt和uvpd，所以可以非常容易地查看页表信息。