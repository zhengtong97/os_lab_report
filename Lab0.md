
# Lab0 
##  看课程视频并阅读实验文档
## 了解汇编（练习一）
> Try below command  
> ```  
> $cd related_info/lab0  
> $gcc -S -m32 lab0_ex1.c \\注意这里-S是大写，否则gcc会找不到main函数  
> ```  
> Then you will get lab_ex1.S. Try to understand the content & relation of C file and S file.  

Relation of C file and S file: 生成的.s文件是对应的汇编代码

## 用gdb调试（练习二）
> Try below command  
> ```  
> $cd related_info/lab0`  
> $gcc -g -m32 lab0_ex2.c  
> ```  
> Then you will get a.out. Try to use gdb to debug lab0_ex2.  
我输入了
>  `$gcc -g -m32 lab0_ex1.c -o lab0_ex2`  
会生成lab0_ex2可执行文件；
运行a.out发现：
> `Segmentation fault(core dumped)`  
段错误（吐核），应该是访问了内存中不应该访问的东西。
试着用gdb调试:
1. 运行 “gdb lab0_ex2” ，加载lab0_ex2 可执行文件;
> $gdb lab0_ex2  
2. 执行装入的 bugging 命令;
> (gdb) run  
出现以下结果：
> Starting program: _home_moocos_moocos_ucore_lab_related_info_lab0/lab0_ex2   
>   
> Program received signal SIGSEGV, Segmentation fault.  
> 0x08048408 in main () at lab0_ex1.c:6  
> 6       asm(      
3. 使用 where 命令查看程序出错的地方；
> (gdb) where  
出现以下结果：
>  #0  0x08048408 in main () at lab0_ex1.c:6  
4. 利用 list 命令查看被调试程序的源代码；
出现以下结果：
> 1    int count=1;  
> 2    int value=1;  
> 3    int buf[10];  
> 4    void main()              
> 5    {                  
> 6       asm(      
> 7        "cld \n\t"  
> 8            "rep \n\t"                           
> 9            "stosl"  
> 10        :  
> 11        : "c" (count), "a" (value) , "D" (buf[0])  
> 12        :  
> 13          );  
> 14    }  
源代码一共有十四行。

这里代码的含义是：
> cld,rep,stos这几条语句的功能是向buf中写上count个value值。冒号后的语句指明输入，输出和被改变的寄存器。通过冒号以后的语句，编译器就知道你的指令需要和改变哪些寄存器，从而可以优化寄存器的分配。其中符号”c”(count)指示要把count的值放入ecx寄存器。  

> a eax   
> b ebx  
> c ecx  
> d edx  
> S esi  
> D edi  
> I 常数值，(0 - 31)  
> q,r 动态分配的寄存器  
> g eax,ebx,ecx,edx或内存变量  
> A 把eax和edx合成一个64位的寄存器(use long longs)  

5. 在 gdb 中，我们在第 11 行处设置断点，看看是否是在第行出错；
> (gdb) break 11  
> (gdb) run  
出现以下结果：
> Starting program: _home_moocos_moocos_ucore_lab_related_info_lab0/lab0_ex2   
>   
> Breakpoint 6, main () at lab0_ex1.c:11  
> 11        : "c" (count), "a" (value) , "D" (buf[0])  
6. 程序重新运行到第 11 行处停止，这时程序正常，然后执行单步命令next；
> (gdb) next  
出现以下结果：
> 6       asm(      
再次执行单步命令next，出现以下结果：
> Program received signal SIGSEGV, Segmentation fault.  
> 0x08048408 in main () at lab0_ex1.c:6  
> 6       asm(      


7. 程序确实出错，我认为能够导致 asm 函数出错的因素就是变量 buf[0]。重新执行测试程，用 print命令查看 buf 的值；
> (gdb) print buf  
出现以下结果：
> $1 = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0}  
将源码中的第11行改为：
`11        : "c" (count), "a" (value) , "D" (buf)`
重新编译，发现程序运行不出错，用gdb调试同样没再出现段错误。
原因在于D对应edi寄存器，是指针寄存器，只能存放指针，不能存放一般的变量数值，所以不能将buf[0]存入D，而只能将buf数组首地址buf存入D。


## 掌握指针和类型转换相关的Ｃ编程
> 对于如下代码段，  
> ```  
> # define STS_CG32        0xC             32-bit Call Gate  
> # define STS_IG32        0xE             32-bit Interrupt Gate  
> # define SETGATE(gate, istrap, sel, off, dpl) {            \  
>     (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \  
>     (gate).gd_ss = (sel);                                \  
>     (gate).gd_args = 0;                                    \  
>     (gate).gd_rsv1 = 0;                                    \  
>     (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \  
>     (gate).gd_s = 0;                                    \  
>     (gate).gd_dpl = (dpl);                                \  
>     (gate).gd_p = 1;                                    \  
>     (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \  
> }  
>   
>  _* Gate descriptors for interrupts and traps *_  
>  struct gatedesc {  
>     unsigned gd_off_15_0 : 16;         low 16 bits of offset in segment  
>     unsigned gd_ss : 16;             segment selector  
>     unsigned gd_args : 5;             # args, 0 for interrupt/trap gates  
>     unsigned gd_rsv1 : 3;             reserved(should be zero I guess)  
>     unsigned gd_type : 4;             type(STS_{TG,IG32,TG32})  
>     unsigned gd_s : 1;                 must be 0 (system)  
>     unsigned gd_dpl : 2;             descriptor(meaning new) privilege level  
>     unsigned gd_p : 1;                 Present  
>     unsigned gd_off_31_16 : 16;         high bits of offset in segment  
>  };  
>   
> ```  
>   
> 如果在其他代码段中有如下语句，  
> ```  
> unsigned intr;  
> intr=8;  
> SETGATE(intr, 0,1,2,3);  
> ```  
> 请问执行上述指令后， intr的值是多少？  
> **具体方法**：Try below command  
> ```  
> $cd related_info/lab0  
> gcc -g -m32 lab0_ex3.c 2>&1|tee make.log“2>&1”的意思是：将输出到标准出错处理的信息，发送到标准输出中。  
> ```  
> If you get gcc’s error, try to read make.log and fix the bugs. If gcc successed, then you will get a.out. Try to answer below question.  
这部分需要用实验楼提供的在线虚拟机。
按照提示输入命令后，出现结果，同时这也是生成的make.log的内容：
> lab0_ex3.c: In function \u2018main\u2019:  
> lab0_ex3.c:48:5: warning: format \u2018%llx\u2019 expects argument of type \u2018long long unsigned int\u2019, but argument 2 has type \u2018struct gatedesc\u2019 [-Wformat=]  
>      printf("gintr is 0x%llx\n",gintr);  
看起来是输出格式出了问题，应输出无符号长整型，而gintr是struct gatedesc类型的，需要将对应行注释掉就可以了。
`\\printf("gintr is 0x%llx\n",gintr);`
因为这个结构中有很多成员，且均为无符号整型，不知道要输出哪个，故根据题目意思应该是删去这一行命令。
## 掌握通用链表结构相关的Ｃ编程
哎呀 不想搞啦==|||
