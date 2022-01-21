# The Adventures of OS

# [**The Adventures of OS**](https://gitee.com/link?target=https%3A%2F%2Fgallium70.github.io%2Fosblog-translation%2Fch8.html%23the-adventures-of-os)

使用Rust的RISC-V操作系统

 [**在Patreon上支持我!**](https://gitee.com/link?target=https%3A%2F%2Fwww.patreon.com%2Fsgmarz)  [**操作系统博客**](https://gitee.com/link?target=http%3A%2F%2Fosblog.stephenmarz.com%2F)  [**RSS订阅** ](https://gitee.com/link?target=http%3A%2F%2Fosblog.stephenmarz.com%2Ffeed.rss)  [**Github** ](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Fsgmarz)  [**EECS网站**](https://gitee.com/link?target=http%3A%2F%2Fweb.eecs.utk.edu%2F~smarz1) 

这是[用Rust编写RISC-V操作系统](https://gitee.com/link?target=http%3A%2F%2Fosblog.stephenmarz.com%2Findex.html)系列教程中的第11章。

[目录](http://osblog.stephenmarz.com/index.html) → [第10章](http://osblog.stephenmarz.com/ch10.html) → (第11章)

# 用户空间进程

2020年6月1日：仅限PATREON

2020年6月8日：公开

## 资源和参考资料

ELF标准可以在这里找到： [ELF File Format (PDF)](http://osblog.stephenmarz.com/files/elf.pdf).

## 简介

这就是我们一直在等待的时刻。十个章节的设置使我们来到了这一时刻--最终能够从磁盘上加载一个进程并运行它。可执行文件的文件格式被称为ELF（可执行和可链接格式）。我将对它进行一些详细介绍，但你可以通过这一种文件类型探索很多途径。

## ELF文件格式

可执行和可链接格式（ELF）是一种广泛使用的文件格式。 如果你使用过Linux，你无疑见过它或它的效果。 这种文件格式包含一个ELF头，后面是程序头。 每一次，我们都在告诉操作系统链接器将可执行部分映射到了哪里。如果你不记得了，我们有一个用于CPU指令的.text部分，用于全局常量的.rodata，用于全局初始化变量的.data，以及用于全局未初始化变量的.bss部分。在ELF格式中，编译器会决定把这些放在哪里。 另外，由于我们使用的是虚拟内存地址，ELF头指定了*入口点*，这就是我们在第一次调度进程时要放在程序计数器中的内容。

[![img](http://osblog.stephenmarz.com/imgs/elf_format.png)](http://osblog.stephenmarz.com/imgs/elf_format.png)

[joke]我的笔迹从4岁起就没有改进过[/joke]

让我们看看一些能帮助我们的Rust结构。这些都在elf.rs中。

```rust
#[repr(C)]
pub struct Header {
    pub magic: u32,
    pub bitsize: u8,
    pub endian: u8,
    pub ident_abi_version: u8,
    pub target_platform: u8,
    pub abi_version: u8,
    pub padding: [u8; 7],
    pub obj_type: u16,
    pub machine: u16, // 0xf3 for RISC-V
    pub version: u32,
    pub entry_addr: usize,
    pub phoff: usize,
    pub shoff: usize,
    pub flags: u32,
    pub ehsize: u16,
    pub phentsize: u16,
    pub phnum: u16,
    pub shentsize: u16,
    pub shnum: u16,
    pub shstrndx: u16,
}
```

所有ELF文件都以这个ELF头结构开始。最上面是0x7f，后面是大写的ELF，即0x7f 0x45 0x4c和0x46，你可以在下面看到。我对ls（列表）命令进行了简单的十六进制转储。你可以看到神奇的东西就在那里。

[![img](http://osblog.stephenmarz.com/imgs/ls_elf.png)](http://osblog.stephenmarz.com/imgs/ls_elf.png)

其余的字段将告诉我们这个ELF文件是为哪个架构制作的。RISC-V已经保留了0xf3作为其机器类型。所以，当我们从磁盘上加载ELF文件时，我们必须确保它是为正确的体系结构制作的。你也会注意到`entry_addr`也在上面。这是一个虚拟内存地址，是`_start`的开始。我们的_start只是简单的调用main，然后当main返回时，它调用exit系统调用。这就是大多数程序的实际工作方式，但它们的工作要严格得多，包括获取命令行参数等等。现在，我们还没有这些。

我们需要知道的字段是`phoff`字段，它指定了程序头的偏移。程序头是一个或多个程序部分的表格。我拍了一张ls（再次）和程序头的快照。你可以用`readelf -l /bin/ls`做同样的事情。下面的代码显示了我是如何用Rust读取ELF头的。

```rust
let elf_hdr;
unsafe {
  elf_hdr = (buffer.get() as *const elf::Header).as_ref().unwrap();
}
if elf_hdr.magic != elf::MAGIC {
  println!("ELF magic didn't match.");
  return;
}
if elf_hdr.machine != elf::MACHINE_RISCV {
  println!("ELF loaded is not RISC-V.");
  return;
}
if elf_hdr.obj_type != elf::TYPE_EXEC {
  println!("ELF is not an executable.");
  return;
}
```

你可以看到，现在我们到了程序头，我对/bin/ls进行了快照。[![img](http://osblog.stephenmarz.com/imgs/ls_program_headers.png)](http://osblog.stephenmarz.com/imgs/ls_program_headers.png)

程序头在Rust中具有以下结构。

```rust
#[repr(C)]
pub struct ProgramHeader {
    pub seg_type: u32,
    pub flags: u32,
    pub off: usize,
    pub vaddr: usize,
    pub paddr: usize,
    pub filesz: usize,
    pub memsz: usize,
    pub align: usize,
}
```

/bin/ls使用共享库，但我们还没那么厉害。所以，我们关心的唯一程序头是由LOAD显示的那些。这些是我们需要为我们的静态二进制文件加载到内存中的部分。 在ProgramHeader结构中，我们需要seg_type为LOAD。标志位告诉我们如何保护虚拟内存。有三个标志EXECUTE（1），WRITE（2）和READ（4）。我们还需要off（偏移量），它告诉我们在ELF文件中要加载到程序内存的部分包含在哪里。最后，vaddr是我们需要指向MMU的地方，即我们将这部分加载到内存中的地方。你可以在test.rs的test_elf()函数中看到我是怎么做的。

```rust
for i in 0..elf_hdr.phnum as usize {
  let ph = ph_tab.add(i).as_ref().unwrap();
  if ph.seg_type != elf::PH_SEG_TYPE_LOAD {
    continue;
  }
  if ph.memsz == 0 {
    continue;
  }
  memcpy(program_mem.add(ph.off), buffer.get().add(ph.off), ph.memsz);
  let mut bits = EntryBits::User.val();
  if ph.flags & elf::PROG_EXECUTE != 0 {
    bits |= EntryBits::Execute.val();
  }
  if ph.flags & elf::PROG_READ != 0 {
    bits |= EntryBits::Read.val();
  }
  if ph.flags & elf::PROG_WRITE != 0 {
    bits |= EntryBits::Write.val();
  }
  let pages = (ph.memsz + PAGE_SIZE) / PAGE_SIZE;
  for i in 0..pages {
    let vaddr = ph.vaddr + i * PAGE_SIZE;
    let paddr = program_mem as usize + ph.off + i * PAGE_SIZE;
    map(table, vaddr, paddr, bits, 0);
  }
}
```

我在上面的代码中所做的就是枚举所有的程序头文件。 ELF头文件通过phnum字段告诉我们有多少个头文件。然后我们检查段的类型，看它是否是LOAD。如果不是，我们就跳过它。 然后，我们检查该段是否真的包含任何东西。如果没有，那么加载它就没有用了。最后，我们把从文件系统中读到的东西（缓冲区）复制到进程的内存（program_mem）中。由于这些是虚拟内存地址，代码的其余部分决定了我们应该如何映射这些页面。

## 运行进程

我们需要映射一些东西，包括堆栈和程序。另外，别忘了将程序计数器设置为entry_addr!

```rust
(*my_proc.frame).pc = elf_hdr.entry_addr;
(*my_proc.frame).regs[2] = STACK_ADDR as usize + STACK_PAGES * PAGE_SIZE;
(*my_proc.frame).mode = CpuMode::User as usize;
(*my_proc.frame).pid = my_proc.pid as usize;
(*my_proc.frame).satp = build_satp(SatpMode::Sv39, my_proc.pid as usize, my_proc.root as usize);
```

在这里，regs[2]是堆栈指针（SP），它必须是有效的并被映射，否则进程将立即出现页面故障。现在一切都准备好了，我们的最后一点执行工作是将其添加到进程列表中。当调度器开始工作时，它将运行我们新造的进程。

```rust
if let Some(mut pl) = unsafe { PROCESS_LIST.take() } {
  println!(
            "Added user process to the scheduler...get ready \
            for take-off!"
  );
  pl.push_back(my_proc);
  unsafe {
    PROCESS_LIST.replace(pl);
  }
}
else {
  println!("Unable to spawn process.");
}
```

## 编写用户空间程序

我们还没有一个C语言库。然而，我正在使操作系统访问newlib，它是一个主要用于嵌入式系统的小型C语言库。目前，我做了一个叫做`startlib`的小库，它将使我们开始第一步，我把printf复制到了里面。

```rust
.section .text.init
.global _start
_start:
  call	main
  li	a0, 93
  j 	make_syscall
```

_start是一个特殊的标签，编译器将用它作为入口地址。回顾一下，当我们建立一个新的进程时，我们在程序计数器中设置了这个地址。在main返回后，我们安排了一个编号为93的系统调用，也就是 "exit "的系统调用。这个系统调用所做的就是取消进程的调度，释放其所有的资源。

还有其他一些实用程序，包括我们的小库中的printf，但是还是让我们做一个简单的程序，看看我们是否能让它工作。为了更加稳健，我将扩展我们所有的可用部分，看看它们是否能正常加载。

```cpp
#include <printf.h>

const int SIZE = 1000;
int myarray[SIZE];
int another_array[5] = {1, 2, 3, 4, 5};

int main()
{
  printf("I'm a C++ program, and I'm running in user space. How about a big, Hello World\n");
  printf("My array is at 0x%p\n", myarray);
  printf("I'm going to start crunching some numbers, so gimme a minute.\n");
  for (int i = 0;i < SIZE;i++) {
    myarray[i] = another_array[i % 5];
  }
  for (int i = 0;i < 100000000;i++) {
    myarray[i % SIZE] += 1;
  }
  printf("Ok, I'm done crunching. Wanna see myarray[0]? It's %d\n", myarray[0]);
  return 0;
}
```

这个程序其实并没有做什么有用的事情，但是它可以看到系统调用是否工作以及上下文切换。在QEMU上，这个程序在我家里的机器上运行大约需要5到8秒。

然后我们用我们的C++工具链（如果你有的话）来编译这个。`riscv64-unknown-elf-g++ -Wall -O0 -ffreestanding -nostartfiles -nostdlib -static -march=rv64g -mabi=lp64d -I. /startlib -L. /startlib -o helloworld.elf helloworld.cpp -lstart`。

如果你没有工具链，你可以在这里下载我的程序：[helloworld.elf](http://osblog.stephenmarz.com/helloworld.elf). 这要求你的系统调用与我的相同，因为它是按系统调用号进行的。

## 上传程序

我们可以使用Linux来上传我们的elf文件。

[![img](http://osblog.stephenmarz.com/imgs/upload_hw.png)](http://osblog.stephenmarz.com/imgs/upload_hw.png)

请注意节点号（26）和文件大小（14776）。你的可能不一样，所以你可能要设置它。修改test.rs，把你的inode和文件大小放在顶部。

```rust
let files_inode = 26u32; // Change to yours!
let files_size = 14776; // Change to yours!
let bytes_to_read = 1024 * 50;
let mut buffer = BlockBuffer::new(bytes_to_read);
let bytes_read = syscall_fs_read(
                                  8,
                                  files_inode,
                                  buffer.get_mut(),
                                  bytes_to_read as u32,
                                  0,
);
if bytes_read != files_size {
  println!(
            "Unable to load program at inode {}, which should \
            be {} bytes, got {}",
            files_inode, files_size, bytes_read
  );
  return;
}
```

这将使用我们的文件系统读取调用，将给定的inode读入内存。然后，我们仔细检查大小是否与stat所说的完全一致。然后我们开始ELF加载过程，正如我之前讨论的那样。

从这里开始，当你`cargo运行`你的操作系统时，你应该看到以下内容。

[![img](http://osblog.stephenmarz.com/imgs/user_prog_screenshot.png)](http://osblog.stephenmarz.com/imgs/user_prog_screenshot.png)

让我们再看看是什么在做这件事：

```cpp
#include <printf.h>

const int SIZE = 1000;
int myarray[SIZE];
int another_array[5] = {1, 2, 3, 4, 5};

int main()
{
  printf("I'm a C++ program, and I'm running in user space. How about a big, Hello World\n");
  printf("My array is at 0x%p\n", myarray);
  printf("I'm going to start crunching some numbers, so gimme a minute.\n");
  for (int i = 0;i < SIZE;i++) {
    myarray[i] = another_array[i % 5];
  }
  for (int i = 0;i < 100000000;i++) {
    myarray[i % SIZE] += 1;
  }
  printf("Ok, I'm done crunching. Wanna see myarray[0]? It's %d\n", myarray[0]);
  return 0;
}
```

好吧，这难道不是一件好事吗？这就是我看到的打印到屏幕上的东西!  请注意，myarray[0]得到了100001。在开始时，我们把数值1放入myarray[0]，然后每1000个100000000，我们再把一个数值放入myarray[0]，总共是100001。所以，是的，我们的部分似乎是在正常运作

## 验证

让我们检查一下我们的helloworld.elf文件，看看我们在Rust中做了什么。让我们首先检查ELF头。你会看到我们的入口点0x101e4就是我们输入PC的内容（你可以println！this out来验证）。

[![img](http://osblog.stephenmarz.com/imgs/helloworld_elf_header.png)](http://osblog.stephenmarz.com/imgs/helloworld_elf_header.png)接下来是程序头文件。这导致我在第一次检查程序头时写出了几个bug。注意到.text、.rodata和.eh_frame被放在第一个头文件中，有读取和执行的权限。还注意到0x303c和0x3040（PH0的结束和PH1的开始）重叠了物理页（但不是虚拟页）。我不喜欢这样。.rodata不应该是可执行的，但它是可执行的，因为它和.text部分在同一个物理和虚拟页面。

[![img](http://osblog.stephenmarz.com/imgs/helloworld_prog_headers.png)](http://osblog.stephenmarz.com/imgs/helloworld_prog_headers.png)

看看这个。我们的ELF头显示的正是我们在操作系统中验证的内容，程序头也是差不多的。在Linux中，唯一真正重要的文件类型是ELF。其他一切都由它需要的任何帮助程序来决定。

## 结论

欢迎来到这个操作系统教程的结尾。为了完善你的操作系统，你将需要增加read、write、readdir等的系统调用。我把libgloss的列表放在syscall.rs中（在底部），这样你就知道libgloss包含哪些系统调用。最终的测试是将gcc编译到你的文件系统中并从那里执行它。另外，我们需要有一个遍历函数来按名字遍历目录结构。目前，我们是通过节点号来定位的，这不是很好。然而，你可以看到一个操作系统的所有有用的部分。

本教程到此结束，但我将在我们的操作系统中加入一些东西。我计划加入图形和网络，但这扩大了本教程的范围，所以我将把它们放在自己的博客文章中。

恭喜你，你现在有一个可以运行进程的操作系统了 像往常一样，你可以在我的GitHub仓库里找到这个操作系统和它的所有更新：https://github.com/sgmarz/osblog.

[目录](http://osblog.stephenmarz.com/index.html) → [第10章](http://osblog.stephenmarz.com/ch10.html) → (第11章)