---
title: BUAA-OSLab3实验总结
catalog: true
date: 2025-04-21 15:29:13
subtitle: 人生本来就是多进程的，你做好管理了吗？
sticky: 10
lang: en
header-img: /img/header_img/ghost_blade5.jpg
tags:
- OS
categories:
- OS
---

# BUAA-OSLab3实验总结——进程管理

> 固执不是傻，高歌到沙哑。

## 前言

​	这几天下了**一阵阵小雨**，風船说湿润了一些，很喜欢下雨天。

​	我倒是感觉很**困**😴，人怎么能困成这样😭。索性让自己睡饱了，半下午来到图书馆开始学一下Lab3🤔。

​	那为什么这次的博客从周一开始写呢🙋‍？

​	因为笔者这次lab3实验想做一次记录性博客，从一个初学者的视角去展示这段旅程🖊，而不是学完之后的回顾总结。那事不宜迟，GoGoGo！出发咯！😀

## 征途中的迷思与Thinking

### 🧐迷思一 模板页目录(base_pgdir)

​	请看相关**注释**：

```c
	/*
	 * We want to map 'UPAGES' and 'UENVS' to *every* user space with PTE_G permission (without PTE_D), then user programs can read (but cannot write) kernel data structures 'pages' and'envs'.
	 * Here we first map them into the *template* page directory 'base_pgdir'.
	 * Later in 'env_setup_vm', we will copy them into each 'env_pgdir'.
	*/
```

#### 1 先问是什么？

- base_pgdir 本质上是一个**页目录 (Page Directory)**。就是我们多级页表结构中的**一级页表**。
- 它被称为“模板”（Template）是因为它**不直接**被任何一个正在运行的用户进程（Env）使用。相反，它包含了一些**所有用户进程都应该拥有的通用内存映射**。
- 可以把它想象成一个预先配置好的蓝图或模板，用于快速创建新进程的地址空间。

#### 2 再问为什么（这样设计）？

- **高效率:** 当创建一个新的进程时，需要为这个进程设置它自己的虚拟地址空间，也就是要创建并填充它自己的页目录 (env_pgdir)。有很多内存区域的映射对于所有用户进程来说都是相同的，如果每次创建新进程时，都从头开始、逐一添加这些通用映射，会非常低效。使用模板页目录，可以**在初始化时就把这些通用映射设置好**。之后创建新进程时，只需将 base_pgdir 的内容**复制**到新进程的 env_pgdir 中，然后再添加该进程特有的映射（如代码段、数据段、栈等）。这大大**加快了进程虚拟内存初始化的速度**。
- **共享内核数据:** 如**注释（代码段）**中所述，操作系统设计者希望让用户程序能够**读取（但不能写入）**某些内核维护的数据结构，比如 pages 数组（描述所有物理页的状态）和 envs 数组（描述所有进程控制块的状态）。这对于实现某些用户态的功能（如用户级调试器、性能监控工具或某些IPC机制）可能很有用。将这些数据映射到*每个*用户进程的地址空间中，并且使用相同的虚拟地址（UPAGES, UENVS），可以简化用户程序的编写。

#### 3 为这段代码逐行注释

```c
	// 分配一个物理页框p
	struct Page *p;
	panic_on(page_alloc(&p));
	p->pp_ref++;
	
	base_pgdir = (Pde *)page2kva(p); // 转化为内核虚拟地址
	// map_segment : 用于在 base_pgdir 中建立一段虚拟内存到物理内存的映射。
	map_segment(base_pgdir, 0, PADDR(pages), UPAGES,
		    ROUND(npage * sizeof(struct Page), PAGE_SIZE), PTE_G);
	map_segment(base_pgdir, 0, PADDR(envs), UENVS, ROUND(NENV * sizeof(struct Env), 		PAGE_SIZE),PTE_G);
```

### 🧐迷思二 env_setup_vm 函数

​	先看看源码（补全版✔）：

```c
static int env_setup_vm(struct Env *e) {
	/* Step 1:
	 *   Allocate a page for the page directory with 'page_alloc'.
	 *   Increase its 'pp_ref' and assign its kernel address to 'e->env_pgdir'.
	 *
	 * Hint:
	 *   You can get the kernel address of a specified physical page using 'page2kva'.
	 */
	struct Page *p;
	try(page_alloc(&p));
	/* Exercise 3.3: Your code here. */
	p->pp_ref++;
 	e->env_pgdir = (Pde *)page2kva(p);
	/* Step 2: Copy the template page directory 'base_pgdir' to 'e->env_pgdir'. */
	/* Hint:
	 *   As a result, the address space of all envs is identical in [UTOP, UVPT).
	 *   See include/mmu.h for layout.
	 */
	memcpy(e->env_pgdir + PDX(UTOP), base_pgdir + PDX(UTOP),
	       sizeof(Pde) * (PDX(UVPT) - PDX(UTOP)));

	/* Step 3: Map its own page table at 'UVPT' with readonly permission.
	 * As a result, user programs can read its page table through 'UVPT' */
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_V;
	return 0;
}
```

​	首先，`Step1`在注释的帮助下很好理解：为进程e分配一个物理页框`p`并以之作为页目录，并且用`e->env_pgdir`记录内核访问该页目录`p`的虚拟地址。

​	`Step2`的注释说 *“Copy the template page directory”*，那我们知道了，要用**模板页目录** “批量生产” 页目录了！

​	但是随之而来的是看不懂的`memcpy`括号里的各路参数🫠，什么`UTOP`哇什么`UVPT`哇...

​	你在OS中高声呼救，回应你的只有【**Hint**】...（好叭其实是我眼瞎）😓

​	那我们CV一段`layout`看看：

```c
o      ULIM     -----> +----------------------------+------------0x8000 0000-------
o                      |         User VPT           |     PDMAP                /|\
o      UVPT     -----> +----------------------------+------------0x7fc0 0000    |
o                      |           pages            |     PDMAP                 |
o      UPAGES   -----> +----------------------------+------------0x7f80 0000    |
o                      |           envs             |     PDMAP                 |
o  UTOP,UENVS   -----> +----------------------------+------------0x7f40 0000    |
```

​	原来这是一段虚拟地址😌。

​	这个虚拟地址范围本身**并不直接存储**任何数据，它只是一个**地址空间区域**。关键在于，在 `env_setup_vm` 中，通过 `memcpy` 从 `base_pgdir` 复制过来的**页目录项 (PDEs)**，为这个范围内的**特定虚拟地址**（如 UPAGES, UENVS）**建立了映射**。这个映射关系大概就是以下两种：（其中用户地址都是虚拟地址）

- **进程 e 的用户地址 UPAGES** <----(通过 e->env_pgdir 中复制来的 PDE)----> **物理地址 PADDR(pages)** (只读)

- **进程 e 的用户地址 UENVS** <----(通过 e->env_pgdir 中复制来的 PDE)----> **物理地址 PADDR(envs)** (只读)

​	这就是我们在上一个迷思提到的**高效性**。以`base_pgdir`为模板通过`memcpy`快速初始化，而不是逐一调用`map_segment`函数创建映射。

​	`Step3`和思考题撞了，请直接看下面`Thinking3.1`！

### 🤔Thinking3.1

> 请结合 MOS 中的页目录自映射应用解释代码中 e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_V 的含义。 

​	让我们时刻记得**自映射**的目标：**让进程能通过虚拟地址 UVPT 这个“窗口”来访问它自己的页目录和页表**。

1. `e->env_pgdir`：进程`e`的页目录数组指针，其中包含了1024个`PDE`，每个`PDE`负责管理4MB的虚拟地址空间。

2. `PDX(UVPT)`：管理`UVPT`所在的4MB虚拟地址空间的那个`PDE`的索引。

3. `PADDR(e->env_pgdir)`：页目录内核虚拟地址对应的物理地址，也就是找到存储`e`的页目录那4KB的物理页框的真实物理地址。

4. `| PTE_V`：与页目录的物理地址合并一起，置有效位。

   那自映射的实现最后是怎样的？我们在进程`e` 的页目录中创建了一个**“精妙”的入口**🚪：

   - **入口的位置:** 由虚拟地址 UVPT 决定的那个 PDE（索引为 `PDX(UVPT)`）。
   - **入口指向哪里:** 指向页目录**自己**所在的物理地址 (`PADDR(e->env_pgdir)`)。
   - **入口的状态:** 是有效的 (`PTE_V`)。

   那自映射的效果是什么？我认为自映射就是让第一级页目录在`UVPT`这个用户态的虚拟地址下变得“可见”，允许用户直接读取其中的**PDEs**。

### 🤔Thinking3.2

>  elf_load_seg 以函数指针的形式，接受外部自定义的回调函数 map_page。 请你找到与之相关的 data 这一参数在此处的来源，并思考它的作用。没有这个参数可不可以？为什么？ 

​	先看`elf_load_seg`的声明：

```c
int elf_load_seg(Elf32_Phdr *ph, const void *bin, elf_mapper_t map_page, void *data);
```

​	指导书指出：`elf_load_seg` 负责将 ELF 文件的一个 segment 加载到内存。为了实现这一功能，需要使用一个回调函数`map_page`，并将额外参数`data`传给它。当`elf_load_seg`解析到一个需要被加载到内存中的页面，就会由这个`map_page`来完成此页面的加载过程。那这里指导书的言外之意已经呼之欲出了，`data`作为传入`map_page`的参数一定是**协助页面加载**的。

​	另外指导书也指出`load_icode`对`elf_load_seg`的调用关系，我们决定层层深入，先看`load_icode`，找到了调用的语句：

```c
panic_on(elf_load_seg(ph, binary + ph->p_offset, load_icode_mapper, e));
```

​	😮我们发现`data`在这里的赋值就是PCB的指针`e`，而正如指导书所述，`load_icode_mapper` 就是 `map_page` 的具体实现。我们再看`elf_load_seg`的实现，发现`data`出现且仅出现在map_page的参数中。那我们可以直接进入自定义的回调函数`load_icode_mapper`探寻`data`的作用了。

​	`load_icode_mapper`部分代码：

```c
static int load_icode_mapper(void *data, u_long va, size_t offset, u_int perm, const void *src, size_t len) {
            /*	data:通用上下文指针，这里就是进程控制块的指针 e
                va:需要进行映射的目标虚拟地址
                offset:当前要处理的数据块 (由 src 指向，长度为 len) 在整个ELF程序段内的字节偏移量
                perm:为虚拟地址 va 映射到的物理页面所要设置的访问权限
                src:指向源数据的指针。这些数据位于内存中加载的原始 ELF 文件二进制内容中
                len:需要从 src 拷贝到新分配物理页面的数据字节数
            */ 
	struct Env *env = (struct Env *)data;
	// some definition...
	/* Step 1: Allocate a page with 'page_alloc'. */
	/* Step 2: If 'src' is not NULL, copy the 'len' bytes started at 'src' into 'offset' at this * page. */
	/* Step 3: Insert 'p' into 'env->env_pgdir' at 'va' with 'perm'. */
	return page_insert(env->env_pgdir, env->env_asid, p, va, perm);
}
```

​	可以看到我们在用`page_insert`建立虚拟地址与物理地址的映射的时候，用到了当前进程空间的页目录基地址(`pgdir`)和`asid`，倘若没有上下文指针`data`，自然就找不到这个PCB的这两个参数了。因此没有这两个参数是万万不行的。

### 🤔Thinking3.3

> 结合 elf_load_seg 的参数和实现，考虑该函数需要处理哪些页面加载的情况。

```c
	// 段起始于页面中
	if (offset != 0) {  
		if ((r = map_page(data, va, offset, perm, bin, 
            MIN(bin_size, PAGE_SIZE - offset))) != 0) {
			/*
				MIN(bin_size, PAGE_SIZE - offset): 需要从 bin 拷贝的数据量
				根据剩余的 bin数据量判断是：
				1. 填满页剩余的部分
				2. 将极小的 bin中所有的数据量全部拷贝进当前页
			*/
			return r; // 建立映射失败，返回错误码
		}
	}

	/* Step 1: load all content of bin into memory. */
	// i作为拷贝起点，根据前段代码中是否存在offset导致先拷贝一部分数据
	// i += PAGE_SIZE即按页复制，直到剩余数据不足以填满一整页
	for (i = offset ? MIN(bin_size, PAGE_SIZE - offset) : 0; i < bin_size; i += PAGE_SIZE) {
		if ((r = map_page(data, va + i, 0, perm, bin + i, MIN(bin_size - i, PAGE_SIZE))) != 0) 	{
			return r;
		}
	}

	/* Step 2: alloc pages to reach `sgsize` when `bin_size` < `sgsize`. */
	// i < sgsize说明还有内存空间需要分配并清零
	while (i < sgsize) {
		// 用 NULL告诉`map_page`只需要分配全0的物理页面即可，无需拷贝数据
		if ((r = map_page(data, va + i, 0, perm, NULL, MIN(sgsize - i, PAGE_SIZE))) != 0) 		{
			return r;
		}
		i += PAGE_SIZE;
	}
	return 0;
```

- `Step0`：函数判断va是否页对齐，如果不对齐，需要将多余的地址记为offset，并且先拷贝一部分的数据；
- `Step1`：按页将段内的数据映射到物理空间；
- `Step2`：若发现其在内存的大小大于在文件中的大小，则需要把多余的空间用0填满。

### 🤔Thinking3.4

> 这里的 env_tf.cp0_epc 字段指示了进程恢复运行时 PC 应恢复到的位置。我们要运行的 进程的代码段预先被载入到了内存中，且程序入口为 e_entry，当我们运行进程时，CPU 将自动从 PC 所指的位置开始执行二进制码。 
>
> 思考上面这一段话，并根据自己在 Lab2 中的理解，回答： 你认为这里的 env_tf.cp0_epc 存储的是物理地址还是虚拟地址?

​	**虚拟地址**；`cp0_epc`中存储的是发生错误时CPU所处的指令地址，CPU所见的均为虚拟地址。

### 🤔Thinking3.5

>  试找出 0、1、2、3 号异常处理函数的具体实现位置。8 号异常（系统调用） 涉及的 do_syscall() 函数将在 Lab4 中实现。

​	既然异常处理函数要和CPU的协处理器打交道，那大概率是用汇编写的`.S`文件啦😊。我们在`genex.S`文件中找到了`handle_int`。

​	但是没找到其余的异常处理？定睛一看：

```assembly
.macro BUILD_HANDLER exception handler
NESTED(handle_\exception, TF_SIZE + 8, zero)
	move    a0, sp
	addiu   sp, sp, -8
	jal     \handler
	addiu   sp, sp, 8
	j       ret_from_exception
END(handle_\exception)
.endm

BUILD_HANDLER tlb do_tlb_refill

#if !defined(LAB) || LAB >= 4
BUILD_HANDLER mod do_tlb_mod
BUILD_HANDLER sys do_syscall
#endif

BUILD_HANDLER reserved do_reserved
```

​	原来如此，这个宏定义竟然定义了一段**汇编入口包装器**，但他们终究会通过`jal     \handler`进行跳转，而这段代码怎么会叫`handle_mod`，`handle_sys`和`handle_tlb`三种名字呢？原来代码段被命名为`handle_\exception`了。

​	那这里就类似于传参了，例如下方的`BUILD_HANDLER tlb do_tlb_refill`，这个异常处理函数自然就叫做`handle_tlb`了，再看第二个参数作为`jal`的对象——`do_tlb_refill`，那是真心熟悉呀🤗，这不是我们在Lab2在tlbex.C写的**快表重填函数**嘛，不出所料`do_tlb_mod`也在其中。

### 🤔Thinking3.6

>  阅读 entry.S、genex.S 和 env_asm.S 这几个文件，并尝试说出时钟中断在哪些时候开启，在哪些时候关闭。 

​	在`entry.S`中，发生异常后CPU会跳转到函数`exc_gen_entry`，部分代码与注释如下：

```assembly
	mfc0    t0, CP0_STATUS  // CP0_STATUS 的内容读取到通用寄存器 t0 中
	and     t0, t0, ~(STATUS_UM | STATUS_EXL | STATUS_IE) // 将 t0 中的这三个位强制清零。
	mtc0    t0, CP0_STATUS // 通用寄存器 t0 中修改后的值写回到 CP0_STATUS 寄存器中
```

​	此前，`STATUS_EXL`因为异常被置1， 修改`CP0_STATUS`后`STATUS_IE`被置0。如此以来，自此到异常处理函数阶段，中断一直保持**禁用状态**。

​	无论是什么异常处理函数，最后应当都跳转到`ret_from_exception`函数结束异常处理。

​	事实真是如此吗？笔者将在稍后的**🧐迷思三**展开详述，先看`ret_from_exception`代码段：

```assembly
FEXPORT(ret_from_exception)
	RESTORE_ALL // 恢复至SAVE_ALL之前的值，包括恢复CP0_STATUS寄存器到它在异常发生之前的值
	eret // 将 CP0_EPC 的值恢复到 PC 且 EXL 会被自动设置为 0 
```

​	此时，因为`EXL`已然置0，那是否开启中断完全看`SAVE_ALL`执行前的`IE`位是否为1，如若是1，中断将变为**开启状态**，反之则保持**关闭状态**。此时应该是**唯一可能开启中断的时机**，其余时间都保持**中断关闭**。

​	这个时候有的朋友就要说了，主播主播🙋‍🤓，`handle_int`根本就没有执行`ret_from_exception`！它明明跳转到了`Schedule`！

​	**还真是，但不完全是🤭。**欲知后事，请看下回。***（广告位招租）***

### 🧐迷思三 `handle_int`到底怎么结束的？

​	看是跳转到了`Schedule`函数，我们去看看——发现是*Exercise 3.12*😓

​	但是我们根据指导书关于**进程调度**的介绍，得知这个函数的作用便是用时间片轮转的算法调度进程，最后**运行进程**。因此最后的语句一定是：`env_run(e)`，其中`e`是我们调度的进程控制块指针。

​	那我们再回顾一下`env_run()`，看到最后的`env_pop_tf`，不妨结合注释看一下它的用途：

```assembly
LEAF(env_pop_tf)
.set reorder
.set at
	mtc0    a1, CP0_ENTRYHI // 设置 ASID
	move    sp, a0 // 将传入的 Trap Frame 地址 (a0) 设置为当前的栈指针 (sp)
	RESET_KCLOCK
	j       ret_from_exception // 兜兜转转又回到了起点
END(env_pop_tf)
```

​	因此我们的推测完全正确，`handle_int`也和其他处理异常函数一样回到了`ret_from_exception`。

### 🤔Thinking3.7

>  阅读相关代码，思考操作系统是怎么根据时钟中断切换进程的。

​	根据指导书描述：

> “一旦时钟中断产生，就会触发 4KC 硬件的异常中断处理流程。系统将 PC 指向 0x80000180， 跳转到.text.exc_gen_entry 代码段执行。对于时钟引起的中断，通过.text.exc_gen_entry 代码段的分发，最终会调用 handle_int 函数进行处理。”

​	值得一提的是，它比我想象中更像`entry`😮。为什么这么说呢？我觉得它可以称作**COP7与OSLab3的接口，软硬件的交界处**。并不需要什么其他函数去调用唤醒，而是和PC值息息相关。这一刻，感觉唤醒了2024年的记忆，思维在神经元不断链接中形成了闭环。

​	此后的内容便如我们在**迷思三**中的逻辑所展示，函数之间调用关系明确，总结大致如下：

​	`exc_gen_entry`-->`exception_handlers(handle_int)`-->`schedule`

​	-->`env_run`-->`env_pop_tf`-->`ret_from_exception`

​	最后在`ret_from_exception`中，时钟中断得到了恢复。

## 后记

​	又到啦最爱的后记环节，感觉自己用心梳理了一天的OSLab3还是蛮有成就感的！😊

​	其实你很难评判一件事情做的是否值得，甚至很难评判它是否有意义。但是我愿意告诉大家我认为它真的很有意义，又是一场伟大的唯心主义的胜利✌。因为我确实比之前总结来说更具自己的思考了，想的东西也很多，和AI坐下来一聊就是一天。给大家截几张和AI的聊天截屏吧📸。

![我是神人](我是神人.png)

![我是睡觉大王](我是睡觉大王.png)

![我与葛米妮](我与葛米妮.png)

​	现在是晚上`22:17`，小累小累😌。快要放假了！还有不少事情说是，但是上周都活过来了，这日子是越来越有盼头了😅。

​	其实想想也是很好笑，人们总说，xx岁是人生最重要的一年。

​	倒也还好，我感觉毕竟人生百八十年呢，每一年都同等重要吧，**没有哪个五年十年能决定我未来走向，也没有哪个一年半载能熄灭我不灭心火**。

​	很喜欢在这里倾吐一点自己的想法，毕竟是**a private corner**，让人有种**如释重负**的感觉😌。

​	感谢你看到这里，那我们就要说再见了！亲友们不要再给我打赏了，不然我要撤二维码了哈哈哈😊

​	言尽于此，回寝歇息。

> 别因为短暂的低谷就失去对自己变强的信心，别灰心，万事才刚刚开始啊
