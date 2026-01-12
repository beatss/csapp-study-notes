**查看`bomb.s`文件有几种方法**：

## 1. **用文本编辑器查看**
```bash
# 使用vim（推荐，支持语法高亮）
vim bomb.s

# 使用nano
nano bomb.s

# 使用gedit（图形界面）
gedit bomb.s
```

## 2. **用命令行工具查看**
```bash
# 查看全部内容（会滚动很快）
cat bomb.s

# 分页查看（推荐）
less bomb.s

# 查看前N行
head -100 bomb.s

# 查看后N行
tail -100 bomb.s
```

## 3. **搜索特定函数**
```bash
# 搜索main函数
grep -n "main" bomb.s

# 搜索phase函数
grep -n "phase_" bomb.s

# 搜索所有函数调用
grep -n "call" bomb.s

# 搜索字符串引用
grep -n "strings" bomb.s
```

## 4. **结合gdb使用（推荐）**
```bash
# 启动gdb
gdb bomb

# 反汇编特定函数
(gdb) disas phase_1

# 反汇编main函数
(gdb) disas main

# 设置断点并运行
(gdb) break phase_1
(gdb) run
```

## 5. **实用技巧**
```bash
# 查看特定函数的汇编代码
grep -A 50 "<phase_1>:" bomb.s

# 配合less搜索
less bomb.s
# 在less中按'/'键，输入phase_1，回车搜索

# 提取某个函数到单独文件
sed -n '/<phase_1>:/,/^$/p' bomb.s > phase1.s

# 统计函数数量
grep -c "^[0-9a-f]* <.*>:$" bomb.s
```

## 6. **建议的工作流程**
1. 先用 `grep -n "phase_" bomb.s` 找到所有phase函数的位置
2. 用 `less bomb.s` 查看整个文件，按G键跳转到末尾查看函数表
3. 重点分析：main函数和phase_1到phase_6
4. 关注：
   - 函数调用（call指令）
   - 字符串比较（常与explode_bomb相关）
   - 输入函数（如scanf, sscanf）
   - 循环和条件跳转

## 提示：
- Bomb Lab的解题关键是理解每个phase的检查逻辑
- 重点关注 `strings_not_equal`、`read_six_numbers` 等函数
- `explode_bomb` 是失败路径，避免走到这里
- 可以在phase函数开始处设置断点，用gdb动态调试


## gdb常用命令
### 启动调试程序，其中带有传入的参数
run <arg1> <arg2> ... ### 简写 r
### 先前如果已经执行过 run <arg1> <arg2> .. 指令后，重新执行时只需直接执行 run 即可。

### 重复上一条指令
按下[Enter]键

### 单步步过
nexti ### 汇编层面，简写 ni
next  ### 源代码层面，简写 n

### 单步步入
stepi ### 汇编层面，简写 si
step  ### 源代码层面，简写 s

### 执行到返回
finish ### 简写 fini

### 强制函数直接返回（几乎不用）
return ### 简写 ret

######### 查看所有设置的端点
info break ### 简写 info b

### 删除端点
del <break num>

### 启动/禁用断点
enable breakpoints
disable breakpoints

### 查看某个寄存器的值
print $rdi ### print 简写 p

### 设置某个寄存器的值
set $rdx = 0x10

### 设置内存上某个地址的值
set {int}0x83040 = 4

### 查看栈上的值
  ### 传入的参数为显示的栈数据个数
stack 20

### 在某个地址上下断点（无参数时默认在$rip处下断）
b *0x400123     ### b 是 break 的简写
b <line_number> ### 在当前源码文件中的第line_number行下断点
b <file_name>:<line_number> ### 在<file_name>源码文件中的第line_number行下断点
### 临时断点（只生效一次
tb *0x400123    ### 全称 temp break
### 条件断点
b *0x400123 if a == 100
### 忽略特定断点cnt次
ignore 断点编号i cnt

### 输出got表相关信息
got

### 输出某个函数在libc上的地址
print system

### 开启/关闭ASLR
aslr off

### 显示当前进程空间内存分布
vmmap

### 显示内存上的值
  ### 格式：
        ### d 按十进制格式显示变量。
        ### u 按十六进制格式显示无符号整型。
        ### o 按八进制格式显示变量。
        ### t 按二进制格式显示变量。
        ### a 按十六进制格式显示变量。
        ### c 按字符格式显示变量。
        ### f 按浮点数格式显示变量。
  ### 大小：
      ### b 一单元1字节
        ### h 一单元2字节
        ### w 一单元4字节
        ### g 一单元8字节
x/3uh 0x54320 ### 以地址0x54320为起始地址，返回3个单元的值，每个单元有两个字节，输出格式为无符号十六进制。

### 切换线程
thread <id> ### 切换至对应ID的线程
info thread ### 查看当前进程中的所有线程信息

### 退出gdb
quit ### 简写 q

### 选择父进程fork/exec后的跟踪对象
set follow-fork-mode parent/child ### fork后跟踪父进程/子进程
set follow-exec-mode new/same     ### 使用开启新 inferior 来跟踪exec程序
set detach-on-fork on/off     ### 是否同时调试fork后的两个程序

### 调试线程时的设置
set non-stop on  ### 设置其他没触发断点的线程继续跑

### 启动Ttext UI
help tui ### 查看tui的相关命令
tui reg general ### 启动TUI时附带寄存器窗口
tui enable   ### 启动TUI
tui disable ### 退出TUI

### 启动layout UI
help layout ### 查看 layout 的相关命令
layout src   ### 显示源代码窗口
layout asm   ### 显示汇编窗口
layout regs  ###显示源代码/汇编和寄存器窗口
layout split ### 显示源代码和汇编窗口
layout next  ### 显示下一个layout
layout prev  ### 显示上一个layout

### 在GDB中直接调用源码中的某个函数
### 这个语句通常对于大型项目中，执行特定类型的Print()函数很有用
call <expr>

### 环境变量相关
show environment [key] ### 显示环境变量
set environment <key>=<val> ... ### 添加环境变量
unset environment [key] ### 清空环境变量

### signal 相关
info signal / info handle ### 查看 signal 设置
  ### operation如下：
  ###   - no stop         
  ###     当被调试的程序收到信号时，GDB不会停住程序的运行，但会打出消息告诉你收到这种信号。
  ###   - stop
  ###     当被调试的程序收到信号时，GDB会停住你的程序。
  ###   - print
  ###     当被调试的程序收到信号时，GDB会显示出一条信息。
  ###   - noprint
  ###     当被调试的程序收到信号时，GDB不会告诉你收到信号的信息。
  ###   - pass / noignore
  ###     当被调试的程序收到信号时，GDB不处理信号。这表示，GDB会把这个信号交给被调试程序会处理。
  ###   - nopass / ignore
  ###     当被调试的程序收到信号时，GDB不会让被调试程序来处理这个信号。
handle <SIG> <operation>