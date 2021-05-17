 在高级语言中，如Objective-C、C和C++里，操作对象是变量，在ARM汇编里，操作对象是寄存器（Register）、内存和栈（stack，也是一片内存区域）。

ARM处理器中的部分寄存器有特殊用途,如下 所示:

| 寄存器   | 用途                            |
| ----- | ----------------------------- |
| R0-R3 | 传递参数和返回值                      |
| R7    | 帧指针，指向母函数于被调用函数在栈中的交界         |
| R9    | 在iOS 3.0以前被系统保留               |
| R12   | 内部过程调用寄存器，dynamic linker 会用到它 |
| R13   | SP寄存器                         |
| R14   | LR寄存器，保存函数返回地址               |
| R15   | PC寄存器                         |

 处理器中名为“program counter”(简称PC)的寄 存器用于存放下一条指令的地址。一般情况下,计算 机一条接一条地顺序执行指令,处理器执行完一条指令后将PC加1,让它指向下一条指令。

分支的 条件一般有4种:

- 操作结果为0(或不为0);
- 操作结果为负数; 
- 操作结果有进位; 
- 运算溢出(比如两个正数相加得到的数超过了 寄存器位数)。

这些条件的判断准则(flag)存放在程序状态寄 存器(Program Status Register,PSR)中,数据处理 相关指令会改变这些flag,分支指令再根据这些flag决 定是否跳转。

### 数据操作指令

数据操作指令有以下2条规则:

1)所有操作数均为32bit;

2)所有结果均为32bit,且只能存放在寄存器

数据操作指令的基本格式是

```armasm
; “cond”的作 用是指定指令“op”在什么条件下执行
; “s”的作用是指定指令“op”是否设置flag
op {cond}{s} Rd, Rn, Op2
```

“cond”,共有下面17种 条件

| 条件   | 含义                                     |
| ---- | -------------------------------------- |
| EQ   | 结果为0（EQual to 0）                       |
| NE   | 结果不为0(Not Equal to 0)                  |
| CS   | 有进位或借位(Carry Set)                      |
| HS   | 同CS(unsigned Higher or Same)           |
| CC   | 没有进位或借位(Carry clear)                   |
| LO   | 同CC(unsigned LOwer)                    |
| MI   | 结果小于0(MInus)                           |
| PL   | 结果大于等于0(PLus)                          |
| VS   | 溢出(oVerflow Set)                       |
| VC   | 无溢出(oVerflow Clear)                    |
| HI   | 无符号比较大于(unsigned HIgher)               |
| LS   | 无符号比较小于等于(unsigned Lower or Same)      |
| GE   | 有符号比较大于等于(signed Greater than orEqual) |
| LT   | 有符号比较小于(signed Less Than)              |
| GT   | 有符号比较大于(signed Greater Than)           |
| LE   | 无符号比较小于等于(signed Less than or Equal)   |
| AL   | 无条件(ALways,默认)                         |

关键字：

有符号是Greater  Less equal

无符号是Higher  Lower  same  （特殊表示）

进位 carry

溢出 overflow



```armasm
; cond 用法 比较R0和R1的值,如果R0大于等于R1,则 R2=R0;否则R2=R1。
comp R0, R1 
mv GE R2, R0 
mv LT R2, R1
```

“s”的作用是指定指令“op”是否设置flag,共有下 面4种flag:

| flag        | 设置方式                                     |
| ----------- | ---------------------------------------- |
| N(Negative) | 如果结果小于0则置1,否则置0;                         |
| Z(Zero)     | 如果结果是0则置1,否则置0;                          |
| C(Carry)    | 对于加操作(包括CMN)来说,如果产生进位则置1,否则置0;对于减操作(包 括CMP)来说,Carry相当于Not-Borrow,如果产生借位则置0,否则置1;对 于有移位操作的非加/减操作来说,C置移出值的最后一位;对于其他的非加/减 操作来说,C的值一般不变;（无符号） |
| V(oVerflow) | 如果操作导致溢出,则置1,否则置0。（有符号）                  |

算术操作

```armasm
ADD R0, R1, R2   ; R0 = R1 + R2
ADC R0, R1, R2   ; R0 = R1 + R2 + C(arry)
SUB R0, R1, R2   ; R0 = R1 - R2
SBC R0, R1, R2   ; R0 = R1 - R2 - !C
RSB R0, R1, R2   ; R0 = R2 - R1   RSB是“Reverse SuB”的缩写
RSC R0, R1, R2   ; R0 = R2 - R1 - !C
```

逻辑操作

```armasm
AND R0, R1, R2   ; R0 = R1 & R2
ORR R0, R1, R2   ; R0 = R1 | R2
EOR R0, R1, R2   ; R0 = R1 ^ R2 Exclusive or
BIC R0, R1, R2   ; R0 = R1 &~ R2
MOV R0, R2       ; R0 = R2
MVN R0, R2       ; R0 = ~R2
```
逻辑移位 和算术移位

算术移位和逻辑移位最大的区别，是算术移位在右移时不改变原来的数的符号而逻辑移位在右移时有可能改变原来的数的符号

[逻辑移位和算术移位有什么区别?](https://edwardluo.wordpress.com/2015/04/08/%E9%80%BB%E8%BE%91%E7%A7%BB%E4%BD%8D%E5%92%8C%E7%AE%97%E6%9C%AF%E7%A7%BB%E4%BD%8D%E6%9C%89%E4%BB%80%E4%B9%88%E5%8C%BA%E5%88%AB%EF%BC%9F/)

```armasm
LSL   ; logic shift left
LSR   ; logic shift right
ASR   ; arithmetic shift right
ROR   ; Recursion Shift right
```

比较操作

```armasm
CMP R1,R2 ; 执行R1 - R2，如果产生借位则c=0，否则c=1  (Compare)
CMN R1,R2 ; 执行R1 + R2，如果产生进位c=1，否则c=0 (Compare not)
TST R1,R2 ; 执行R1 & R2, 如果结果为零，则z=1，否则z=0 (Test bit) 用于测试某一位的值，示例：0110 0010 & 0000 0010 这里结果为1，表示R1的第二位为1
TEQ R1,R2 ; 执行R1 ^ R2, 如果结果为零，则z=1，否则z=0(Test Equal),用于测示两个值是否相等，示例：0110 0010 & 0110 0010 ，这里结果为0 ，表示R1=R2
```

乘法操作

```armasm
MUL R4,R3,R2 ; R4=R3*R2  (multiply)
MLA R4,R3,R2,R1 ; R4=R3*R2+R1 (multiply and add）
```

### 内存操作指令

内存操作指令的基本格式是

```armasm
; Rn是基地址寄存器，用于存放基地址；cond :指定指令op在什么条件下执行
; type 指定op 操作的数据类型 B(Unsigned Byte) SB(Signed Byte) H(unsigned Halfword) SH,默认word
op {cond}{type} Rd,[Rn,Op2]
```

ARM内存操作基础指令只有两个：

```armasm
; LDR (Load Register) 将数据从内存存到寄存器
LDR Rt，[Rn，{，#offset}] ; Rt = *(Rn{+offset}) ,{}代表可选 []表示取值，而不是取地址
LDR Rt，[Rn,#offset]!     ; Rt = *(Rn + offset); Rn = Rn +offset  ! 代表是可写回Rn
LDR Rt，[Rn]，#offset     ; Rt = *Rn; Rn = Rn +offset

; STR (Store Register) 将数据从寄存器中读出来，存在内存中
STR Rt，[Rn，{，#offset}] ; *(Rn{+offset}) = Rt ,{}代表可选 []表示取值，而不是取地址
STR Rt，[Rn,#offset]!     ; *(Rn + offset) = Rt; Rn = Rn +offset
STR Rt，[Rn]，#offset     ; *Rn = Rt; Rn = Rn +offset
```

两个变种:操作双字节

```armasm
; LDRD (Load Register double) 
LDRD R4，R5，[R9，#offset] ; R4 = *(R9+offset) ,R5= *(R9+offset+4)

; STRD
STRD R4，R5，[R9，#offset] ; *(R9+offset)=R4 ,*(R9+offset+4) = R5 
```

