最近在看ucosII基于Cortex-M3的源码。遇到了一些疑问，然后上网查了下资料，做了些实验，有了一些体会，写下来用作以后复习。

---

抛出问题：函数返回的BX LR和异常（中断）返回BX LR指令都有什么区别？？

几个容易混淆的问题：  


- 调用函数时候MCU不会主动将寄存器入栈，可是异常中断进入中断服务函数MCU会主动将R0、R1、R2、R3、R12、LR、PC、xPSR寄存器入栈。
- 调用函数的LR和中断服务函数的LR有不同的意义。
- 离开普通函数和异常中断函数时执行BX LR的行为是不一样的。  

实验：  
	
	void bb()
	{}
	void aa()
	{
    	bb();
	}
                              
	int main(void)
	{	
		bb();
    	aa();
    	Test(); 
    	while( 1 );
	}

	下面的是汇编，通过软件触发PSV中断。
	Test PROC

    CPSID   I
    
    LDR     R0, =NVIC_SYSPRI14                                  
    LDR     R1, =NVIC_PENDSV_PRI
    STRB    R1, [R0]
    
    LDR     R0, =NVIC_INT_CTRL                                  
    LDR     R1, =NVIC_PENDSVSET
    STR     R1, [R0]
    
    CPSIE   I 
    
    B   .

通过KEIL5仿真该段程序，并且观察寄存器和堆栈的变化。

----

![图1](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/1.png)

即将进入第一个bb（）函数时候如上图所示。

![图2](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/2.png)

当进入了第一个bb（）函数时候，LR寄存器为0x0800033D，这个时候的LR寄存器保存的是跨过bb（）函数之后第一条指令的地址，也就是跳转到aa()的指令的地址，为什么不是0x800033C呢，M3权威手册有这么一段话：
>如果向 PC 中写数据，就会引起一次程序的分支（ 但是不更新 LR 寄存器）。 CM3 中的指
令至少是半字对齐的，所以 PC 的 LSB 总是读回 0。然而， 在分支时，无论是直接写 PC 的值
还是使用分支指令，都必须保证加载到 PC 的数值是奇数（即 LSB=1） ，用以表明这是在
Thumb 状态下执行。倘若写了 0，则视为企图转入 ARM 模式，CM3 将产生一个 fault 异
常。

也就是说，为了防止在函数返回时执行BX LR，会将LSB为0的地址写进PC寄存器，所以LR寄存器会被改为原指令地址+1，这两个地址其实都是同一条指令。然后可以发现，这时的堆栈是干净的，也就是这个调用函数的过程没有对寄存器进行压栈。

![图3](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/3.png)

当执行BX LR指令退出bb（）之后，就会刷新PC寄存器。

![图4](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/4.png)

进入aa（）函数之后，情况和第一次进入bb（）函数一样，并没有进行压栈操作。不过，从汇编中可以看到，即将会进行一次压栈操作，并调用bb（）函数。如果不这样进行压栈操作，那么进入bb（）函数之后改变了lr寄存器，就会导致无法从aa（）函数返回到main（）函数，导致程序跑飞，不过这个压栈的过程，是编译器帮我们实现的，对于MCU来说，整个函数调用过程都没有主动压栈过。

![图5](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/5.png)

执行了压栈指令之后，我们可以从memory看到，Test（）的指令地址被保存进栈了，这个指令地址就是跨过aa（）函数的第一条指令地址。同时R13寄存器也被改变了。

![图6](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/6.png)
![图7](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/7.png)

执行了跳转指令和BX LR指令，堆栈没有动过，没有什么特别变化，直到出栈指令。

![图8](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/8.png)

出栈指令执行之后，R13改变为0X20000400，同时栈里面的0x80000341会POP到PC寄存器上，这就是一个常见的小技巧，将保存的LR寄存器内容直接弹到PC里面去。

至此，通过8张图说明了，普通调用普通函数时候LR寄存器和PC寄存器的变化。  

总结：

1. 从main即将进入bb（）时候，寄存器不压栈，lr记录跨过bb（）紧跟的第一条指令，当从bb（）返回main时候，直接将lr弹出到PC里面去。
2. 从main即将进入aa（）时候，寄存器不压栈，lr记录跨过aa（）紧跟的第一条指令，当即将调用bb（）时候，将lr压栈，然后跳转到bb（），再返回到aa（），当从aa（）返回到main时候，将保存在栈中的lr弹出到PC里面去。

---

接下来的就是异常（中断）的例子了。

![图9](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/9.png)

上图所示是进入PSV中断前的各个状态，当执行CPSIE I打开中断之后，就会马上进入中断。

![图10](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/10.png)

进入中断后，我们可以看到，R13，R14，R15寄存器都改变了。R13改变说明栈顶改变了，从Memory看到栈里存的内容是入中断前的R0、R1、R2、R3、R12、LR、PC、xPSR的寄存器内容，这里的PC有点类似于调用普通函数时候的LR，保存从中断函数返回的地址，栈里的PC好像和上一张图的PC不一样，是因为上一张图点“仿真下一步”的时候，PC就改变了，改变后还没截图，就已经立刻进入了中断，所以给人的错觉就是+2了，其实并没有。R14寄存器这时候被重定义为一种叫exc-return的值，至于0xfffffff9具体是什么意思，可以查相关手册，是用来从中断函数返回被中断函数时候，告诉MCU，接下来需要使用MSP还是PSP堆栈，是继续待在特权模式或者切换到用户模式。

![图11](https://raw.githubusercontent.com/HJDonv/Cortex-M3-note/master/LR%E5%AF%84%E5%AD%98%E5%99%A8%E3%80%81PC%E5%AF%84%E5%AD%98%E5%99%A8%E5%9C%A8%E8%B7%B3%E8%BD%AC%E5%88%B0%E5%87%BD%E6%95%B0%E5%92%8C%E5%BC%82%E5%B8%B8%E7%9A%84%E5%8C%BA%E5%88%AB/Picture/11.png)

当执行BX lr的时候，会将0xfffffff9弹进PC寄存器，这时候MCU会识别这个exc-return的值，并进行从中断函数返回的操作，将栈里的内容按照顺序弹到寄存器里面去。

总结：

1. 函数将被打断瞬间，MCU硬件自动将R0、R1、R2、R3、R12、LR、PC、xPSR寄存器压栈，并将当前LR寄存器重定义为exc-return的值，然后进入中断函数，从中断函数返回到被打断函数时，将exc-return弹到PC寄存器，触发中断返回操作，然后MCU硬件自动将当前R13指向的栈内容按顺序弹出到寄存器里面去。
