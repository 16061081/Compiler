# 记录

mips的栈空间是0~0x2ffc，即\$sp默认是0x2ffc. 之后我们不停地减少\$sp来存储到栈空间。

## 运行栈设计

| 运行存储内容 | 相对函数栈顶的偏移 | 说明 |
| --- | --- | --- |
| 参数 | 0 |  |
| 函数的局部变量 | 参数个数 | 要注意数组的情形计算 |
| $sp | 参数个数+局部变量个数 |  |
| $ra | 参数个数+局部变量个数+1 |  |
| $fp | 参数个数+局部变量个数+2 |  |
| 中间代码产生的变量 | 参数个数+局部变量个数+3 |  |


index_proc是分程序索引表，代表的是每个程序的第一个声明的标识符在符号表的第几位，由这个可以推断出每个函数有多少参数+局部常变量()。其中0..index_proc[1]-1存放的是全局常变量。 下图即代表有7个全局常量或变量。另外，index_proc.back()~cnt_tab的都是main函数里定义的常变量。例如下图的tab[20]是main函数，cnt_tab=22，代表还有2个在main函数里定义的常变量。这在我们之后寻找四元式的变量在符号表的位置很有用。

Program 1 :  8
Program 2 :  12
Program 3 :  18
Program 4 :  20


取消了对于x = a[i]会采用LOAD x a j这种四元式，直接归并成= x a[i]这种形式，在生成mips代码时直接对a[i]进行分析。同理对于a[i] = j这种形式也直接归并到赋值语句四元式的范畴。

在四元式中增加了**BEGIN**这个标记。这个是我人为加上的一个标记，代表的是函数中变量定义结束的位置，在这个时候我进行了ra和sp压到运行栈的步骤。

文法中的变量种类按其产生来源包括三类：

* 原测试程序中的定义的常变量，包括全局常变量、局部常变量、参数。都存储在符号表里。
* 中间代码产生的临时变量。统一为int类型。
* 函数的返回值RET。从某种意义上来说，这个值是由中间代码产生的，但强调这个是因为它的特殊性：这个变量具有类型。比如我们print(max('a', 'b')); max是一个char的函数，我们肯定输出的是一个字符而非整数。

为了能够区分返回的RET是什么类型，我把中间代码原来的返回值RET改成了RET_int或者RET_char代表调用的函数是什么类型。

提前设计的能力，是否能够在编码之前预料到之后可能会发生的各种情况。比如我一开始没有把标签作为一个单独的指令，而是直接与某条中间代码或者mips语句绑定，因此我的四元式和mips指令的struct一开始都是有一个类型为int的label，存放这句话是否对应了某个标签，默认值都是-1代表不对应任何标签。这种我自认为比较优雅的设计直到我遇到了if-else语句的else后面跟着的是一个空语句即`else ;`这样的情况，我的代码首先会把if语句的不满足对应到else后面的那句话，然后if执行结束的最后也会绑定到这句话，此时问题就发生了，之后的那句绑定就会覆盖掉之前的绑定。因此，我不得不改成vector<int> label来存储对应的多个label，本来自认为优雅的设计瞬间就被直接把label作为一条单独的四元式给秒杀了。因此，如果在初始设计数据结构的时候就能预料到这样的特殊情况，就会避免不必要的重构麻烦。

ToDo:

* 改造条件判断的四元式，使得都是slt，因为其它的都是扩展指令，做一条要4次，而做slt这个原始指令只需要2次。
* 关于变量和函数返回值RET重名，函数和LABEL_x重名的问题。
* ~~最后的一句语句增加一句气泡，防止出现打了一个LABEL但没有语句接收这个label的情况。~~

* ~~查找代码里的//!!!用mp_func进行修复。~~

* 针对转义字符原样输出的问题。目前针对”\n”这种形式的字符串，我的代码输出了换行(Mars默认的处理方式)而不是原样输出。（解决方案：最开始的时候，读入方式用逐个读字符而不是读完整个字符串。这样就可以把\转义成\\然后输出就不会被转义了。）


### 关于数组引用

首先，在初始形成的四元式中，我并没有单独的分割出数组引用的方式。而是仍然归并到原来的类型，例如赋值语句等。做法就是把a[i]拼成一个整体作为一个操作数。但是，我在第三个操作数中分别写了[]=和=[]这样的提示，便于之后判断是不是数组引用的形式吧。

![](/Users/Mr.ZY/GitHub/Compiler/pic1.jpg)

* a[i] = x形式
这种形式的赋值语句一定是某个基本块的最后一句。即它们会分割出基本块。

* x = a[i]形式
仍需要考虑优化。

处理的语义错误：

* switch后的case类型和switch变量的类型不匹配。输出!!!ERROR Incorrect type with switch_Linex$$$
* case后出现的情况有重复的：!!!ERROR::: SAME case_2$$$
* 函数/过程返回的结果的类型与该函数/过程的类型不一致，要报错。
* 函数调用的值参和函数定义的参数列表的类型不一致，输出```"!!!ERROR: parameter type not coincide with xxxx.$$$"```。之后会指出是第一个参数的类型错了，例如```"***MISS The 2 Parameter from int to char.$$$"```
* ~~只要是字符型向整型转换，都报错。即不允许x = 'a'; 其中a是int型。~~
* 数组的下标不能越界。例如定义的是int a[100];然后访问a[9999]
* 数组的下标不能是字符型。例如a['k']
* 访问一个数组元素但是却没有用下标即直接a=1; a是一个数组元素
* 对于常量的赋值，输出"!!!ERROR:::Const value cannot be changed:::" 




对于常数表达式的优化其实是很容易的。比如从`q=1+(2+3)*4;`我们可以非常轻松地就转换成`q=21`。只需要在词法分析expression的时候检验一下op2和op3是不是都是整数，然后直接计算(+/-)出这个结果给op1就行，对于term的*/同理。然后都不需要考虑()，因为这个递归的结果最终会体现到op2 op3里。

PUSH参数的时候，就直接塞到内存区。之后在函数里再使用时，我们就权且当做一个函数里的局部变量/临时变量来使用。这样的好处是耦合度低，不然又是一堆判断。

对于临时寄存器池的加载，一个优化： 如果是=左部的值，即要修改某个变量，我们从临时寄存器池分配一个寄存器的时候，不管怎样都不需要lw（原因是显然的，因为这个值马上就要改变）

去掉这些

\#\#14 = 2 * #13
mul	$t2,2,$t1 (没有这种语法)

不得不


### DAG图不需要补偿代码的原因



tmp = a;
    a = b;
    b = tmp;
    
    叶节点是a，然后b和tmp都
    
    #tmp = a
move	$t2,$t0
\#b = a
move	$t1,$t0
\#a = b


localRegEndBlockOut这个宏对应的优化是：如果不是全局变量，那么我们就检查这个块的out集合里有没有这个元素，如果没有就不需要再sw回内存区了。

