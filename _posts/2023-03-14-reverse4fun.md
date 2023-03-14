---
layout:     post   				    # 使用的布局（不需要改）
title:      2023-03-14-reverse4fun
subtitle:   防守+进攻？ #副标题
date:       2023-03-14 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# reverse4fun



## Reverse Part

### 1 license checker 0x03

crackme的下载路径在这里：https://crackmes.one/crackme/62072dd633c5d46c8bcbfd9b，本身的代码非常直接，所以可以直接看disass出来的代码



```assembly
        (gdb) disass
        Dump of assembler code for function main:
           0x0000555555555179 <+0>:       push   %rbp             #标准开局，保存堆栈环境
           0x000055555555517a <+1>:       mov    %rsp,%rbp
           0x000055555555517d <+4>:       sub    $0x30,%rsp
           #非常标准的对应main函数的两个参数int main(int argc, char** argv),这个是argc
           0x0000555555555181 <+8>:     mov    %edi,-0x24(%rbp)
           0x0000555555555184 <+11>:    mov    %rsi,-0x30(%rbp)  #对应argv
           0x0000555555555188 <+15>:    mov    %fs:0x28,%rax
           0x0000555555555191 <+24>:    mov    %rax,-0x8(%rbp)
           0x0000555555555195 <+28>:    xor    %eax,%eax
           # #比较有没有参数，换言之有没有输入参数，如果有输入参数跳转到下面箭头指的地方
           0x0000555555555197 <+30>:    cmpl   $0x2,-0x24(%rbp)
-------    0x000055555555519b <+34>:    je     0x5555555551c5 <main+76>
|          # 没输入参数的话直接开始调用printf函数打印下面的格式
|          0x000055555555519d <+36>:    mov    -0x30(%rbp),%rax
|          0x00005555555551a1 <+40>:    mov    (%rax),%rax
|          0x00005555555551a4 <+43>:    mov    %rax,%rsi
|          # 格式的字符对应"Usage : %s <license pass code here [numbers only]>\n"
|          0x00005555555551a7 <+46>:    lea    0xe5a(%rip),%rax        # 0x555555556008
|          0x00005555555551ae <+53>:    mov    %rax,%rdi
|          0x00005555555551b1 <+56>:    mov    $0x0,%eax
|          0x00005555555551b6 <+61>:    callq  0x555555555050 <printf@plt>
|          0x00005555555551bb <+66>:    mov    $0x0,%edi
|          0x00005555555551c0 <+71>:    callq  0x555555555070 <exit@plt>
|          #下面的两个地址存储着两个参数
|          # 清空两个数值，这里存储着一个计算出来的和
------>    0x00005555555551c5 <+76>:    movl   $0x0,-0x10(%rbp)
           #这个地址存储的是处理过的字符串的字符的个数
           0x00005555555551cc <+83>:    movl   $0x0,-0xc(%rbp)
        -- 0x00005555555551d3 <+90>:    jmp    0x555555555201 <main+136>
-------->  0x00005555555551d5 <+92>:    mov    -0x30(%rbp),%rax #还是拿到输入的数字的位置
|       |  0x00005555555551d9 <+96>:    add    $0x8,%rax
|       |  0x00005555555551dd <+100>:   mov    (%rax),%rdx
|       |  0x00005555555551e0 <+103>:   mov    -0xc(%rbp),%eax     # 这个存储着处理过的字符串字符的个数，第一次是0
|       |  0x00005555555551e3 <+106>:   cltq
|       |  0x00005555555551e5 <+108>:   add    %rdx,%rax           #找到当前处理的字符
|       |  #取rax，实际上就是输入的数字所在的字符串的1byte，拓展到eax里
|       |  0x00005555555551e8 <+111>:   movzbl (%rax),%eax
|       |  0x00005555555551eb <+114>:   mov    %al,-0x11(%rbp)  #丢到栈里面,可以看到是刚才-0x10(%rbp)的低一位
|       |  0x00005555555551ee <+117>:   lea    -0x11(%rbp),%rax
|       |  0x00005555555551f2 <+121>:   mov    %rax,%rdi
|       |  #转换为整数，整个过程是一个循环，就是在不断地将每一位转换为数字，然后加到-0x10(%rbp)上
|       |  0x00005555555551f5 <+124>:   callq  0x555555555060 <atoi@plt>
|       |  #eax存储着从字符转换为数字的结果（返回值）,存储到求和到刚才舒适的存储和的位置
|       |  0x00005555555551fa <+129>:   add    %eax,-0x10(%rbp)
|       |  #addl加了一个1，实际上就是诸位比较，还是继续运行，
|       |  0x00005555555551fd <+132>:   addl   $0x1,-0xc(%rbp)
|       |  #argv的起始地址，
|       --->0x0000555555555201 <+136>:  mov    -0x30(%rbp),%rax
|          # 偏移8字节，实际上目的是找到第二个元素,也就是我们输入参数所对应字符串的地址
|          0x0000555555555205 <+140>:   add    $0x8,%rax
|          # 将元素的值赋值给rax，就是输入的数字的所在字符串的地址，
|          0x0000555555555209 <+144>:   mov    (%rax),%rax
|          0x000055555555520c <+147>:   mov    %rax,%rdi
|          # 对字符串调用strlen判断长度
|          0x000055555555520f <+150>:   callq  0x555555555040 <strlen@plt>
|          # 返回的字符串长度放在了eax里面，和已经操作的字符串的字符个数比较一下，看看是不是处理完了
|          0x0000555555555214 <+155>:   cmp    %eax,-0xc(%rbp)
---        0x0000555555555217 <+158>:   jl     0x5555555551d5 <main+92>
           #比较输入的参数字符串每一个转换为数字后加起来以后是不是0x32，所以构造一个字符串各位求和是0x32的即可
           0x0000555555555219 <+160>:   cmpl   $0x32,-0x10(%rbp)
           0x000055555555521d <+164>:   jne    0x555555555238 <main+191>
           0x000055555555521f <+166>:   lea    0xe1a(%rip),%rax        # 0x555555556040
           0x0000555555555226 <+173>:   mov    %rax,%rdi
           0x0000555555555229 <+176>:   callq  0x555555555030 <puts@plt>
           0x000055555555522e <+181>:   mov    $0x0,%edi
           0x0000555555555233 <+186>:   callq  0x555555555070 <exit@plt>
           0x0000555555555238 <+191>:   lea    0xe25(%rip),%rax        # 0x555555556064
           0x000055555555523f <+198>:   mov    %rax,%rdi
           0x0000555555555242 <+201>:   callq  0x555555555030 <puts@plt>
           0x0000555555555247 <+206>:   mov    $0x0,%edi
           0x000055555555524c <+211>:   callq  0x555555555070 <exit@plt>
```



### 2 trycrackme

trycrackme的下载地址为https://crackmes.one/crackme/61c8deff33c5d413767ca0ea，直接从汇编就能看出来到底在做什么

```asm
     Dump of assembler code for function main:
        # 标准开局，保存堆栈
        0x00005555555551af <+0>:        push   %rbp
        0x00005555555551b0 <+1>:        mov    %rsp,%rbp
        0x00005555555551b3 <+4>:        sub    $0xe0,%rsp
        #存储argc & argv，不过这里没用到
        0x00005555555551ba <+11>:       mov    %edi,-0xd4(%rbp)
        0x00005555555551c0 <+17>:       mov    %rsi,-0xe0(%rbp)
        0x00005555555551c7 <+24>:       mov    %fs:0x28,%rax
        0x00005555555551d0 <+33>:       mov    %rax,-0x8(%rbp)
        0x00005555555551d4 <+37>:       xor    %eax,%eax
        # 注意这个常量，是我们比较的关键
        0x00005555555551d6 <+39>:       movabs $0x3534323773734034,%rax
        # 存储到了-0xbb(%rbp)的位置
        0x00005555555551e0 <+49>:       mov    %rax,-0xbb(%rbp)
        # 给上面的常量补充了两个字节的数字，拼接到最后面,拼接完以后看一下具体的内容
        # (gdb) x/16xb 140737488347509
        # 0x7fffffffe175: 0x34    0x40    0x73    0x73    0x37    0x32    0x34    0x35
        # 0x7fffffffe17d: 0x33    0x36    0x00    0x00    0x00    0x00    0x00    0x00
        0x00005555555551e7 <+56>:       movw   $0x3633,-0xb3(%rbp)
        0x00005555555551f0 <+65>:       movb   $0x0,-0xb1(%rbp)
        0x00005555555551f7 <+72>:       lea    -0xbb(%rbp),%rax
        0x00005555555551fe <+79>:       mov    %rax,%rdi
        0x0000555555555201 <+82>:       callq  0x555555555050 <strlen@plt>
        # 存储我们刚才拼接出来的字符串长度，明确地看出来是10
        0x0000555555555206 <+87>:       mov    %eax,-0xc0(%rbp)
        0x000055555555520c <+93>:       mov    $0x0,%eax
        # 调用banner打印一些flag，没有啥用，不用管
        0x0000555555555211 <+98>:       callq  0x555555555199 <banner>
        # 准备调用printf提示用户输入数据，格式化字符串为Put the key:
        0x0000555555555216 <+103>:      lea    0xef9(%rip),%rax        # 0x555555556116
        0x000055555555521d <+110>:      mov    %rax,%rdi
        0x0000555555555220 <+113>:      mov    $0x0,%eax
        0x0000555555555225 <+118>:      callq  0x555555555070 <printf@plt>
        # 记住这个-0xb0(%rbp)的地址，这个是存储scanf输入进来的地址
        0x000055555555522a <+123>:      lea    -0xb0(%rbp),%rax
        0x0000555555555231 <+130>:      mov    %rax,%rsi
        0x0000555555555234 <+133>:      lea    0xee9(%rip),%rax        # 0x555555556124
        0x000055555555523b <+140>:      mov    %rax,%rdi
        0x000055555555523e <+143>:      mov    $0x0,%eax
        0x0000555555555243 <+148>:      callq  0x555555555080 <__isoc99_scanf@plt>
        # 初始化两个变量，分别存储上面-0xbb(%rbp)字符串处理过的字符个数
        # 和要算出来作为正确的code所处理过的字符个数
        0x0000555555555248 <+153>:      movl   $0x0,-0xc8(%rbp)
        0x0000555555555252 <+163>:      movl   $0x0,-0xc4(%rbp)
   |--- 0x000055555555525c <+173>:      jmp    0x5555555552ab <main+252>
   |    # -0xc8(%rbp)是刚才那个常量字符串处理过的byte数，所以cltq下面那句就很明显了
|-----> 0x000055555555525e <+175>:      mov    -0xc8(%rbp),%eax
|  |    0x0000555555555264 <+181>:      cltq
|  |    # 这里的意思是把刚才-0xbb(%rbp)字符串的字符,hex形式丢到eax
|  |    0x0000555555555266 <+183>:      movzbl -0xbb(%rbp,%rax,1),%eax
|  |    # 先做有符号数拓展，再做无符号数拓展，不过都小于0x80，所以无所谓了
|  |    0x000055555555526e <+191>:      movsbl %al,%eax
|  |    0x0000555555555271 <+194>:      movzbl %al,%eax
|  |    0x0000555555555274 <+197>:      mov    -0xc4(%rbp),%edx
|  |    0x000055555555527a <+203>:      movslq %edx,%rdx
|  |    # rcx存储了计算结果的首地址， -0x70(%rbp)是我们最后比较的参照物的地址
|  |    0x000055555555527d <+206>:      lea    -0x70(%rbp),%rcx
|  |    # 加上已经结算过的结果，实际上就是挪动指针,存储下面sprintf的结果
|  |    0x0000555555555281 <+210>:      add    %rdx,%rcx
|  |    0x0000555555555284 <+213>:      mov    %eax,%edx
|  |    # 这个字符是 %02x
|  |    0x0000555555555286 <+215>:      lea    0xe9a(%rip),%rax        # 0x555555556127
|  |    0x000055555555528d <+222>:      mov    %rax,%rsi
|  |    0x0000555555555290 <+225>:      mov    %rcx,%rdi
|  |    0x0000555555555293 <+228>:      mov    $0x0,%eax
|  |    # 这里调用sprintf的含义就非常清楚了，从hex编码转换为字符串
|  |    # 原先是hex 0x34，那么转换为字符串"34"
|  |    0x0000555555555298 <+233>:      callq  0x555555555090 <sprintf@plt>
|  |    # hex 字符串处理过一byte后挪一
|  |    # 而sprintf处理的结果是2byte（两个char字符）
|  |    0x000055555555529d <+238>:      addl   $0x1,-0xc8(%rbp)
|  |    0x00005555555552a4 <+245>:      addl   $0x2,-0xc4(%rbp)
|  |    #开始处理，先找到第一个字符，看看和上面的字符串长度10的大小，判断有没有处理完
|  ---> 0x00005555555552ab <+252>:      mov    -0xc8(%rbp),%eax
|       0x00005555555552b1 <+258>:      cmp    -0xc0(%rbp),%eax
|------ 0x00005555555552b7 <+264>:      jl     0x55555555525e <main+175>
        0x00005555555552b9 <+266>:      lea    -0x70(%rbp),%rax
        0x00005555555552bd <+270>:      mov    %rax,%rdi
        0x00005555555552c0 <+273>:      callq  0x555555555050 <strlen@plt>
        0x00005555555552c5 <+278>:      mov    %rax,%rdx
        # 算出来的正确的code
        0x00005555555552c8 <+281>:      lea    -0x70(%rbp),%rcx
        # 输入的code
        0x00005555555552cc <+285>:      lea    -0xb0(%rbp),%rax
        0x00005555555552d3 <+292>:      mov    %rcx,%rsi
        0x00005555555552d6 <+295>:      mov    %rax,%rdi
        # 比较
        0x00005555555552d9 <+298>:      callq  0x555555555030 <strncmp@plt>
        0x00005555555552de <+303>:      test   %eax,%eax
   |----0x00005555555552e0 <+305>:      je     0x5555555552fd <main+334>
   |    0x00005555555552e2 <+307>:      lea    0xe43(%rip),%rax        # 0x55555555612c
   |    0x00005555555552e9 <+314>:      mov    %rax,%rdi
   |    0x00005555555552ec <+317>:      mov    $0x0,%eax
   |    0x00005555555552f1 <+322>:      callq  0x555555555070 <printf@plt>
   |    0x00005555555552f6 <+327>:      mov    $0xffffffff,%eax
   |    0x00005555555552fb <+332>:      jmp    0x555555555316 <main+359>
   |    # 这里就是正确的结果，所以我们只需要输入一个字符串和上面常量字符串从hex到字符串转换的结果即可
   |--->0x00005555555552fd <+334>:      lea    0xe3b(%rip),%rax        # 0x55555555613f
        0x0000555555555304 <+341>:      mov    %rax,%rdi
        0x0000555555555307 <+344>:      mov    $0x0,%eax
        0x000055555555530c <+349>:      callq  0x555555555070 <printf@plt>
        0x0000555555555311 <+354>:      mov    $0x0,%eax
        0x0000555555555316 <+359>:      mov    -0x8(%rbp),%rdx
        0x000055555555531a <+363>:      sub    %fs:0x28,%rdx
        0x0000555555555323 <+372>:      je     0x55555555532a <main+379>
        0x0000555555555325 <+374>:      callq  0x555555555060 <__stack_chk_fail@plt>
        0x000055555555532a <+379>:      leaveq
        0x000055555555532b <+380>:      retq
     End of assembler dump.        
```





### 3 fr0zien 的crackme

```assembly
   0x0000555555555165 <+0>:     push   %rbp
   0x0000555555555166 <+1>:     mov    %rsp,%rbp
   0x0000555555555169 <+4>:     sub    $0xa0,%rsp
   # argc
   0x0000555555555170 <+11>:    mov    %edi,-0x94(%rbp)
   # argv, 可以通过argv找到具体的参数
   0x0000555555555176 <+17>:    mov    %rsi,-0xa0(%rbp)
   #stack gard
   0x000055555555517d <+24>:    mov    %fs:0x28,%rax
   0x0000555555555186 <+33>:    mov    %rax,-0x8(%rbp)
   0x000055555555518a <+37>:    xor    %eax,%eax
   # 检查参数是不是两个
   0x000055555555518c <+39>:    cmpl   $0x1,-0x94(%rbp)
   0x0000555555555193 <+46>:    jne    0x5555555551ab <main+70>
   # 如果没有输入参数，那么直接输出提示的字符：Usage: ./crackme FLAG
   0x0000555555555195 <+48>:    lea    0xe68(%rip),%rdi        # 0x555555556004
   0x000055555555519c <+55>:    callq  0x555555555030 <puts@plt>
   0x00005555555551a1 <+60>:    mov    $0x1,%eax
   0x00005555555551a6 <+65>:    jmpq   0x555555555355 <main+496>
   # 比较参数是不是0x15的长度, 下面的语句明显是先找到参数，调用strlen检查字符串的长度
   0x00005555555551ab <+70>:    mov    -0xa0(%rbp),%rax
   0x00005555555551b2 <+77>:    add    $0x8,%rax
   0x00005555555551b6 <+81>:    mov    (%rax),%rax
   0x00005555555551b9 <+84>:    mov    %rax,%rdi
   0x00005555555551bc <+87>:    callq  0x555555555040 <strlen@plt>
   # 比较参数是不是0x15的长度, 0x15才会往下走，否则报错
   0x00005555555551c1 <+92>:    cmp    $0x15,%rax
   0x00005555555551c5 <+96>:    je     0x5555555551dd <main+120>
   0x00005555555551c7 <+98>:    lea    0xe4c(%rip),%rdi        # 0x55555555601a
   0x00005555555551ce <+105>:   callq  0x555555555030 <puts@plt>
   0x00005555555551d3 <+110>:   mov    $0x1,%eax
   0x00005555555551d8 <+115>:   jmpq   0x555555555355 <main+496>
   # 该字符串为sup3r_s3cr3t_k3y_1337
   0x00005555555551dd <+120>:   lea    0xe41(%rip),%rax        # 0x555555556025
   0x00005555555551e4 <+127>:   mov    %rax,-0x88(%rbp)
   # 注意这个地址-0x90(%rbp)从0开始循环
   0x00005555555551eb <+134>:   movl   $0x0,-0x90(%rbp)
    |--0x00005555555551f5 <+144>:       jmp    0x555555555225 <main+192>
----|->  0x00005555555551f7 <+146>:     mov    -0x90(%rbp),%eax <------------
|   |  0x00005555555551fd <+152>:       movslq %eax,%rdx
|   |  0x0000555555555200 <+155>:       mov    -0x88(%rbp),%rax
|   |  0x0000555555555207 <+162>:       add    %rdx,%rax
|   |  0x000055555555520a <+165>:       movzbl (%rax),%eax
|   |  # 刚才上面的字符串对应的ascii每个减去0x22，再存入代码中
|   |  0x000055555555520d <+168>:       sub    $0x22,%eax
|   |  0x0000555555555210 <+171>:       mov    %eax,%edx
|   |  0x0000555555555212 <+173>:       mov    -0x90(%rbp),%eax
|   |  0x0000555555555218 <+179>:       cltq
|   |  0x000055555555521a <+181>:       mov    %dl,-0x20(%rbp,%rax,1)
|   |  0x000055555555521e <+185>:       addl   $0x1,-0x90(%rbp)
|   |  # 这里比较是不是到了0x14的长度，就是上面字符串的长度处理完了没
|   |->0x0000555555555225 <+192>:       cmpl   $0x14,-0x90(%rbp)
|----  0x000055555555522c <+199>:       jle    0x5555555551f7 <main+146> --
   # 这里开始存储另一个ascii码，这里注意是movl，间隔为4,下面会看到具体的比较，记住这些常量，一会会用到
   0x000055555555522e <+201>:   movl   $0x37,-0x80(%rbp)
   0x0000555555555235 <+208>:   movl   $0x3f,-0x7c(%rbp)
   0x000055555555523c <+215>:   movl   $0x2f,-0x78(%rbp)
   0x0000555555555243 <+222>:   movl   $0x76,-0x74(%rbp)
   0x000055555555524a <+229>:   movl   $0x2b,-0x70(%rbp)
   0x0000555555555251 <+236>:   movl   $0x62,-0x6c(%rbp)
   0x0000555555555258 <+243>:   movl   $0x28,-0x68(%rbp)
   0x000055555555525f <+250>:   movl   $0x21,-0x64(%rbp)
   0x0000555555555266 <+257>:   movl   $0x34,-0x60(%rbp)
   0x000055555555526d <+264>:   movl   $0xf,-0x5c(%rbp)
   0x0000555555555274 <+271>:   movl   $0x77,-0x58(%rbp)
   0x000055555555527b <+278>:   movl   $0x62,-0x54(%rbp)
   0x0000555555555282 <+285>:   movl   $0x48,-0x50(%rbp)
   0x0000555555555289 <+292>:   movl   $0x27,-0x4c(%rbp)
   0x0000555555555290 <+299>:   movl   $0x75,-0x48(%rbp)
   0x0000555555555297 <+306>:   movl   $0x8,-0x44(%rbp)
   0x000055555555529e <+313>:   movl   $0x56,-0x40(%rbp)
   0x00005555555552a5 <+320>:   movl   $0x6a,-0x3c(%rbp)
   0x00005555555552ac <+327>:   movl   $0x68,-0x38(%rbp)
   0x00005555555552b3 <+334>:   movl   $0x4e,-0x34(%rbp)
   0x00005555555552ba <+341>:   movl   $0x68,-0x30(%rbp)
   # 这里在此保存了一个0，-0x8c(%rbp)，计数用
   0x00005555555552c1 <+348>:   movl   $0x0,-0x8c(%rbp)
   | --0x00005555555552cb <+358>:       jmp    0x555555555325 <main+448>
   |   # 还是先找到输入的参数对应的字符串
---|-->0x00005555555552cd <+360>:       mov    -0xa0(%rbp),%rax
|  |   0x00005555555552d4 <+367>:       add    $0x8,%rax
|  |   0x00005555555552d8 <+371>:       mov    (%rax),%rdx
|  |   # 我们是一个char一个char比较，所以得加上刚才的计数,存储到eax
|  |   0x00005555555552db <+374>:       mov    -0x8c(%rbp),%eax
|  |   0x00005555555552e1 <+380>:       cltq
|  |   0x00005555555552e3 <+382>:       add    %rdx,%rax
|  |   # 拿到具体对应的char，存储到edx里面
|  |   0x00005555555552e6 <+385>:       movzbl (%rax),%edx
|  |   0x00005555555552e9 <+388>:       mov    -0x8c(%rbp),%eax
|  |   0x00005555555552ef <+394>:       cltq
|  |   # 在拿到刚才上面算出来的常量字符串减去0x22后面存储的结果
|  |   0x00005555555552f1 <+396>:       movzbl -0x20(%rbp,%rax,1),%eax
|  |   # 两者求异或
|  |   0x00005555555552f6 <+401>:       xor    %edx,%eax
|  |   0x00005555555552f8 <+403>:       movsbl %al,%edx
|  |   0x00005555555552fb <+406>:       mov    -0x8c(%rbp),%eax
|  |   0x0000555555555301 <+412>:       cltq
|  |    # 再拿到刚才四个四个存储的常量，进行比较，如果都一致就成功，否则失败
|  |   0x0000555555555303 <+414>:       mov    -0x80(%rbp,%rax,4),%eax
|  |   0x0000555555555307 <+418>:       cmp    %eax,%edx
|  |   0x0000555555555309 <+420>:       je     0x55555555531e <main+441>
|  |   0x000055555555530b <+422>:       lea    0xd08(%rip),%rdi        # 0x55555555601a
|  |   0x0000555555555312 <+429>:       callq  0x555555555030 <puts@plt>
|  |   0x0000555555555317 <+434>:       mov    $0x1,%eax
|  |   0x000055555555531c <+439>:       jmp    0x555555555355 <main+496>
|  |   0x000055555555531e <+441>:       addl   $0x1,-0x8c(%rbp)
|  --> 0x0000555555555325 <+448>:       cmpl   $0x14,-0x8c(%rbp)
|------0x000055555555532c <+455>:       jle    0x5555555552cd <main+360>
   0x000055555555532e <+457>:   mov    -0xa0(%rbp),%rax
   0x0000555555555335 <+464>:   add    $0x8,%rax
   0x0000555555555339 <+468>:   mov    (%rax),%rax
   0x000055555555533c <+471>:   mov    %rax,%rsi
   0x000055555555533f <+474>:   lea    0xcf5(%rip),%rdi        # 0x55555555603b
   0x0000555555555346 <+481>:   mov    $0x0,%eax
   0x000055555555534b <+486>:   callq  0x555555555060 <printf@plt>
   0x0000555555555350 <+491>:   mov    $0x0,%eax
   0x0000555555555355 <+496>:   mov    -0x8(%rbp),%rcx
   0x0000555555555359 <+500>:   sub    %fs:0x28,%rcx
   0x0000555555555362 <+509>:   je     0x555555555369 <main+516>
   0x0000555555555364 <+511>:   callq  0x555555555050 <__stack_chk_fail@plt>
   0x0000555555555369 <+516>:   leaveq
   0x000055555555536a <+517>:   retq

```



```
rax            0x7fffffffe030
0x5555555552b4 <main+255>       mov    %rax,-0x40(%rbp) # 上面的地址放到-0x40(%rbp)里面去

0x5555555552bc <main+263>       movq   $0x1,(%rax) # 放一个64bit的数字，然后开始每个都放
```





### 3 [RainerZimmerman](https://crackmes.one/user/RainerZimmerman)'s Keygen

这个题比较有意思的是它实际上是优化了除法的表示，不过总的来说还是比较简单，掌握了除法优化规则就大概能看明白了

```assembly
# main函数部分dump
Dump of assembler code for function __libc_start_call_main:
=> 0x00007f3aed342d10 <+0>:	push   %rax
   0x00007f3aed342d11 <+1>:	pop    %rax
   0x00007f3aed342d12 <+2>:	sub    $0x98,%rsp
   0x00007f3aed342d19 <+9>:	mov    %rdi,0x8(%rsp)
   0x00007f3aed342d1e <+14>:	lea    0x20(%rsp),%rdi
   0x00007f3aed342d23 <+19>:	mov    %esi,0x14(%rsp)
   0x00007f3aed342d27 <+23>:	mov    %rdx,0x18(%rsp)
   0x00007f3aed342d2c <+28>:	mov    %fs:0x28,%rax
   0x00007f3aed342d35 <+37>:	mov    %rax,0x88(%rsp)
   0x00007f3aed342d3d <+45>:	xor    %eax,%eax
   0x00007f3aed342d3f <+47>:	call   0x7f3aed35b1e0 <_setjmp>
   0x00007f3aed342d44 <+52>:	endbr64 
   0x00007f3aed342d48 <+56>:	test   %eax,%eax
   0x00007f3aed342d4a <+58>:	jne    0x7f3aed342d97 <__libc_start_call_main+135>
   0x00007f3aed342d4c <+60>:	mov    %fs:0x300,%rax
   0x00007f3aed342d55 <+69>:	mov    %rax,0x68(%rsp)
   0x00007f3aed342d5a <+74>:	mov    %fs:0x2f8,%rax
   0x00007f3aed342d63 <+83>:	mov    %rax,0x70(%rsp)
   0x00007f3aed342d68 <+88>:	lea    0x20(%rsp),%rax
   0x00007f3aed342d6d <+93>:	mov    %rax,%fs:0x300
   0x00007f3aed342d76 <+102>:	mov    0x1ef23b(%rip),%rax        # 0x7f3aed531fb8
   0x00007f3aed342d7d <+109>:	mov    0x14(%rsp),%edi
   0x00007f3aed342d81 <+113>:	mov    0x18(%rsp),%rsi
   0x00007f3aed342d86 <+118>:	mov    (%rax),%rdx
   0x00007f3aed342d89 <+121>:	mov    0x8(%rsp),%rax
   0x00007f3aed342d8e <+126>:	call   *%rax                         <======这里就是真正计算key的部分
   0x00007f3aed342d90 <+128>:	mov    %eax,%edi
   0x00007f3aed342d92 <+130>:	call   0x7f3aed35e5f0 <__GI_exit>
   0x00007f3aed342d97 <+135>:	call   0x7f3aed3aa670 <__GI___nptl_deallocate_tsd>
   0x00007f3aed342d9c <+140>:	lock decl 0x1ef505(%rip)        # 0x7f3aed5322a8 <__nptl_nthreads>
   0x00007f3aed342da3 <+147>:	sete   %al
   0x00007f3aed342da6 <+150>:	test   %al,%al
   0x00007f3aed342da8 <+152>:	jne    0x7f3aed342db8 <__libc_start_call_main+168>
   0x00007f3aed342daa <+154>:	mov    $0x3c,%edx
   0x00007f3aed342daf <+159>:	nop
   0x00007f3aed342db0 <+160>:	xor    %edi,%edi
   0x00007f3aed342db2 <+162>:	mov    %edx,%eax
   0x00007f3aed342db4 <+164>:	syscall 
   0x00007f3aed342db6 <+166>:	jmp    0x7f3aed342db0 <__libc_start_call_main+160>
   0x00007f3aed342db8 <+168>:	xor    %edi,%edi
   0x00007f3aed342dba <+170>:	jmp    0x7f3aed342d92 <__libc_start_call_main+130>
   
   # 从上面的0x00007f3aed342d8e <+126>:	call   *%rax 蹦过来到这里，这里实际上是输入keygen的部分
   (gdb) disass $pc,$pc+100
Dump of assembler code from 0x5571d8a48397 to 0x5571d8a483fb:
=> 0x00005571d8a48397:	mov    %edi,-0x34(%rbp)
   0x00005571d8a4839a:	mov    %rsi,-0x40(%rbp)
   0x00005571d8a4839e:	lea    0xc5f(%rip),%rax        # 0x5571d8a49004
   0x00005571d8a483a5:	mov    %rax,%rsi
   0x00005571d8a483a8:	lea    0x2cd1(%rip),%rax        # 0x5571d8a4b080 <_ZSt4cout>
   0x00005571d8a483af:	mov    %rax,%rdi
   0x00005571d8a483b2:	call   0x5571d8a48070 <_ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@plt>
   0x00005571d8a483b7:	lea    -0x30(%rbp),%rax
   0x00005571d8a483bb:	mov    %rax,%rdi
   0x00005571d8a483be:	call   0x5571d8a480a0 <_ZNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEC1Ev@plt>
   0x00005571d8a483c3:	lea    -0x30(%rbp),%rax
   0x00005571d8a483c7:	mov    %rax,%rsi
   0x00005571d8a483ca:	lea    0x2dcf(%rip),%rax        # 0x5571d8a4b1a0 <_ZSt3cin>
   0x00005571d8a483d1:	mov    %rax,%rdi
   0x00005571d8a483d4:	call   0x5571d8a48090 <_ZStrsIcSt11char_traitsIcESaIcEERSt13basic_istreamIT_T0_ES7_RNSt7__cxx1112basic_stringIS4_S5_T1_EE@plt>
   0x00005571d8a483d9:	lea    -0x30(%rbp),%rax
   0x00005571d8a483dd:	mov    %rax,%rdi
   0x00005571d8a483e0:	call   0x5571d8a48030 <_ZNKSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEE5c_strEv@plt>
   0x00005571d8a483e5:	mov    %rax,%rdi
   0x00005571d8a483e8:	call   0x5571d8a48255            <================在这里做真正的keygen的计算
   0x00005571d8a483ed:	test   %al,%al
   0x00005571d8a483ef:	je     0x5571d8a4841e
   0x00005571d8a483f1:	lea    0xc18(%rip),%rax        # 0x5571d8a49010
   0x00005571d8a483f8:	mov    %rax,%rsi
End of assembler dump.
(gdb) 

   #从上面的0x00005571d8a483e8:	call   0x5571d8a48255 蹦过来
   (gdb) disass $pc,$pc+400
Dump of assembler code from 0x5571d8a48255 to 0x5571d8a483e5:
=> 0x00005571d8a48255:	push   %rbp
   0x00005571d8a48256:	mov    %rsp,%rbp
   0x00005571d8a48259:	sub    $0x30,%rsp
   0x00005571d8a4825d:	mov    %rdi,-0x28(%rbp)    # -0x28(%rbp)地址保存的输入的原始字符串
   0x00005571d8a48261:	mov    -0x28(%rbp),%rax
   0x00005571d8a48265:	mov    %rax,%rdi
   0x00005571d8a48268:	call   0x5571d8a48040 <strlen@plt>    # 判断长度是不是0xd，不是就直接返回失败拉
   0x00005571d8a4826d:	mov    %eax,-0x8(%rbp)
   0x00005571d8a48270:	cmpl   $0xd,-0x8(%rbp)
   0x00005571d8a48274:	jne    0x5571d8a48285
   0x00005571d8a48276:	mov    -0x28(%rbp),%rax
   0x00005571d8a4827a:	add    $0x3,%rax
   0x00005571d8a4827e:	movzbl (%rax),%eax
   0x00005571d8a48281:	cmp    $0x2d,%al                     # 比较位置3，也就是第四个字符是不是分隔符“-”
   0x00005571d8a48283:	je     0x5571d8a4828f
   0x00005571d8a48285:	mov    $0x0,%eax
   0x00005571d8a4828a:	jmp    0x5571d8a4838c
   0x00005571d8a4828f:	mov    -0x28(%rbp),%rax
   0x00005571d8a48293:	mov    $0x3,%edx
   0x00005571d8a48298:	mov    $0x0,%esi
   0x00005571d8a4829d:	mov    %rax,%rdi
   0x00005571d8a482a0:	call   0x5571d8a481ee              #这个函数负责计算给定的字符串的数字的求和结果，这里edx是3，所以三位的求和
   0x00005571d8a482a5:	mov    %eax,-0xc(%rbp)             #记住这个地址，这个地址是一直参与运算的
   0x00005571d8a482a8:	cmpl   $0xffffffff,-0xc(%rbp)
   0x00005571d8a482ac:	jne    0x5571d8a482b8
   0x00005571d8a482ae:	mov    $0x0,%eax
   0x00005571d8a482b3:	jmp    0x5571d8a4838c
   0x00005571d8a482b8:	mov    -0xc(%rbp),%edx             # 下面的运算实际上是往-0xc(%rbp)存入 
   0x00005571d8a482bb:	movslq %edx,%rax                   # sum(0,2) - 3*[sum(0,2)* 0x55555556>>32-sum(0,2)>>31]
   0x00005571d8a482be:	imul   $0x55555556,%rax,%rax       # 这个实际上就是sum(0,2) % 3
   0x00005571d8a482c5:	shr    $0x20,%rax
   0x00005571d8a482c9:	mov    %rax,%rcx
   0x00005571d8a482cc:	mov    %edx,%eax
   0x00005571d8a482ce:	sar    $0x1f,%eax
   0x00005571d8a482d1:	sub    %eax,%ecx
   0x00005571d8a482d3:	mov    %ecx,%eax
   0x00005571d8a482d5:	add    %eax,%eax
   0x00005571d8a482d7:	add    %ecx,%eax
   0x00005571d8a482d9:	sub    %eax,%edx
   0x00005571d8a482db:	mov    %edx,-0xc(%rbp)
   0x00005571d8a482de:	movl   $0x4,-0x4(%rbp)
   0x00005571d8a482e5:	jmp    0x5571d8a4837d
   0x00005571d8a482ea:	mov    -0x4(%rbp),%ecx
   0x00005571d8a482ed:	mov    -0x28(%rbp),%rax
   0x00005571d8a482f1:	mov    $0x3,%edx
   0x00005571d8a482f6:	mov    %ecx,%esi
   0x00005571d8a482f8:	mov    %rax,%rdi
   0x00005571d8a482fb:	call   0x5571d8a481ee
   0x00005571d8a48300:	mov    %eax,-0x10(%rbp)             # 后面三位数字的求和，每次根据conter加三位
   0x00005571d8a48303:	cmpl   $0xffffffff,-0x10(%rbp)
   0x00005571d8a48307:	jne    0x5571d8a48310
   0x00005571d8a48309:	mov    $0x0,%eax
   0x00005571d8a4830e:	jmp    0x5571d8a4838c
   0x00005571d8a48310:	mov    -0x4(%rbp),%eax              # 这个就是counter
   0x00005571d8a48313:	sub    $0x4,%eax
   0x00005571d8a48316:	movslq %eax,%rdx
   0x00005571d8a48319:	imul   $0x55555556,%rdx,%rdx        # 从conter拿到具体要计算数字的位置，可以看到这个就直接是
   0x00005571d8a48320:	shr    $0x20,%rdx                   # [(counter-4)* 0x55555556>>32-(counter-4)>>31]，这个是除法，除以3
   0x00005571d8a48324:	sar    $0x1f,%eax
   0x00005571d8a48327:	sub    %eax,%edx
   0x00005571d8a48329:	movslq %edx,%rdx
   0x00005571d8a4832c:	mov    -0x28(%rbp),%rax
   0x00005571d8a48330:	add    %rdx,%rax
   0x00005571d8a48333:	movzbl (%rax),%eax
   0x00005571d8a48336:	movsbl %al,%eax
   0x00005571d8a48339:	mov    %eax,%edi
   0x00005571d8a4833b:	call   0x5571d8a481c9               # 从计算出来的position，拿到对应数字的值
   0x00005571d8a48340:	xor    -0xc(%rbp),%eax
   0x00005571d8a48343:	mov    %eax,-0x14(%rbp)             # 这里注意，存储的是
   0x00005571d8a48346:	mov    -0x10(%rbp),%ecx             # 拿到了刚才的求和
   0x00005571d8a48349:	movslq %ecx,%rax                    # 这个地方计算的值是下面，x就是counter
   0x00005571d8a4834c:	imul   $0x38e38e39,%rax,%rax        # sum(x,x+2) - 9*[sum(x,x+2)* 38e38e39>>33-sum(x,x+2)>>31]
   0x00005571d8a48353:	shr    $0x20,%rax                   # 实际上是sum(x,x+2) % 9
   0x00005571d8a48357:	mov    %eax,%edx
   0x00005571d8a48359:	sar    %edx
   0x00005571d8a4835b:	mov    %ecx,%eax
   0x00005571d8a4835d:	sar    $0x1f,%eax
   0x00005571d8a48360:	sub    %eax,%edx
   0x00005571d8a48362:	mov    %edx,%eax
   0x00005571d8a48364:	shl    $0x3,%eax
   0x00005571d8a48367:	add    %edx,%eax
   0x00005571d8a48369:	sub    %eax,%ecx
   0x00005571d8a4836b:	mov    %ecx,%edx
   0x00005571d8a4836d:	cmp    %edx,-0x14(%rbp)            # 这里基本就清楚了，要求-0xc(%rbp) XOR num[(counter-4)/3] = sum(x,x+2) % 9
   0x00005571d8a48370:	je     0x5571d8a48379
   0x00005571d8a48372:	mov    $0x0,%eax
   0x00005571d8a48377:	jmp    0x5571d8a4838c
   0x00005571d8a48379:	addl   $0x3,-0x4(%rbp)
   0x00005571d8a4837d:	cmpl   $0xc,-0x4(%rbp)
   0x00005571d8a48381:	jle    0x5571d8a482ea
   0x00005571d8a48387:	mov    $0x1,%eax
   0x00005571d8a4838c:	leave  
   0x00005571d8a4838d:	ret    

```



基本上走一趟就明白了，所以写个函数跑了下

```c++
#include <iostream>
#include <vector>

using namespace::std;

int main() {
  int tmp_value = 0;
  for (int i = 0; i <= 9; ++i) {
     for ( int j = 0; j <= 9; ++j) {
        for ( int k = 0; k <= 9; ++k) {
                bool find_456 = false;
                bool find_789 = false;
                bool find_abc = false;
                int tmp_sum_456 = 0;
                int tmp_sum_789 = 0;
                int tmp_sum_abc = 0;
                tmp_value = (i+j +k)%3;
                //tmp_value = (i+j+k)-(3*(i+j+k)*0x55555556>>32)+(3*(i+j+k)>>31);
                //std::cout <<" iterate tmp_value is " << tmp_value << std::endl;
                for ( int sum = 0 ; sum <=27; ++sum) {
                        int final_ans = sum % 9;
                        int last_tmp = tmp_value ^ i;
                        if (final_ans == last_tmp) {
                                find_456=true;
                                tmp_sum_456 = sum;
                        }
                        last_tmp = tmp_value ^ j;
                        if (final_ans == last_tmp) {
                                find_789=true;
                                tmp_sum_789 = sum;
                        }
                        last_tmp = tmp_value ^ k;
                        if (final_ans == last_tmp) {
                                find_abc=true;
                                tmp_sum_abc = sum;
                        }
                        if ((find_abc == true) && (find_456 == true) && (find_789 == true)) {
                                std::cout <<"##########"<<tmp_sum_456 <<" " << tmp_sum_789<<" " << tmp_sum_abc <<std::endl;
                                std::cout << "!!!!!!!!!!!!!"<<i <<" " << j <<" " << k <<std::endl;
                        }

                }

        }
     }
  }
  return 0;
}

```



最后随便输入一个000-000000000，过了。。。。














## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)