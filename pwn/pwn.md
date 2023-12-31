# PWN

[TOC]

## ELF

### ELF文件的类型

ELF即”可执行可链接格式“，Linux系统上所运行的就是ELF格式的文件，相关定义在`/usr/include/elf.h`文件里。

ELF文件分为三种类型，可执行文件(.exec)、可重定位文件(.rel)和共享目标文件(.dyn)：

- 可执行文件：经过链接的、可执行的目标文件，通常也被称为程序。
- 可重定位文件：由源文件编译而成且尚未链接的目标文件，通常以.o作为扩展名。用于与其他目标文件进行链接以构成可执行文件或动态链接库，通常是一段位置独立的代码。
- 共享目标文件：动态链接库文件。用于在链接过程中与其他动态链接库或可重定位文件一起构建新的目标文件，或者在可执行文件加载时，链接到进程中作为运行代码的一部分。

除此之外，核心转储文件（core dump file）作为进程意外终止时进程地址空间的转储，也是ELF文件的一种。使用gdb读取这类文件  可以辅助调试和查找程序崩溃的原因。

### ELF文件的结构

通常目标文件都会包含三个节：

- 代码（.text）：用于保存可执行的机器指令，通常设置为只读以防止被改写和利用
- 数据（.data）：用于保存已初始化的全局变量和局部静态变量
- BSS（.bss）：用于保存未初始化的全局变量和局部动态变量

ELF文件头位于目标文件最开始的位置，包含描述整个文件的一些基本信息，例如ELF文件类型、版本、目标机器、程序入口、段表和节表的位置和长度等。文件头部存在魔术字符（7f 45 4c 46），即字符串”\177ELF“，当文件被映射到内存时，可以通过搜索该字符确定映射地址，这在dump内存时非常有用。

节头表保存目标文件的节信息，表的每一项都是一个`Elf64_Shdr`结构体，记录了节的名字、长度、偏移、读写权限等信息。节头表的位置记录在文件头的`e_shoff`域中。节头表对于程序运行并不是必须的，所以常有程序去除节头表以增加反编译器的分析难度。

## 静态链接

静态链接在地址空间分配时使用相似节合并，将不同目标文件相同属性的节合并为一个节，如将`main.o`与`func.o`的`.text`节合并为新的`.text`节，将`main.o`与`func.o`的`.data`节合并为新的`.data`节，当前的链接器首先对各个节的长度、属性和偏移进行分析，然后将输入目标文件中符号表的符号定义与符号引用统一生成全局符号表，最后读取输入文件的各类信息对符号进行解析、重定位等操作。相似节的合并就发生在重定位时。完成后，程序中的每条指令和全局变量就都有唯一的运行时内存地址了。

可重定位文件中最重要的就是要包含重定位表，用于告诉链接器如何修改节的内容。每一个重定位表对应一个需要被重定位的节，例如名为`.rel.text`的节用于保存`.text`节的重定位表。`.rel.text`包含两个重定位入口，shared的类型`R_X86_64_32`用于绝对寻址，CPU将直接使用在指令中编码的32位值作为有效地址。func的类型`R_X86_64_PC32`用于相对寻址，CPU将指令中编码的32位值加上PC（下一条指令地址）的值得到有效地址。

## 动态链接

GCC默认使用动态链接编译，通过下面的命令我们将`func.c`编译为共享库，然后使用这个库编译`main.c`。参数`-shared`表示生成共享库，`-fpic`表示生成与位置无关的代码。这样可执行文件`func.ELF2`就会在加载时与`func.so`进行动态链接。需要注意的是，动态加载器`ld-Linux.so`本身就是一个共享库，因此加载器会加载并运行动态加载器，并由动态加载器来完成其他共享库以及符号的重定位。

```shell
gcc -shared -fpic -o func.so func.c
gcc -fno-stack-protector -o func.ELF2 main.c ./func.so
ldd func.ELF2
```

### 位置无关代码

可以加载而无须重定位的代码称为位置无关代码（PIC），它是共享库必须具有的属性，通过PIC，一个共享库的代码可以被无限多个进程所共享，从而节约内存资源。

由于一个程序的数据段和代码段的相对距离总是**保持不变**的，因此，指令和变量之间的距离是一个运行时常量，与绝对内存地址无关。于是就有了全局偏移量表（GOT），它位于数据段的开头用于保存全局变量和库函数的引用，每个条目占8个字节，在加载时会进行重定位并填入符号的绝对地址。

实际上，为了引入RELRO保护机制，GOT被拆分为`.got`节和`.got.plt`节两个部分，前者不需要延迟绑定，用于保存全局变量引用，加载到内存后被标记为只读；需要延迟绑定的后者则用于保存函数引用，具有读写权限。

### 延迟绑定

延迟绑定的基本思想是当函数第一次被调用时，动态链接器才进行符号查找、重定位等操作，如果未被调用则不进行绑定。

ELF文件通过过程链接表（PLT）和GOT的配合来实现延迟绑定，每个被调用的库函数都有一组对应的PLT和GOT。

位于代码段`.plt`节的PLT是一个数组，每个条目占16个字节。其中`PLT[0]`用于跳转到动态链接器，`PLT[1]`用于调用系统启动函数`_libc_start_main()`，我们熟悉的`main()`函数就是在这里面调用的，从`PLT[2]`开始就是被调用的各个函数条目。

位于数据段`.got.plt`节的GOT也是一个数组，每个条目占8个字节。其中`GOT[0]`和`GOT[1]`包含动态链接器在解析函数地址时所需要的两个地址（`.dynamic`和`relor`条目），`GOT[2]`是动态链接器`ld-Linux.so`的入口处，从`GOT[3]`开始就是被调用的各个函数条目，这些条目默认指向对应PLT条目的第二条指令，完成绑定后才会被修改为函数的实际地址。

## 汇编基础

### 指令集架构

在Linux发行版中，将x86-64称为amd64，而x86则称为i386。

#### 常用汇编指令

**MOV:**MOV指令的基本格式中，第一个参数为目的操作数，第二个参数为源操作数。如语句`MOV EAX, ECX`表示将ECX寄存器的值拷贝到EAX中。MOV指令支持从寄存器到寄存器、从内存到寄存器、从寄存器到内存、从立即数到内存和从立即数到寄存器的数据传送。

**INC、DEC:**分别用于操作数加1和减1，操作数可以是寄存器也可以是内存。

**ADD、SUB:**将长度相同的操作数进行相加和从目的操作数中减去源操作数。

**NEG:**对操作数执行求补运算。

**JMP:**无条件跳转指令，在编写汇编语言时需要使用一个标号来标识，汇编器在编译时就会将该标号转换成响应的偏移量。一般来说，该标号必须和JMP指令位于同一函数中，但使用全局标号则不受限制。

```assembly
	JMP labell
	MOV EBX, 0
labell:
	MOV EAX, 0
```

**LOOP:**用以创建一个循环代码块，ECX寄存器为循环的计数器，没经过一次循环，ECX的值将减去1。

```assembly
MOV AX, 0
MOV ECX, 3
L1:
INC AX
LOOP L1
XOR EAX, EBX
```
**PUSH、POP:**对ESP\RSP\SP寄存器的值进行减法运算，并使其减去4（32位）或8（64位），将操作数写入上述寄存器中指针指向的内存中。POP指令先从ESP\RSP\SP寄存器指向的内存中读取数据写入其他内存地址或寄存器，再将栈指针的数据增加4或8。

**LEAVE:**相当于`MOV ESP, EBP; POP EBP`将esp设为ebp的值，并恢复保留的父函数ebp。

### 寄存器

| 操作数 |                        可用寄存器名称                        |
| :----: | :----------------------------------------------------------: |
|  8位   | AL、BL、CL、DL、DIL、SIL、BPL、SPL、R8L、R9L、R10L、R11L、R12L、R13L、R14L、R15L |
|  16位  | AX、BX、CX、DX、DI、SI、BP、SP、R8W、R9W、R10W、R11W、R12W、R13W、R14W、R15W |
|  32位  | EAX、EBX、ECX、EDX、EDI、ESI、EBP、ESP、R8D、R9D、R10D、R11D、R12D、R13D、R14D、R15D |
|  64位  | RAX、RBX、RCX、RDX、RDI、RSI、RBP、RSP、R8、R9、R10、R11、R12、R13、R14、R15 |

需要注意的是，在64位模式下，操作数的默认大小仍然为32位，且有8个通用寄存器；当给每条汇编指令增加REX（寄存器扩展）的前缀后，操作数变为64位，且增加了8个带有标号的通用寄存器（R8~R15）。

此外，64位处理器还有两个特点：第一，64位与32位有着相同的标志位状态；第二，64位模式下不能访问通用寄存器的高位字节（如AH、BH、CH及DH）。

### 数据类型

- 整数常量：对于整数常量，仅给出1234这类数字无法确定它的进制，因此需要使用后缀进行区分。此外，由于十六进制包含一些字母，为了避免汇编器解释成汇编指令或标识符，需要在以字母开头的十六机制数前加0表示，如0ABCDh（h表示十六进制）
- 浮点数常量：x86架构中有单独的浮点数寄存器和浮点数指令来处理相关浮点数常量。我们通常以十进制表示浮点数，而以十六进制编码浮点数。浮点数中至少包含一个整数和一个十进制的小数点。
- 字符串常量：字符串常量在内存中是以整数字节序列保存，下面是字符串'ABCDEFGH'在gdb中显示的样子，Intel处理器默认使用小端序表示字节序，TCP/IP协议的字节序使用大端。

```shell
gef x/s 0x4005d4
0x4005d4: 'ABCDEFGH'
gef x/gx 0x4005d4
0x4005d4: '0x4847464544434241'	// 小数端表示法
gef x/8x 0x4005d4
0x4005d4: 0x41 0x42 0x43 0x44 0x45 0x46 0x47 0x48 // 十六进制值对应字符的ASCII码
```

## Linux安全机制

### Linux调用约定

#### 内核接口

X86-32系统调用约定：Linux系统调用使用寄存器传递参数。eax为**syscall_number**，ebx、ecx、edx、esi和ebp用于将6个参数传递给系统调用。返回值保存在eax中。所有其他寄存器都保留在int 0x80中。

X86-64系统调用约定：内核接口使用的寄存器有rdi、rsi、rdx、r10、r8和r9。系统调用通过syscall指令完成。除了rcx、r11和rax，其他的寄存器都被保留（在系统调用时不要修改寄存器的值）。系统调用的编号必须在寄存器rax中传递。系统调用的参数限制为6个，不直接从堆栈上传递任何参数。返回时，rax中包含了系统调用的结果，而且只有integer或者memory类型的值才会被传递给内核。

### Stack Canaries

一种用于对抗栈溢出攻击的技术，即SSP安全机制，有时也叫作stack cookies。canary的值是栈上的一个随机数，在程序启动时随机生成并保存在比函数返回地址更低的位置。由于栈溢出是从低地址向高地址进行覆盖，因此攻击者想要控制函数的返回指针，就一定要先覆盖到canary，程序只需要在函数返回前检查canary是否被篡改，就可以达到保护栈的目的。

Canary通常分为3类：

- terminator canaries：由于许多栈溢出都是由于字符串操作（如strcpy）不当所产生的，而这些字符串以NULL"/x00"结尾，换个角度看也就是会被"/x00"所截断。基于这一点，terminator canaries将低位设置为"/x00"，既可以防止被泄露，也可以防止被伪造。截断字符还包括CR（0x0d）、LF（0x0a）、EOF（0xff）。
- random canaries：为防止canaries被攻击者猜到，random canaries通常在程序初始化时随机生成并保存在一个相对安全的地方。当然如果攻击者知道它的位置还是有可能被读出来。随机数通常由**/dev/urandom**生成，有时也使用当前时间的哈希。
- random XOR canaries：与random canaries类似，但多了一个XOR操作，这样无论是canaries被篡改还是与之XOR的控制数据被篡改，都会发生错误，这样就增加了攻击难度。

在Linux中，fs寄存器被用于存放线程局部存储（TLS），TLS主要是为了避免多个线程同时访存同一全局变量或者静态变量时所导致的冲突，尤其是多个线程同时需要修改这一变量时。TLS为每一个使用该全局变量的线程都提供一个变量值的副本，每一个线程均可以独立地改变自己的副本，而不会和其他线程的副本冲突。在glibc的实现中，`stack_guard`定义在TLS结构体偏移0x28的地方。在32位程序中canary则变成了gs寄存器偏移0x14的地方。程序加载时TLS中的canary被取出放到栈上，在函数返回前和TLS中的canary进行比较。

攻击canaries的主要目的是避免程序崩溃，那么就有两种思路：第一种将canaries的值泄露出来，然后在栈溢出时覆盖上去，使其保持不变；第二种则是同时篡改TLS和栈上的canaries，这样在检查的时候就能够通过。

### No-eXecute

No-eXecute（NX），表示不可执行，其原理是将数据所在的内存页（例如堆和栈）标识为不可执行，如果程序产生溢出转入执行shellcode时，CPU就会抛出异常。通常我们使用可执行空间保护（executable space protection）作为一个统称，来描述这种防止传统代码注入攻击的技术—攻击者将恶意代码注入正在运行的程序中，然后使用内存损坏漏洞将控制流重定向到该代码。实施这种保护的技术有多种名称，在Windows上称为数据执行保护（DEP），在Linux上则有NX、W^X、PaX、和Exec Shield等。

NX的实现需要结合软件和硬件共同完成。首先在硬件层面，它利用处理器的NX位，对相应页表项中的第63位进行设置，设置为1表示内容不可执行，设置为0则表示内容可执行。一旦程序计数器（PC）被放到受保护的页面内，就会触发硬件层面的异常。其次，在软件层面，操作系统需要支持NX，以便正确配置页表，但有时这会给自修改代码或者动态生成的代码带来一些问题，这时软件需要使用适当的API来分配内存，Linux上使用mprotect或mmap，这些API允许更改已分配页面的保护级别。

在Linux中，当装载器将程序装载进内存空间后，将程序的`.text`节标记为可执行，而其余的数据段（`.data、.bss`等）以及栈、堆均不可执行。因此，传统的通过修改GOT来执行shellcode的方式不再可行。但这种保护并不能阻止代码重用攻击（ret2libc）。

### ASLR

大多数攻击都基于攻击者知道程序的内存布局，因此，引入内存布局的随机化能够有效增加漏洞利用的难度，一种技术就是地址空间分布随机化（address space layout randomization，ASLR），在Linux上，ASLR的全局配置`/proc/sys/kernel/randomize_va_space`有三种情况：0表示关闭ASLR；1表示部分开启（将mmap的基址，stack和vdso页面随机化）；2表示完全开启。

| ASLR  | executable（main） | PLT（system） | heap | stack | shared libraries |
| :---: | :----------------: | :-----------: | :--: | :---: | :--------------: |
|   0   |         ×          |       ×       |  ×   |   ×   |        ×         |
|   1   |         ×          |       ×       |  ×   |   √   |        √         |
|   2   |         ×          |       ×       |  √   |   √   |        √         |
| 2+PIE |         √          |       √       |  √   |   √   |        √         |

### PIE

由于ASLR是一种操作系统层面的技术，而二进制程序本身是不支持随机化加载的，便出现了一些绕过方式，例如ret2plt、GOT劫持、地址爆破等。于是，人们于2003年引入了位置无关可执行文件（position-independent executable，PIE），它在应用层的编译器上实现，通过将程序编译为位置无关代码（PIC）使程序可以被加载到任意位置，就像是一个特殊的共享库。但是，无论是ASLR还是PIE，由于粒度问题，被随机化的都只是某个对象的起始地址，而在该对象的内部依然保持原来的结构，也就是说相对偏移是不会变的。

## 分析工具

### GDB

#### GDB的基本操作

- break -b
  - break：不带参数时，在所选栈帧中执行的下一条指令处下断点。
  - break<function>：在函数体入口处下断点。
  - break <line>：在当前源码文件指定行的开始处下断点。
  - break<address>：在程序指令地址处下断点。
- clear用法和break类似，功能改为清除指定位置的断点。
- info
  - info breakpoints --i b：查看断点，观察点和捕获点的列表。
  - info reg：查看当前寄存器信息。
  - info frame：打印指定栈帧的详细信息。
- step -s：单步步进，可附加参数N表示执行N次。
- next -n：单步步过。与step不同，当调用子程序时，此命令不会进入子程序，而是将其视为单个源代码行执行。

### IDA Pro Usage

## 漏洞利用开发

### Pwntools Usage

#### pwntools的常用子模块

- pwnlib.adb:安卓调试桥；
- pwnlib.asm：汇编和反汇编，支持i386、i686、amd64、thumb等；
- pwnlib.encoders：对shellcode进行编码；
- pwnlib.elf：操作ELF可执行文件和共享库；
- pwnlib.fmtstr：格式化字符串利用工具；
- pwnlib.libcdb：libc数据库；
- pwnlib.log：日志记录管理；
- pwnlib.rop：ROP利用工具；
- pwnlib.tubes：与sockets、processes、ssh等进行连接；
- pwnlib.util：一些实用小工具。

##### pwnlib.tubes

`p = process（'/bin/sh'）`

`r = remote（'127.0.0.1', 1080）`

`l = listen（1080）`

`s = ssh（host = 'example.com, user = 'name', password = 'passwd'）`

`interactive()`：交互模式，能够同时读写管道，通常在获得shell之后调用；

`recv(numb = 1096, timeout = default)`：接收最多numb字节的数据；

`recvn(numb, timeout = default)`：接收numb字节的数据；

`recvall()`：接收数据直到EOF；

`recvline(keepends = True)`：接收一行，可选择是否保留行尾的”\n“；

`recvrepeat(timeout = default)`：接收数据直到EOF或timeout；

`recvuntil(delims, timeout = default)`：接收数据直到delims出现；

`send(data)`：发送数据；

`sendafter(delim, data, timeout = default)`：相当于`recvuntil(delims, timeout = default)`和`send(data)`的组合；

`sendline(data)`：发送一行，默认在行尾加"\n"；

`sendlineafter(delim, data, timeout = default)`：相当于`recvuntil(delims, timeout = default)`和`sendline(data)`的组合；

`close()`：关闭管道。

##### pwnlib.context

该模块用于设置运行时变量，例如目标系统、目标体系结构、端序、日志等。

```shell
context.clear()		# 恢复默认值
context.os = 'linux'
context.arch = 'arm'
context.bits = 32
context.endian = 'little'
vars(context)
context.update(os = 'linux', arch = 'amd64', bits = 64)
context.log_level = 'debug'
context.log_file = '/tmp/pwnlog.txt'
```

##### pwnlib.elf

该模块用于操作ELF文件，包括符号查找、虚拟内存、文件偏移，以及修改和保存二进制文件等功能。

```shell
e = ELF('/bin/cat')
print hex(e.address)
print hex(e.symbols['write'], hex(e.got['write']), hex(e.plt['write']))
```

上面的代码分别获得了ELF文件装载的基地址、符号地址、GOT地址和PLT地址。我们也可以用它加载一个libc。

```shell
e = ELF('/lib/x86_64-linux-gnu/libc.so.6')
print hex(e.symbols['system'])
```

也可以修改ELF文件的代码

```shell
e = ELF('/bin/cat')
e.read(e.address + 1, 3)
e.asm(e.address, 'ret')
e.save('/tmp/quiet-cat')
print disasm(file('/tmp/quiet-cat', 'rb').read(1))
```

`asm(address, assembly)`：汇编指令assembly并将其插入ELF的address地址处，需要使用ELF.save()函数来保存；

`bss(offset)`：返回.bss段加上offset后的地址；

`checksec()`：查看文件开启的安全保护；

`disable_nx()`：关闭NX；

`disasm(address, n_bytes)`：返回对地址address反汇编n字节的字符串；

`offset_to_vaddr(offset)`：将偏移offset转换为虚拟地址；

`vaddr_to_offset(address)`：将虚拟地址address转换为文件偏移；

`read(address, count)`：从虚拟地址address读取count个字节的数据；

`write(address, data)`：在虚拟地址address写入data；

`section(name)`：获取name段的数据；

`debug()`：使用gdb.debug()进行调试。

## 整数安全

计算机中的整数分为有符号整数和无符号整数两种，通常保存在一个固定长度的内存空间内，它能存储的最大值和最小值都是固定的，在32位系统和64位系统中值得关注的取值范围有

```c++
C数据类型                           最小值                          最大值
  char                             -128                           127
  unsigned char                      0                            255
// 32位
  long                        -2 147 483 648                  2 147 483 647
  unsigned long                      0                        4 294 967 295
// 64位
  long                  -9 223 372 036 854 775 808    9 223 372 036 854 775 807
  unsigned long                      0               18 446 744 073 709 551 615
```

### 整数溢出

关于整数的异常情况主要有三种：

1. 溢出，只有有符号数才会发生溢出。有符号数的最高位表示符号，在两正或两负相加时，有可能改变符号位的值，产生溢出。
2. 回绕，无符号数0-1会变成最大的数，如1字节的无符号数会变为255，而255+1会变成最小数0。
3. 截断，将一个较大宽度的数存入一个宽度小的操作数中，高位发生截断。

```c
int i;
i = INT_MAX;		// 2 147 483 647
i++;					  // 上溢出	
printf("i = %d\n", i);		// i = -2 147 483 648

i = INT_MIN;		// -2 147 483 648
i--;						// 下溢出
printf("i = %d\n", i);		// i = 2 147 483 647

unsigned int ui;
ui = UINT_MAX;		// 在x84-32上为 4 294 967 295
ui++;
printf("ui = %u\n", ui);		// ui = 0
ui = 0;
ui--;
printf("ui = %d\n", ui);		// 在x84-32上为 4 294 967 295

0xffffffff + 0x00000001
  = 0x0000000100000000		(long long)
  = 0x00000000		(long)

0x00123456 * 0x00654321
  = 0x000007336BF94116		(long long)
  = 0x6BF94116		(long)
```

## 格式化字符串

### 格式化输出函数

c语言中定义的变参函数由固定数量（至少一个）的强制参数和数量可变的可选参数组成，强制性参数在前，可选参数在后，可选参数的类型可以变化，而数量由强制参数的值或者用来定义可选参数列表的特殊值决定。C语言标准定义了下面的格式化输出函数。

```c
#include <stdio.h>
/*
省略号表示可选参数
*/

/* 按照格式字符串将输出写入流中。三个参数分别是流、格式字符串和变参列表 */
int fprintf(FILE *stream, const char *format, ...);
/* 等同于输出流为stdout的fprintf() */
int printf(const char *format, ...);
/* 等同于输出是文件描述符的fprintf() */
int dprintf(int fd, const char *format, ...);
/* 等同于输出不是写入流而是写入数组,在写入的字符串末尾会自动添加'/0' */
int sprintf(char *str, const char *format, ...);
/* 等同于sprintf(),但指定了可写入字符的最大值size，超过size-1的部分被丢弃，并在末尾添加一个空字符 */
int snprintf(char *str, size_t size, const char *format, ...);
```

格式化字符串由普通字符（包括"%"）和转换规则构成的字符序列。普通字符被原封不动地复制到输出流中。转换规则根据与实参对应的转换指示符对其进行转换，然后将结果写入输出流中。常见的规则可参见[CTF Wiki](https://ctf-wiki.org/pwn/linux/user-mode/fmtstr/fmtstr-intro/)。

### 格式化字符串漏洞

在x86结构下，格式化字符串的参数是通过栈传递的。根据cdecl的调用约定，在进入printf()函数之前，程序将参数从右到左依次压栈。进入printf()函数之后，函数首先获取第一个参数，一次读取一个字符。如果字符不是“%”，那么字符被直接复制到输出。否则，读取下一个非空字符，获取相应的参数并解析输出。如下面这个例子：

```c
void main(){
		printf("%s %d %s %x %x %x %3$s", "Hello World!", 233, "\n");
}
Hello world! 233
  f7fb53dc ffffce00 0
```

程序打印出了七个值，而参数只有三个，分析可知三个“%x”打印的是0xffffcde0～0xffffcde8的数据，而最后一个参数“%3$s“则是对第三个参数”\n“的重用。

对于格式化字符串漏洞的利用主要有：使程序崩溃、栈数据泄露、任意地址内存泄露、栈数据覆盖、任意地址内存覆盖。

#### 使程序崩溃

通常，使用类似下面的格式化字符串即可触发崩溃。原因有3点：

1. 对于每一个“%s”，printf()都要从栈中获取一个数字，将其视为一个地址，然后打印出地址指向的内存，直到出现一个空字符；
2. 获取的某个数字可能并不是一个地址；
3. 获得的数字确实是一个地址，但该地址是受保护的。

```c
printf("%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s%s")
```

#### 栈数据泄露

攻击者可以利用格式化函数获得内存数据，为漏洞利用做准备。格式化函数会根据格式化字符串从栈上取值，由于x86的栈由高地址向低地址增长，同时printf()函数的参数是以逆序被压入栈的，所以参数在内存中出现的顺序与在printf()调用时出现的顺序是一致的。

如果我们想要按顺序泄露栈数据，可以输入“%08x.%08x.%08x.%08x“作为格式字符串，要求printf()从栈中取出5个参数并将它们以8位十六进制数的形式打印。如果想要直接泄露指定的某个数据，则可以使用类似于“%n$x“的格式化字符串（x也可以替换为其它参数，如s等），n表示位于格式字符串后的第n个数据。

#### 任意地址内存泄露

攻击者使用类似“%s”的格式规范就可以泄露出参数（指针）所指向内存的数据，程序会将它作为一个ASCII字符串处理，直到遇到一个空字符。所以，如果攻击者能够操纵这个参数的值，那么就可以泄露任意地址的内容。利用这种方法，我们可以把某函数的GOT地址传进去，从而获得所对应函数的虚拟地址。然后根据函数在libc中的相对位置，就可以计算出任意函数地址，例如system()。

#### 栈数据覆盖

我们可以通过格式化字符串修改栈和内存来劫持程序的执行流。“%n”转换指示符将当前已经成功写入流或缓冲区中的字符个数存储到由参数指定的整数中。

## ROP gadget

在栈缓冲区溢出的基础上，利用程序中已有的小片段(gadgets) 来改变某些寄存器或者变量的值，从而控制程序的执行流程。所谓gadgets 就是以ret 结尾的指令序列，通过这些指令序列，我们可以修改某些内存或者寄存器的值。

```python
# system函数向上找两个字长执行system(/bin/sh)后pop出自己的参数并执行下一个函数
p32(system_addr) + p32(pop_ebx_ret) + p32(binsh_addr) + p32(puts_addr) + p32(pop_ebx_ret) + p32(addr) + p32(exit_addr) + p32(pop_ebx_ret) + p32(0)
```

