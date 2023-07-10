## 介绍

在这个实验需要实现能使受保护的用户进程运行的基本内核功能。增强JOS，内核建立起数据结构来跟踪用户进程，创建一个用户进程，然后加载一个程序镜像到这个进程。还需要使JOS内核能够处理用户进程调用的各种系统调用和它产生的各种异常。



## Part A:用户进程与异常处理

新的头文件`inc/env.h`包含了JOS对于用户进程的基本定义。内核使用`Env`数据结构跟踪每个用户进程。这个实验只需要创建一个用户进程，但是需要设计JOS内核去支持多进程。lab4中会利用这个特性去`fork`用户进程。

`kern/env.c`中定义了3个关于进程的全局变量。

```c
struct Env *envs = NULL;		// All environments
struct Env *curenv = NULL;		// The current env
static struct Env *env_free_list;	// Free environment list
```

一旦JOS启动并运行，envs指针将指向一个表示系统中所有进程的Env结构数组。在我们的设计中，JOS内核将最多支持NENV同时活动的进程，尽管在任何给定时间运行的进程通常要少得多(NENV是一个常量，在`inc/env.h`中定义。）一旦分配，envs数组的NENV个可能的进程的都会包含一个Env数据结构的实例。
JOS内核在Env_free_列表中保留所有非活动的Env结构。这种设计允许轻松分配和取消分配进程，因为它们只需添加到自由列表或从自由列表中删除即可。
内核使用curenv符号随时跟踪当前执行的进程。在启动期间，在运行第一个进程之前，curenv最初设置为NULL。

### 进程状态

inc/Env.h中对Env结构的定义如下（在的实验中将添加更多字段）：

```c
struct Env{
    struct Trapframe env_tf；//保存的寄存器
    struct Env*Env_link；//下一个可用进程
    envid_t env_id；//唯一的进程标识符
    envid_t env_parent_id；//此env的父项的env\u id
    enum EnvType env_type；//表示特殊的系统进程
    unsigned env_status；//进程的状态
    uint32_t env_runs；//进程已运行的次数
        
    //地址空间
    pde_t *env_pgdir；//页面目录的内核虚拟地址
};
```


以下是进程每个字段的用途：
env_tf：
		此结构在inc/trap.h中定义，在进程未运行时（即内核或其他进程正在运行时）保存进程的已保存寄存器值。内核在从用户模式切换到内核模式时会保存这些信息，以便以后可以在停止的位置恢复进程。
env_list：
		这是指向“进程自由”列表中下一个进程的链接。env_free_list指向列表中的第一个可用进程。
env_id：
		内核在此存储一个值，该值唯一标识当前使用此Env结构的进程（即使用envs数组中的此特定插槽）。用户进程终止后，内核可能会将相同的Env结构重新分配给不同的进程，但是新进程将具有与旧进程不同的Env_id，即使新进程正在重新使用envs数组中的相同插槽。
env_parent_id:
		内核在这里存储创建此进程的进程的env_id。通过这种方式，进程可以形成一个“家族树”，这将有助于做出安全决策，即允许哪些进程对谁做什么。
env_type：
		这用于区分特殊进程。对于大多数进程，它将是ENV_TYPE_USER。我们将在以后的实验室中为特殊的系统服务进程介绍更多类型，比如ENV_TYPE_FILE。
env_status：
		此变量包含以下值之一：
		ENV_FREE：
				指示进程结构处于非活动状态，因此位于进程空闲列表中。
		ENV_RUNNABLE：
				指示Env结构表示等待在处理器上运行的进程。
		ENV_RUNNING：
				指示Env结构表示当前运行的进程。
		ENV_NOT_RUNNABLE：
				指示Env结构表示当前活动的进程，但它当前尚未准备好运行：例如，它正在等待来自另一个进程的进程间通信		（IPC）。
		ENV_DYING：
				指示Env结构表示僵尸进程。僵尸进程将在下次捕获到内核时被释放。在实验4之前，不会使用此标志。
env_pgdir：
		此变量保存此进程的页面目录的内核虚拟地址。
与Unix进程一样，JOS进程将“线程”和“地址空间”的概念结合起来。线程主要由保存的寄存器（env_tf字段）定义，地址空间由env_pgdir指向的页面目录和页面表定义。要运行进程，内核必须使用保存的寄存器和适当的地址空间设置CPU。
我们的struct Env类似于xv6中的struct proc。这两种结构都将进程的用户模式寄存器状态保存在Trapframe结构中。在JOS中，各个进程不像xv6中的进程那样有自己的内核堆栈。内核中一次只能有一个活动的JOS进程，因此JOS只需要一个内核堆栈。

###  分配进程数组

In lab 2, you allocated memory in `mem_init()` for the `pages[]` array, which is a table the kernel uses to keep track of which pages are free and which are not. You will now need to modify `mem_init()` further to allocate a similar array of `Env` structures, called `envs`.

在lab2，已经在`mem_init()`已经分配了一个pages的数组，内核用这个数组追踪页面的分配情况。修改`mem_init()`，分配一个类似的Env结构体数组，叫作envs。

```c
	// Make 'envs' point to an array of size 'NENV' of 'struct Env'.
	// LAB 3: Your code here.
	envs = (struct Env *) boot_alloc(sizeof(struct Env) * NENV);
	memset(envs, 0, sizeof(struct Env) * NENV);
	
	// Map the 'envs' array read-only by the user at linear address UENVS
	// (ie. perm = PTE_U | PTE_P).
	// Permissions:
	//    - the new image at UENVS  -- kernel R, user R
	//    - envs itself -- kernel RW, user NONE
	// LAB 3: Your code here.
	boot_map_region(kern_pgdir, UENVS, PTSIZE, PADDR(envs), PTE_U);
	
```

### 创建和运行进程

因为现在还没有文件系统，所以我们设置内核去加载一个嵌入在内核本身的静态二进制映像。JOS把这个二进制文件以ELF可执行映像嵌入到内核。

In `i386_init()` in `kern/init.c` you'll see code to run one of these binary images in an environment. However, the critical functions to set up user environments are not complete; you will need to fill them in.

在kern/init.c的`i386_init()`，有代码去在一个进程中运行一个二进制映像。在此之前需要完成关键的函数去创建一个用户进程。

1. env_init()

   初始化所有的Env结构体，把他们加入env_free_list。还要调用env_init_percpu，用不同的权限等级配置分段硬件。内核是privilege level 0,

   用户进程是privilege level 3。

   ```c
   void
   env_init(void)
   {
   	// Set up envs array
   	// LAB 3: Your code here.
   	env_free_list = envs;
   	envs->env_id = 0;
   	envs->env_status = ENV_FREE;
   	for(int i = 1; i < NENV; ++i) {
   		(envs + i)->env_id = 0;
   		(envs + i)->env_status = ENV_FREE;
   		(envs + i - 1)->env_link = envs + i;
   	}
   	// Per-CPU part of the initialization
   	env_init_percpu();
   }
   
   ```




2. env_setup_vm()

   为新进程分配一个页目录，并初始化内核部分的地址空间。一般不需要记录虚拟地址在UTOP以上页面引用数，但是为了以后进程销毁时能顺利释放这个页目录页，所以增加这个物理页的引用数。 然后复制内核的地址空间。最后把UVPT映射到当前进程的页目录物理地址e->env_pgdir处。

   ```c
   	if (!(p = page_alloc(ALLOC_ZERO)))
   		return -E_NO_MEM;
   	// LAB 3: Your code here.
   	e->env_pgdir =(pde_t *) page2kva(p);
   	++p->pp_ref;
   	memcpy(e->env_pgdir, kern_pgdir, PGSIZE);
   	// UVPT maps the env's own page table read-only.
   	// Permissions: kernel R, user R
   	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;
   ```

3. region_alloc()

   为进程以虚拟地址va开始的len字节的地址分配并映射物理页.

   ```c
   static void
   region_alloc(struct Env *e, void *va, size_t len)
   {
   	// LAB 3: Your code here.
   	// (But only if you need it for load_icode.)
   	//
   	// Hint: It is easier to use region_alloc if the caller can pass
   	//   'va' and 'len' values that are not page-aligned.
   	//   You should round va down, and round (va + len) up.
   	//   (Watch out for corner-cases!)
   	void *end = ROUNDUP(va + len, PGSIZE);
   	va = ROUNDDOWN(va, PGSIZE);
   	while (va < end)
   	{
   		struct PageInfo* pp = page_alloc(0);
   		if(!pp)
   			panic("page_alloc error! There is no enough physical memory left!\n");
   		if(page_insert(e->env_pgdir, pp, va, PTE_W | PTE_U))
   			panic("page_insert error! There is no enough physical memory left!\n");
   		va += PGSIZE;
   	}
   }
   ```

4.  load_icode()

   解析一个ELF二进制映像，把它的内容加载到用户进程中。 在进程的地址空间里为ELF映像中需要加载的段分配内存，将这些段的内容复制到分配的内存里，前置要求将这个进程的页目录的物理地址放到lcr3寄存器。将env的eip改成这个映像的入口地址。最后分配一页空间作为进程的栈。

   ```c
   static void
   load_icode(struct Env *e, uint8_t *binary)
   {
   	// LAB 3: Your code here.
   	struct Elf * elf = (struct Elf *) binary;
   
   	if(elf->e_magic != ELF_MAGIC)
   		panic("It is not an elf binary!\n");
   	struct Proghdr* ph, *eph;
   	ph = (struct Proghdr *)((uint8_t*) elf + elf->e_phoff);
   	eph = ph + elf->e_phnum;
   
   	lcr3(PADDR(e->env_pgdir));
   	for(; ph < eph; ph++) {
   		if(ph->p_type != ELF_PROG_LOAD) 
   			continue;
   		region_alloc(e, (void *)ph->p_va, ph->p_memsz);
   		memset((void *)ph->p_va, 0, ph->p_memsz);
   		memcpy ((void *)ph->p_va, (void *)( binary + ph->p_offset), ph->p_filesz);
   	}
   
   	e->env_tf.tf_eip = elf->e_entry;
   	// Now map one page for the program's initial stack
   	// at virtual address USTACKTOP - PGSIZE.
   
   	// LAB 3: Your code here.
   	struct PageInfo* pp = page_alloc(0);
   	if(!pp)
   		panic("There is no enough physical memory!\n");
   	page_insert(e->env_pgdir, pp,(void*) USTACKTOP - PGSIZE, PTE_W | PTE_U);
   	lcr3(PADDR(kern_pgdir));
   }
   ```

5. env_create()

   调用env_alloc()和load_icode()创建一个进程。

   ```c
   void
   env_create(uint8_t *binary, enum EnvType type)
   {
   	// LAB 3: Your code here.
   	struct Env* env;
   	if(env_alloc(&env, 0))
   		panic("env_alloc fail!\n");
   	load_icode(env, binary);
   	env->env_type = type;
   }
   ```

6. env_run()

   运行给定的进程。将curenv指向需要运行的进程指针，然后页目录寄存器的值设为该进程的页目录物理地址，最后恢复调用env_pop_tf函数。

   ```c
   void
   env_run(struct Env *e)
   {
   	// LAB 3: Your code here.
   	if(curenv){
   		if(curenv->env_status == ENV_RUNNING)
   			curenv->env_status = ENV_RUNNABLE;
          	if(curenv != e)
   			curenv->env_runs += 1;
   	} 
   	
   	curenv = e;
   	curenv->env_status = ENV_RUNNING;
   	lcr3(PADDR(curenv->env_pgdir));
   
   	env_pop_tf(&curenv->env_tf);
   ```

   env_pop_tf（）函数的函数执行过程如下：
   
   1. 将该进程的tramframe所在的地址传到栈顶寄存器，所以此时栈顶执行tramframe的地址。
   2. 然后popal恢复通用寄存器的值。
   3. 恢复es，ds的值。
   4. 跳过trap代号和错误代码，这两个数值已经没用。
   5. 最后iret指令恢复eip，cs，eflags，esp，ss寄存器。
   
   ```
   void
   env_pop_tf(struct Trapframe *tf)
   {
   	// Record the CPU we are running on for user-space debugging
   	curenv->env_cpunum = cpunum();
   
   	asm volatile(
   		"\tmovl %0,%%esp\n"
   		"\tpopal\n"
   		"\tpopl %%es\n"
   		"\tpopl %%ds\n"
   		"\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
   		"\tiret\n"
   		: : "g" (tf) : "memory");
   	panic("iret failed");  /* mostly to placate the compiler */
   }
   ```
   
   

###  处理中段和异常

​		用户空间中的第一个INT $ 0x30系统调用指令是一个死胡同：一旦处理器进入用户模式，就无法退出。现在需要实现基本的异常和系统调用处理，以便内核可以从用户模式代码中恢复处理器的控制。 

### 受保护控制传输的基础

​		异常和中断都是“受保护的控制传输”，这会导致处理器从用户模式切换到内核模式（CPL=0），而不会给用户模式代码任何干扰内核或其他进程功能的机会。在英特尔的术语中，中断是一种受保护的控制传输，由处理器外部的异步事件（如外部设备I/O活动通知）引起。相反，异常是由当前运行的代码同步引起的受保护的控制传输，例如，由于被零除或无效的内存访问。
​		为了确保这些受保护的控制传输得到实际保护，处理器的中断/异常机制被设计为在中断或异常发生时当前运行的代码不能随意选择内核的进入位置或进入方式。相反，处理器确保只能在被控制的条件下进入内核。在x86上，两种机制共同提供此保护：

1. ​	中断描述符表。处理器确保中断和异常只能导致内核在几个特定的、定义严格的入口点进入，这些入口点由内核本身决定，而不是由执行中断或异常时运行的代码决定。
   ​	x86允许多达256个不同的中断或异常入口点进入内核，每个入口点具有不同的中断向量。向量是介于0和255之间的数字。中断向量由中断源决定：不同的设备、错误条件和对内核的应用程序请求生成具有不同向量的中断。CPU使用向量作为处理器中断描述符表（IDT）的索引，内核将其设置在内核专用内存中，与GDT非常相似。处理器从该表中的相应条目加载：

   - 要加载到指令指针（EIP）寄存器的值，指向用于处理该类型异常的内核代码。

   - 要加载到代码段（CS）寄存器中的值，它在位0-1中包含异常处理程序运行时的特权级别。（在JOS中，所有异常都在内核模式下处理，特权级别为0。）

2. ​    任务状态段。处理器需要一个位置来保存中断或异常发生之前的旧处理器状态，例如处理器调用异常处理程序之前的EIP和CS的原始值，以便异常处理程序可以稍后恢复该旧状态并从中断处恢复中断的代码。但是，旧处理器状态的这个保存区域反过来必须受到保护，不受非特权用户模式代码的影响；否则，错误或恶意用户代码可能会危害内核。
   ​	 因此，当x86处理器发生中断或陷阱，导致特权级别从用户模式更改为内核模式时，它也会切换到内核内存中的堆栈。称为任务状态段（TSS）的结构指定该堆栈所在的段选择器和地址。处理器（在此新堆栈上）推送SS、ESP、EFLAGS、CS、EIP和可选错误代码。然后从中断描述符加载CS和EIP，并将ESP和SS设置为引用新堆栈。
   ​	 尽管TSS很大，并且可能服务于多种用途，但JOS仅使用它来定义处理器在从用户模式转换到内核模式时应切换到的内核堆栈。由于JOS中的“内核模式”在x86上是特权级别0，因此处理器在进入内核模式时使用TSS的ESP0和SS0字段来定义内核堆栈。JOS不使用任何其他TSS字段。



### 异常和中断的类型
​		x86处理器内部生成的所有同步异常使用0到31之间的中断向量，因此映射到IDT条目0-31。例如，页面错误总是通过向量14导致异常。大于31的中断向量仅用于软件中断，这些中断可以由int指令生成，也可以由外部设备在需要注意时引起的异步硬件中断生成。
​		在本节中，我们将扩展JOS以处理向量0-31中内部生成的x86异常。在下一节中，我们将使JOS处理软件中断向量48（0x30），JOS（相当任意地）将其用作其系统调用中断向量。在实验室4中，我们将扩展JOS以处理外部生成的硬件中断，如时钟中断。

### 示例

​		将这些片段放在一起，并通过一个示例进行跟踪。假设处理器在用户环境中执行代码，并遇到一条试图除以零的divide指令。

1. 处理器切换到TSS的SS0和ESP0字段定义的堆栈，在JOS中，该堆栈将分别保存值GD_KD和KSTACKTOP。

2. 处理器将异常参数推送到内核堆栈上，从地址KSTACKTOP开始：

   ```assembly
                        +--------------------+ KSTACKTOP             
                        | 0x00000 | old SS   |     " - 4
                        |      old ESP       |     " - 8
                        |     old EFLAGS     |     " - 12
                        | 0x00000 | old CS   |     " - 16
                        |      old EIP       |     " - 20 <---- ESP 
                        +--------------------+             
   ```

3. 因为我们正在处理一个除法错误，即x86上的中断向量0，所以处理器读取IDT条目0并将CS:EIP设置为指向该条目所描述的处理函数。

4. handler函数控制并处理异常，例如通过终止用户环境。



​		对于某些类型的x86异常，除了上面的“标准”五个字之外，处理器还会将另一个包含错误代码的字推送到堆栈上。页面错误异常14号是一个重要的例子。堆栈在异常处理程序的开头如下所示：

```assembly
                     +--------------------+ KSTACKTOP             
                     | 0x00000 | old SS   |     " - 4
                     |      old ESP       |     " - 8
                     |     old EFLAGS     |     " - 12
                     | 0x00000 | old CS   |     " - 16
                     |      old EIP       |     " - 20
                     |     error code     |     " - 24 <---- ESP
                     +--------------------+             
```

### 多重异常和中断

​		处理器可以从内核模式和用户模式接受异常和中断。然而，只有在从用户模式进入内核时，x86处理器才会在将其旧寄存器状态推送到堆栈上并通过IDT调用适当的异常处理程序之前自动切换堆栈。当中断或异常发生时，如果处理器已经处于内核模式（CS寄存器的低位2位已经为零），那么CPU只是在同一内核堆栈上推送更多值。通过这种方式，内核可以优雅地处理由内核本身中的代码引起的嵌套异常。此功能是实现保护的一个重要工具，我们将在后面的系统调用部分中看到。
如果处理器已经处于内核模式，并且发生嵌套异常，因为它不需要切换堆栈，它不会保存旧的SS或ESP寄存器。因此，对于不推送错误代码的异常类型，内核堆栈在异常处理程序的条目中看起来如下所示：

```assembly
                     +--------------------+ <---- old ESP
                     |     old EFLAGS     |     " - 4
                     | 0x00000 | old CS   |     " - 8
                     |      old EIP       |     " - 12
                     +--------------------+             
```

​		对于推送错误代码的异常类型，处理器会像以前一样，在旧EIP之后立即推送错误代码。

### 设置IDT

​		设置IDT来处理中断向量0-31（处理器异常）。我们将在本实验室稍后处理系统调用中断，并在稍后的实验室中添加中断32-47（设备IRQ）。
​		头文件inc/trap.h和kern/trap.h包含与中断和异常相关的重要定义，您需要熟悉这些定义。文件kern/trap.h包含对内核严格私有的定义，而inc/trap.h包含对用户级程序和库可能有用的定义。
​		注意：0-31范围内的某些异常由Intel定义为保留。因为它们永远不会由处理器生成，所以如何处理它们并不重要。

总体控制流程如下所示：

```assembly
      IDT                   trapentry.S         trap.c
   
+----------------+                        
|   &handler1    |---------> handler1:          trap (struct Trapframe *tf)
|                |             // do stuff      {
|                |             call trap          // handle the exception/interrupt
|                |             // ...           }
+----------------+
|   &handler2    |--------> handler2:
|                |            // do stuff
|                |            call trap
|                |            // ...
+----------------+
       .
       .
       .
+----------------+
|   &handlerX    |--------> handlerX:
|                |             // do stuff
|                |             call trap
|                |             // ...
+----------------+
```

​		每个异常或中断都应该在trapentry.S中有自己的处理程序，trap_init（）应该使用这些处理程序的地址初始化IDT。每个处理程序都应该在堆栈上构建一个struct Trapframe（参见inc/trap.h），并使用指向Trapframe的指针调用trap（）（在trap.c中）。trap（）然后处理异常/中断或分派给特定的处理函数。



​		编辑trapentry.S和trap.c并实现上述功能。在trapentry.S中为inc/trap.h中定义的每个陷阱添加一个入口点，提供TRAPHANDLER宏所引用的所有陷阱。您还需要修改trap_init（）以初始化idt，使其指向trapentry.S中定义的每个入口点。



​        首先声明trap的处理函数，使用SETGATE把它们的地址填到中断描述表相应的条目中 。本人才疏学浅，调用门的知识我目前还不是很了解，内容待补充。

```c
	void divide_handler();
	void debug_handler();
	void nmi_handler();
	void brkpt_handler();
	void overflow_handler();
	void illegalop_handler();
	void bounds_handler();
	void device_handler();
	void double_handler();
	void taskswitch_handler();
	void stack_handler();
	void protection_handler();
	void page_handler();
	void machine_handler();
	void floating_handler();
	void aligment_handler();
	void simd_handler();
	void syscall_handler();
	void segment_handler();

	void timer_handler();
	SETGATE(idt[T_DIVIDE],0,GD_KT,divide_handler,0);
	SETGATE(idt[T_DEBUG],0,GD_KT,debug_handler,0);
	SETGATE(idt[T_NMI],0, GD_KT,nmi_handler,0);
	SETGATE(idt[T_BRKPT],0,GD_KT,brkpt_handler,3);
	SETGATE(idt[T_OFLOW],0,GD_KT,overflow_handler,0);
	SETGATE(idt[T_BOUND],0,GD_KT,bounds_handler,0);
	SETGATE(idt[T_ILLOP],0,GD_KT,illegalop_handler,0);
	SETGATE(idt[T_DEVICE],0,GD_KT,device_handler,0);
	SETGATE(idt[T_DBLFLT],0,GD_KT,double_handler,0);
	SETGATE(idt[T_TSS],0,GD_KT,taskswitch_handler,0);
	SETGATE(idt[T_SEGNP],0,GD_KT,segment_handler,0);
	SETGATE(idt[T_STACK],0,GD_KT,stack_handler,0);
	SETGATE(idt[T_GPFLT],0,GD_KT,protection_handler,0);
	SETGATE(idt[T_PGFLT],0,GD_KT,page_handler,0);
	SETGATE(idt[T_FPERR],0,GD_KT,floating_handler,0);
	SETGATE(idt[T_ALIGN],0,GD_KT,aligment_handler,0);
	SETGATE(idt[T_MCHK],0,GD_KT,machine_handler,0);
	SETGATE(idt[T_SIMDERR],0,GD_KT,simd_handler,0);
```



个人认为，在前面声明了traphandler之后，下面汇编语言的功能是定义这些函数。如果有些类型的trap会压栈错误代码，就用`#define TRAPHANDLER(name, num)`定义这个函数，如果有些trap没有`error code`，使用`#define TRAPHANDLER(name, num)`定义相应的traphandler，会压栈一个0来保持两种类型trap的栈的格式相同。然后两种handler都会压栈trap类型的代号，最后执行_alltraps。

```assembly
/* TRAPHANDLER defines a globally-visible function for handling a trap.
 * It pushes a trap number onto the stack, then jumps to _alltraps.
 * Use TRAPHANDLER for traps where the CPU automatically pushes an error code.
 *
 * You shouldn't call a TRAPHANDLER function from C, but you may
 * need to _declare_ one in C (for instance, to get a function pointer
 * during IDT setup).  You can declare the function with
 *   void NAME();
 * where NAME is the argument passed to TRAPHANDLER.
 */
#define TRAPHANDLER(name, num)						\
	.globl name;		/* define global symbol for 'name' */	\
	.type name, @function;	/* symbol type is function */		\
	.align 2;		/* align function definition */		\
	name:			/* function starts here */		\
	pushl $(num);	/* cpu已经自动压栈错误代码，所以就只用压栈trap类型代号了*/		\
	jmp _alltraps

/* Use TRAPHANDLER_NOEC for traps where the CPU doesn't push an error code.
 * It pushes a 0 in place of the error code, so the trap frame has the same
 * format in either case.
 */
#define TRAPHANDLER_NOEC(name, num)					\
	.globl name;							\
	.type name, @function;						\
	.align 2;							\
	name:								\
	pushl $0;	/* 没有错误代码的trap压栈一个0使得与压栈错误代码的trap有相同的栈帧格式*/ 	\
	pushl $(num);							\
	jmp _alltraps
	
.text
	TRAPHANDLER_NOEC(divide_handler, T_DIVIDE);
	TRAPHANDLER_NOEC(debug_handler, T_DEBUG);
	TRAPHANDLER_NOEC(nmi_handler, T_NMI);
	TRAPHANDLER_NOEC(brkpt_handler, T_BRKPT);
	TRAPHANDLER_NOEC(overflow_handler, T_OFLOW);
	TRAPHANDLER_NOEC(bounds_handler, T_BOUND);
	TRAPHANDLER_NOEC(illegalop_handler, T_ILLOP);
	TRAPHANDLER_NOEC(device_handler, T_DEVICE);

	TRAPHANDLER(double_handler, T_DBLFLT);
	TRAPHANDLER(taskswitch_handler, T_TSS);
	TRAPHANDLER(segment_handler, T_SEGNP);
	TRAPHANDLER(stack_handler, T_STACK);
	TRAPHANDLER(protection_handler, T_GPFLT);
	TRAPHANDLER(page_handler, T_PGFLT);

	TRAPHANDLER_NOEC(floating_handler, T_FPERR);
	TRAPHANDLER_NOEC(aligment_handler, T_ALIGN);
	TRAPHANDLER_NOEC(machine_handler, T_MCHK);
	TRAPHANDLER_NOEC(simd_handler, T_SIMDERR);
```



​		最后是所以traphandler相同的部分，首先安装`struct TrapFrame`的格式把剩下的内容压栈。然后把内核数据段的选择子加载到数据段和附加段寄存器中，之前从用户模式到内核模式时，已经把代码段和栈段寄存器的值更改为内核段的选择子了。最后把栈顶制作入栈作为trap函数的参数`struct Trapframe *tf`的值，并调用trap函数。`call trap`之后`trap`不会返回。

```assembly
 _alltraps:
	pushl %ds // 压栈ds
	pushl %es // 压栈ds
	pushal  // 压栈通用寄存器 ，此时trapframe所有的成员都已经压栈
	
	movl $GD_KD, %eax 
	movw %ax, %ds 
	movw %ax, %es
	
	pushl %esp
	call trap
```

## Part B：页面错误、断点异常和系统调用

### 处理页面错误

​		当处理器出现页面错误时，它将导致错误的线性（即虚拟）地址存储在一个特殊的处理器控制寄存器CR2中。在trap.c中，page_fault_handler（）用于处理页面错误异常。修改trap_dispatch（）以将页面错误异常分派给页面错误处理程序。

```c
	if(tf->tf_trapno == T_PGFLT) {
		page_fault_handler(tf);
	}
```

### 断点异常

​		断点异常中断向量3（T_BRKPT）通常用于允许调试器通过临时将相关程序指令替换为特殊的1字节int3软件中断指令，在程序代码中插入断点。在JOS中，我们将稍微滥用此异常，将其转换为任何用户环境都可以用来调用JOS内核监视器的原始伪系统调用。如果我们将JOS内核监视器视为一个基本的调试器，那么这种用法实际上有点合适。例如，lib/panic.c中的panic（）的用户模式实现在显示其panic消息后执行int3。

​		修改trap_dispatch（）以使断点异常调用内核监视器。

```c
	else if(tf->tf_trapno == T_BRKPT) {
		monitor(tf);
		return;
	}
```

### 系统调用

​		应用程序将在寄存器中传递系统调用号和系统调用参数。这样，内核就不需要在用户环境的堆栈或指令流中四处搜索。系统调用号将位于%eax中，参数（最多五个）将分别位于%edx、%ecx、%ebx、%edi和%esi中。内核将返回值传回%eax。

```c
	应用程序调用lib/syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)这个系统调用，传入参数。
    然后执行下面这些指令，把参数传到相应的寄存器，然后int %1触发系统调用trap，进入内核态执行有关指令，最后结果保存在eax，并把eax的值传到ret。
    	asm volatile("int %1\n"
		     : "=a" (ret)
		     : "i" (T_SYSCALL),
		       "a" (num),
		       "d" (a1),
		       "c" (a2),
		       "b" (a3),
		       "D" (a4),
		       "S" (a5)
		     : "cc", "memory");

	进入内核后根据trapno，调用syscall。结果保存在tf->tf_regs.reg_eax里，从内核返回用户进程之后eax的值为tf->tf_regs.reg_eax的值。
        
	if(tf->tf_trapno == T_SYSCALL) {
		tf->tf_regs.reg_eax = syscall(tf->tf_regs.reg_eax, tf->tf_regs.reg_edx, tf->tf_regs.reg_ecx,
		tf->tf_regs.reg_ebx,tf->tf_regs.reg_edi,tf->tf_regs.reg_esi);
		return;
	}

内核系统调用函数根据syscallno调用相应的函数。
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
	// Call the function corresponding to the 'syscallno' parameter.
	// Return any appropriate return value.
	// LAB 3: Your code here.

	// panic("syscall not implemented");

	switch (syscallno) {
		case(SYS_cputs):
			user_mem_assert(curenv, (void*)a1, a2, PTE_U);
			sys_cputs((void*) a1, a2);
			return 0;
		case(SYS_cgetc):
			return sys_cgetc();
		case(SYS_getenvid):
			return sys_getenvid();
		case(SYS_env_destroy):
			return sys_env_destroy(a1);
		default:
			return -E_INVAL;
	}
}
```

### 用户模式启动

​		用户程序从lib/entry.S的顶部开始运行。经过一些设置后，此代码调用lib/libmain.c中的libmain（）。

```c
#include <inc/mmu.h>
#include <inc/memlayout.h>

.data
// Define the global symbols 'envs', 'pages', 'uvpt', and 'uvpd'
// so that they can be used in C as if they were ordinary global arrays.
// 定义一些全局变量，以被用户程序使用
	.globl envs
	.set envs, UENVS
	.globl pages
	.set pages, UPAGES
	.globl uvpt
	.set uvpt, UVPT
	.globl uvpd
	.set uvpd, (UVPT+(UVPT>>12)*4)


// Entrypoint - this is where the kernel (or our parent environment)
// starts us running when we are initially loaded into a new environment.
.text
.globl _start
_start:
	// See if we were started with arguments on the stack
	// 当用户程序是从内核中加载来的，此时没有传递给用户程序argc和argv。所以esp = USTACKTOP
	// 其他情况时，比如用户程序是从文件系统中读出再调用时，此时会传递参数，所以esp < USTACKTOP
	cmpl $USTACKTOP, %esp
	jne args_exist

	// If not, push dummy argc/argv arguments.
	// This happens when we are loaded by the kernel,
	// because the kernel does not know about passing arguments.
    // 没有参数时压栈默认参数
	pushl $0
	pushl $0

args_exist:
	call libmain
1:	jmp 1b
```



​			初始化全局指针thisenv，使其指向envs[]数组中此环境的struct Env。

```
lib/libmain.c:
    // set thisenv to point at our Env structure in envs[].
	// LAB 3: Your code here.
	envid_t vid =  sys_getenvid();
	thisenv = &envs[ENVX(vid)];
```

### 页面错误和内存保护

​		内存保护是操作系统的一项关键功能，确保一个程序中的错误不会损坏其他程序或损坏操作系统本身。
​		操作系统通常依靠硬件支持来实现内存保护。操作系统会通知硬件哪些虚拟地址有效，哪些无效。当程序试图访问无效地址或没有权限访问的地址时，处理器会在导致故障的指令处停止程序，然后将有关尝试操作的信息放入内核。如果故障是可修复的，内核可以修复它并让程序继续运行。如果故障无法修复，则程序将无法继续，因为它将永远无法通过导致故障的指令。
​		在许多系统中，内核最初分配一个堆栈页，然后如果程序访问堆栈下的页面时出错，内核将自动分配这些页面并让程序继续。通过这样做，内核只分配程序所需的堆栈内存，但程序可以在其堆栈任意大的假象下工作。

内核态发生错误，就会panic。

```
// LAB 3: Your code here.
	if((tf->tf_cs & 3) == 0){
			panic("kernel occurs page falut! va %08x ip %08x\n",rcr2(),tf->tf_eip);
	}
```

读取kern/pmap.c中的user_mem_assert，并在同一文件中实现user_mem_check。

```c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
	// LAB 3: Your code here.
	if(len == 0) return 0;

	uintptr_t t_va = (uintptr_t)(va);
	if(t_va +len >= ULIM) {
		user_mem_check_addr = t_va > ULIM ? t_va : ULIM;
		return -E_FAULT;
	}
	perm |= PTE_P;
	uintptr_t va_end = ROUNDUP(t_va+len,PGSIZE);
	uintptr_t va_start = ROUNDDOWN(t_va, PGSIZE);

	for(; va_start < va_end; va_start += PGSIZE) {
		pte_t* pte = pgdir_walk(env->env_pgdir, (void*)va_start, 0);
		if(!pte || (*pte & perm) != perm) {
			user_mem_check_addr = va_start == ROUNDDOWN(t_va, PGSIZE) ? t_va : va_start;
			return -E_FAULT;
		} 
	}
	return 0;
}
```

Change `kern/syscall.c` to sanity check arguments to system calls.

```c
    case(SYS_cputs):
        user_mem_assert(curenv,(void*)a1, a2,PTE_U);
        sys_cputs((void*) a1, a2);
        return 0;
			
			
    static void
    sys_cputs(const char *s, size_t len)
    {
        // Check that the user has permission to read memory [s, s+len).
        // Destroy the environment if not.

        // LAB 3: Your code here.
        user_mem_check(curenv,(void *)s, len,PTE_U);
        // Print the string supplied by the user.
        cprintf("%.*s", len, s);
    }

```

最后，将kern/kdebug.c中的debuginfo_eip更改为在usd、stabs和stabstr上调用用户_mem_check。

```c
// Make sure this memory is valid.
		// Return -1 if it is not.  Hint: Call user_mem_check.
		// LAB 3: Your code here.
		if(user_mem_check(curenv, (const void*)usd, sizeof(struct UserStabData), PTE_U))
			return -1;
		stabs = usd->stabs;
		stab_end = usd->stab_end;
		stabstr = usd->stabstr;
		stabstr_end = usd->stabstr_end;

		// Make sure the STABS and string table memory is valid.
		// LAB 3: Your code here.
		if(user_mem_check(curenv, (const void*)stabs, (uintptr_t)stab_end - (uintptr_t)stabs, PTE_U))
			return -1;
		if(user_mem_check(curenv, (const void*)stabstr, (uintptr_t)stabstr_end - (uintptr_t)stabstr, PTE_U))
			return -1;
```

运行user/breakpoint，内核监视器运行回溯跟踪，并在内核因页面错误而崩溃之前看到回溯遍历到lib/libmain.c。是什么导致此页面错误？你不需要修复它，但你应该理解它为什么会发生。

A. umain()函数的参数只要2个0，参数的地址再往上就是Empty Memory，内核和用户程序都没有访问权，且页面不存在，所以page fault occurs。

```c
K> mon_traceback
ebp efffff00  eip f0100aac  args 00000001 efffff28 f01d2000 f01068e6 00001000
kern/monitor.c:26: monitor+339
ebp efffff80  eip f0104350  args f01d2000 efffffbc f03bc000 00000092 eebfd000
kern/trap.c:26: trap+312
ebp efffffb0  eip f010441f  args efffffbc 00000000 00000000 eebfdfc0 efffffdc
kern/syscall.c:7: syscall+0
ebp eebfdfc0  eip 800087  args 00000000 00000000 eebfdff0 00800058 00000000
lib/libmain.c:9: libmain+78
ebp eebfdff0  eip 800031  args 00000000 00000000Incoming TRAP frame at 0xeffffe64
kernel panic at kern/trap.c:264: kernel occurs page falut! va eebfe000 ip f01008d9

Welcome to the JOS kernel monitor!
```

