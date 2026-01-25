# CMU 15-213 机器级编程要点速查

## 一、基础概念
- **机器代码**：处理器直接执行的二进制指令
- **汇编代码**：机器代码的文本表示（AT&T格式）
- **程序计数器（PC/rip）**：指向下一条指令地址
- **寄存器文件**：16个64位通用寄存器
- **条件码**：CF/ZF/SF/OF 标志位

## 二、x86-64寄存器（重点16个）

| 64位 | 32位 | 16位 | 8位 | 主要用途 |
|------|------|------|-----|----------|
| %rax | %eax | %ax | %al | 返回值 |
| %rbx | %ebx | %bx | %bl | 被调用者保存 |
| %rcx | %ecx | %cx | %cl | 第4个参数 |
| %rdx | %edx | %dx | %dl | 第3个参数 |
| %rsi | %esi | %si | %sil | 第2个参数 |
| %rdi | %edi | %di | %dil | 第1个参数 |
| %rbp | %ebp | %bp | %bpl | 基指针（可选） |
| %rsp | %esp | %sp | %spl | 栈指针 |
| %r8  | %r8d | %r8w | %r8b | 第5个参数 |
| %r9  | %r9d | %r9w | %r9b | 第6个参数 |
| %r10 | %r10d | %r10w | %r10b | 调用者保存 |
| %r11 | %r11d | %r11w | %r11b | 调用者保存 |
| %r12-15 | 对应32位 | 对应16位 | 对应8位 | 被调用者保存 |

## 三、数据格式与内存寻址

### 1. 数据类型大小
```
char     1字节     byte
short    2字节     word
int      4字节     double word
long     8字节     quad word
指针     8字节     quad word
float    4字节     single precision
double   8字节     double precision
```

### 2. 寻址模式
```
立即数寻址:        $0x123
寄存器寻址:        %rax
绝对寻址:          0x1234
间接寻址:          (%rax)
基址+偏移:         8(%rbp)
变址寻址:          (%rax,%rbx,4)  # 地址 = rax + rbx*4
```

## 四、关键指令集

### 1. 数据传输
```asm
movq  src, dst      # 移动（复制）数据
movsbq/movslq       # 符号扩展移动
movzbq/movzlq       # 零扩展移动
pushq src           # 压栈：rsp-=8, [rsp]=src
popq  dst           # 弹栈：dst=[rsp], rsp+=8
```

### 2. 算术运算
```asm
leaq  src, dst      # 加载有效地址（常用于计算）
addq  src, dst      # dst += src
subq  src, dst      # dst -= src
imulq src, dst      # dst *= src（有符号）
xorq  src, dst      # dst ^= src
andq  src, dst      # dst &= src
orq   src, dst      # dst |= src
salq/shlq shft, dst # 左移
sarq      shft, dst # 算术右移（有符号）
shrq      shft, dst # 逻辑右移（无符号）
```

### 3. 控制流指令
```asm
cmpq  src2, src1    # 比较 src1-src2，设置条件码
testq src2, src1    # 测试 src1&src2，设置条件码

jmp   label         # 无条件跳转
je/jz label         # 等于/零时跳转
jne/jnz label       # 不等于/非零时跳转
jg/jnle label       # 有符号大于时跳转
jl/jnge label       # 有符号小于时跳转
ja/jnbe label       # 无符号大于时跳转
jb/jnae label       # 无符号小于时跳转

cmovcc src, dst     # 条件移动（避免分支预测惩罚）
```

### 4. 过程调用
```asm
call  label         # 1. push返回地址 2. jmp到label
ret                 # 1. pop返回地址 2. jmp到该地址
```

## 五、栈帧结构（典型布局）
```
高地址
...                 # 调用者的栈帧
参数7               # (如果存在)
参数6               # 
...                 #
返回地址            # call指令压入
保存的%rbp          # 可选
被调用者保存寄存器  # 如%rbx, %r12-15
局部变量           #
参数构造区         # (如果函数有超过6个参数)
低地址             # ← %rsp指向这里
```

## 六、调用约定（System V AMD64 ABI）

### 1. 参数传递
- **整数/指针参数**：%rdi, %rsi, %rdx, %rcx, %r8, %r9
- **浮点参数**：%xmm0-%xmm7
- **更多参数**：通过栈传递（右→左压栈）

### 2. 返回值
- **整数/指针**：%rax
- **浮点**：%xmm0
- **大结构体**：通过%rdi传递隐藏指针

### 3. 寄存器保存责任
- **调用者保存**：%rax, %rcx, %rdx, %rsi, %rdi, %r8-11, %xmm0-15
- **被调用者保存**：%rbx, %rbp, %r12-15

## 七、条件码与条件执行

### 1. 条件码（EFLAGS）
- **CF**：进位标志（无符号溢出）
- **ZF**：零标志（结果为0）
- **SF**：符号标志（结果为负）
- **OF**：溢出标志（有符号溢出）

### 2. 条件设置
```asm
sete  dst   # 等于时设置字节 = ZF
setne dst   # 不等于时设置 = !ZF
setg  dst   # 有符号大于 = !(SF^OF) & !ZF
setl  dst   # 有符号小于 = SF^OF
seta  dst   # 无符号大于 = !CF & !ZF
setb  dst   # 无符号小于 = CF
```

## 八、数据结构表示

### 1. 数组访问
```c
// T A[N];
// 元素A[i]的地址：A + i*sizeof(T)
// 汇编：leaq (%rdi,%rsi,4), %rax  # int数组，i在%rsi
```

### 2. 结构体对齐
- 基本对齐：K字节类型必须在K的倍数地址
- 结构体对齐：按最大成员类型对齐
- 节省空间：按大小降序排列成员

### 3. 联合体
- 所有成员共享内存
- 大小为最大成员的大小

## 九、缓冲区溢出与防护

### 1. 常见攻击
- **栈破坏**：覆盖返回地址
- **代码注入**：注入并执行恶意代码
- **ROP攻击**：利用现有代码片段（gadgets）

### 2. 防护机制
- **栈随机化（ASLR）**：随机化栈地址
- **栈不可执行**：NX位标记栈为不可执行
- **栈金丝雀**：在返回地址前插入校验值
- **地址空间布局随机化**：随机化代码位置

## 十、浮点运算

### 1. 寄存器
- **XMM寄存器**：%xmm0-%xmm15，128位
- **用于**：标量/向量浮点运算

### 2. 基本操作
```asm
movss src, dst      # 移动单精度浮点
movsd src, dst      # 移动双精度浮点
addss src, dst      # 单精度加法
addsd src, dst      # 双精度加法
mulss, mulsd        # 乘法
subss, subsd        # 减法
divss, divsd        # 除法
ucomiss, ucomisd    # 浮点数比较
```

## 十一、调试与分析工具

### 1. 常用命令
```bash
gcc -Og -S file.c    # 生成汇编文件
gcc -Og -c file.c    # 生成目标文件
objdump -d file.o    # 反汇编
gdb ./a.out          # 调试
    (gdb) x/10xb address   # 检查内存
    (gdb) disas func       # 反汇编函数
    (gdb) break *address   # 在地址设断点
    (gdb) stepi            # 单步执行指令
```

### 2. 理解汇编
- 关注函数调用、参数传递、返回值
- 注意栈指针(%rsp)的变化
- 区分寄存器的调用者/被调用者保存

## 十二、性能优化要点

### 1. 关键概念
- **CPI**：每条指令周期数
- **关键路径**：限制性能的数据依赖链
- **吞吐量界限**：功能单元的最大产能

### 2. 优化策略
- **循环展开**：减少分支开销
- **指令级并行**：利用多个功能单元
- **减少数据依赖**：打破关键路径
- **使用SIMD指令**：向量化计算

---

**记忆技巧**：
1. 记住"前六个参数寄存器"顺序：rdi, rsi, rdx, rcx, r8, r9
2. 栈向低地址增长，push时%rsp减小
3. 条件码设置：比较(cmp)在跳转(jmp)之前
4. 调用约定：调用者保存易失寄存器，被调用者保存持久寄存器

**实践建议**：
- 用`gcc -S -Og`查看简单C代码的汇编
- 用`gdb`单步执行，观察寄存器变化
- 写小段代码，预测其汇编形式，再验证

这份速查涵盖了CMU 15-213机器级编程的核心内容。工作中可以分段学习，每次聚焦一个小主题（如寄存器用途、调用约定等），结合实践加深理解。



