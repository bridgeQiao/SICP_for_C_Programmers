# 第5章：编译器原理

## 引言：从高级代码到机器码

你写的代码最终如何变成CPU能执行的指令？

```c
int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}

// 这段代码如何变成？
// mov eax, [ebp+8]
// cmp eax, 0
// je .L_base_case
// ...
```

**本章目标**：理解编译器的工作原理

- 寄存器机器模型
- 栈帧和函数调用
- 从解释到编译
- 垃圾回收的实现

---

## 5.1 寄存器机器

### 5.1.1 真实CPU的简化模型

**x86-64寄存器**：

```
通用寄存器:
  RAX, RBX, RCX, RDX, RSI, RDI, R8-R15

专用寄存器:
  RSP - 栈指针
  RBP - 基址指针
  RIP - 指令指针
```

**我们设计的简化寄存器机器**：

```
寄存器:
  val - 存放值
  arg1 - 参数1
  arg2 - 参数2
  proc - 当前过程
  continue - 返回地址

操作:
  assign - 赋值
  test - 比较
  branch - 条件跳转
  goto - 跳转
  save/restore - 栈操作
```

### 5.1.2 控制器语言

**例子：阶乘的迭代版本**

```scheme
(define (factorial n)
  (define (iter product counter)
    (if (> counter n)
        product
        (iter (* product counter)
              (+ counter 1))))
  (iter 1 1))
```

**寄存器机器代码**：

```
(controller
  (assign counter (const 1))
  (assign product (const 1))

 fact-loop
  (test (op >) (reg counter) (reg n))
  (branch (label fact-done))
  (assign
   product
   (op *) (reg product) (reg counter))
  (assign counter (op +) (reg counter) (const 1))
  (goto (label fact-loop))

 fact-done)
```

**对应的C代码**：

```c
int factorial(int n) {
    int product = 1;
    int counter = 1;

fact_loop:
    if (counter > n) goto fact_done;
    product = product * counter;
    counter = counter + 1;
    goto fact_loop;

fact_done:
    return product;
}
```

### 5.1.3 递归的挑战

**递归阶乘**：

```scheme
(define (factorial n)
  (if (= n 0)
      1
      (* n (factorial (- n 1)))))
```

**问题**：需要保存`n`的值，等递归返回后再用

**用栈保存**：

```
(controller
  (assign continue (label fact-done))

 fact-loop
  (test (op =) (reg n) (const 0))
  (branch (label base-case))
  (save continue)
  (save n)
  (assign n (op -) (reg n) (const 1))
  (assign continue (label after-fact))
  (goto (label fact-loop))

 after-fact
  (restore n)
  (restore continue)
  (assign val (op *) (reg n) (reg val))
  (goto (reg continue))

 base-case
  (assign val (const 1))
  (goto (reg continue))

 fact-done)
```

**对应的C代码（显式栈）**：

```c
int factorial(int n) {
    int stack_n[100];  // 手动栈
    int stack_continue[100];
    int sp = 0;

    int val;
    int continue_label = DONE;

fact_loop:
    if (n == 0) goto base_case;

    // 保存状态
    stack_n[sp] = n;
    stack_continue[sp] = continue_label;
    sp++;

    // 递归调用
    n = n - 1;
    continue_label = AFTER_FACT;
    goto fact_loop;

after_fact:
    sp--;
    n = stack_n[sp];
    continue_label = stack_continue[sp];
    val = n * val;
    goto continue_label;

base_case:
    val = 1;
    goto continue_label;

done:
    return val;
}
```

**这就是递归的本质！**

---

## 5.2 栈帧和调用约定

### 5.2.1 函数调用需要什么？

```c
int foo(int a, int b) {
    int x = a + b;
    return bar(x);
}

int bar(int y) {
    return y * 2;
}
```

**调用 foo(3, 4) 时发生什么？**

```
调用前:
1. 保存返回地址
2. 传递参数 (3, 4)
3. 跳转到foo

foo内:
4. 保存旧的基址指针
5. 设置新的基址指针
6. 分配局部变量空间
   ┌──────────────┐
   │ 局部变量 x   │  ← RBP - 4
   ├──────────────┤
   │ 旧的RBP      │  ← RBP
   ├──────────────┤
   │ 返回地址     │  ← RBP + 8
   ├──────────────┤
   │ 参数 b (4)   │  ← RBP + 16
   │ 参数 a (3)   │  ← RBP + 24
   └──────────────┘

调用bar:
7. 设置参数
8. 保存返回地址
9. 跳转到bar
   ... 新的栈帧 ...

bar返回:
10. 恢复返回地址
11. 清理参数

foo返回:
12. 设置返回值
13. 恢复RBP
14. 恢复返回地址
```

### 5.2.2 真实的x86-64调用约定

**System V AMD64 ABI**：

```
参数传递（前6个整数参数）:
  第1个: RDI
  第2个: RSI
  第3个: RDX
  第4个: RCX
  第5个: R8
  第6个: R9
  第7个+: 栈

返回值: RAX

被调用者保存（callee-saved）:
  RBX, RBP, R12-R15

调用者保存（caller-saved）:
  RAX, RCX, RDX, RSI, RDI, R8-R11
```

**例子：编译 factorial(5)**

```c
// C代码
int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);
}
```

**汇编输出（简化）**：

```asm
factorial:
    ; RDI = n

    ; 检查 n == 0
    test    rdi, rdi
    jne     .Lrecurse

    ; 基本情况：返回1
    mov     eax, 1
    ret

.Lrecurse:
    ; 递归情况

    ; 保存需要保持的寄存器
    push    rbx          ; 被调用者保存

    ; 保存当前的n
    mov     rbx, rdi     ; rbx = n

    ; 准备参数 n-1
    lea     edi, [rdi-1] ; edi = n-1

    ; 递归调用
    call    factorial    ; 返回值在RAX

    ; n * factorial(n-1)
    imul    rax, rbx     ; rax = rax * rbx

    ; 恢复并返回
    pop     rbx
    ret
```

**栈帧图**：

```
调用 factorial(5) 时的栈:

高地址
    ↓
┌─────────────────┐
│ 返回地址        │  ← RSP (刚进入factorial)
├─────────────────┤
│ 旧的RBX (5)     │  ← RSP (push rbx后)
├─────────────────┤
│ 返回地址        │  ← RSP (递归调用factorial(4))
├─────────────────┤
│ 旧的RBX (4)     │
├─────────────────┤
│ 返回地址        │  ← RSP (递归调用factorial(3))
├─────────────────┤
│ ...             │
└─────────────────┘
    ↓
低地址
```

**栈深度 = 递归深度！**

这就是为什么深度递归会栈溢出。

### 5.2.3 尾调用优化

**尾递归版本**：

```c
int factorial_tail(int n, int acc) {
    if (n == 0) return acc;
    return factorial_tail(n - 1, n * acc);
}
```

**汇编（启用优化）**：

```asm
factorial_tail:
    ; RDI = n, RSI = acc

    ; 检查 n == 0
    test    rdi, rdi
    je      .Ldone

    ; 尾调用优化：复用当前栈帧
    lea     esi, [rsi + rdi - 1]  ; acc = acc * n
    lea     edi, [rdi - 1]         ; n = n - 1
    jmp     factorial_tail         ; 跳转（不是call！）

.Ldone:
    mov     eax, esi               ; 返回acc
    ret
```

**关键区别**：
- `call` 会压栈（新栈帧）
- `jmp` 不会（复用栈帧）

**结果**：O(1)栈空间！

---

## 5.3 从解释到编译

### 5.3.1 解释 vs 编译

**解释执行**：

```
源代码 → 解释器 → 执行
         每次都解析
```

**编译执行**：

```
源代码 → 编译器 → 机器码 → 执行
         只编译一次   直接执行
```

### 5.3.2 简单的编译器

**输入**：Scheme表达式
**输出**：寄存器机器指令

**例子：编译 (+ 1 2)**

```scheme
;; Scheme代码
(+ 1 2)

;; 编译后的指令
(assign val (const 1))
(assign arg1 (reg val))
(assign val (const 2))
(assign arg2 (reg val))
(assign val (op +) (reg arg1) (reg arg2))
```

**编译框架**：

```c
// 编译器结构
typedef struct {
    Instruction* code;      // 生成的指令
    int stack_need;         // 需要的栈空间
} CompileResult;

// 编译表达式
CompileResult compile(Value* exp) {
    if (is_number(exp)) {
        return compile_number(exp);
    }
    if (is_symbol(exp)) {
        return compile_variable(exp);
    }
    if (is_list(exp)) {
        Value* op = car(exp);
        if (is_symbol(op, "+")) {
            return compile_add(cdr(exp));
        }
        if (is_symbol(op, "if")) {
            return compile_if(exp);
        }
        if (is_symbol(op, "lambda")) {
            return compile_lambda(exp);
        }
        return compile_application(exp);
    }
    error("Cannot compile");
}

// 编译数字
CompileResult compile_number(Value* exp) {
    Instruction* code = malloc(sizeof(Instruction));
    code->opcode = ASSIGN;
    code->dest = "val";
    code->src_type = SRC_CONST;
    code->src.const_value = exp->as.number;

    CompileResult result;
    result.code = code;
    result.stack_need = 0;
    return result;
}
```

### 5.3.3 编译函数调用

**输入**：`(factorial 5)`

```
1. 编译 operator (factorial)
   → 加载 factorial 到 proc

2. 编译 operands (5)
   → 加载 5 到 arg1

3. 生成函数调用指令
   → 设置 continue
   → 保存环境
   → 跳转到 proc
```

**生成的代码**：

```
(assign proc (op lookup-variable) (const factorial))
(assign val (const 5))
(assign arg1 (reg val))
(save continue)
(save env)
(assign continue (label after-call))
(goto (reg proc))

 after-call
(restore env)
(restore continue)
```

### 5.3.4 优化：不必要的保存

**追踪寄存器使用**：

```c
// 哪些寄存器需要保存？
// 原则：只保存"冲突"的寄存器

CompileResult preserving(RegSet regs, CompileResult code1, CompileResult code2) {
    // 如果code2修改了regs中的寄存器
    // 在执行code2之前保存它们
    if (intersects(modified_registers(code2), regs)) {
        return insert_save_restore(code1, code2);
    } else {
        return append(code1, code2);
    }
}
```

**例子**：

```
没有优化：
(save env)
(save proc)
(assign val (const 5))
(assign arg1 (reg val))
(restore proc)
(restore env)
(assign proc (op lookup) (const factorial))

优化后：
(assign val (const 5))
(assign arg1 (reg val))
(assign proc (op lookup) (const factorial))

// 因为 assign val 不修改 proc 和 env
```

---

## 5.4 垃圾回收

### 5.4.1 问题：手动管理的痛苦

**C的内存管理**：

```c
// 创建对象
int* arr = malloc(sizeof(int) * 100);

// 使用
for (int i = 0; i < 100; i++) {
    arr[i] = i;
}

// 记得释放！
free(arr);

// 如果忘记？
// → 内存泄漏

// 如果提前free？
// → 悬空指针

// 如果重复free？
// → 双重释放错误
```

### 5.4.2 引用计数

**最简单的GC**：

```c
typedef struct GCObject {
    int ref_count;
    void (*destructor)(struct GCObject*);
    // ... 数据 ...
} GCObject;

void gc_retain(GCObject* obj) {
    obj->ref_count++;
}

void gc_release(GCObject* obj) {
    obj->ref_count--;
    if (obj->ref_count == 0) {
        if (obj->destructor) {
            obj->destructor(obj);
        }
        free(obj);
    }
}
```

**使用**：

```c
// 创建
GCObject* obj = gc_create(...);

// 使用
gc_retain(obj);
process(obj);
gc_release(obj);

// 销毁
gc_release(obj);  // ref_count变为0，自动释放
```

**问题：循环引用**

```c
typedef struct Node {
    GCObject* gc;
    struct Node* next;
    struct Node* prev;  // 双向链表
} Node;

Node* a = create_node();
Node* b = create_node();
a->next = b;
b->prev = a;  // 循环引用！

gc_release(a);
gc_release(b);
// ref_count都是1，不会释放！
```

### 5.4.3 标记-清除

**两阶段算法**：

```
阶段1：标记
  从根集合（栈、全局变量）开始
  遍历所有可达对象
  标记为"活"

阶段2：清除
  遍历堆中所有对象
  未标记的 → 释放
  已标记的 → 清除标记，下次再用
```

**实现**：

```c
typedef struct GCObject {
    bool marked;
    struct GCObject* next;  // 链接到所有对象
    // ... 数据 ...
} GCObject;

GCObject* gc_all_objects = NULL;

// 分配时自动注册
void* gc_alloc(size_t size) {
    GCObject* obj = malloc(size);
    obj->marked = false;
    obj->next = gc_all_objects;
    gc_all_objects = obj;
    return obj;
}

// 标记阶段
void gc_mark(GCObject* obj) {
    if (obj == NULL || obj->marked) return;

    obj->marked = true;

    // 递归标记引用的对象
    if (is_list(obj)) {
        gc_mark(obj->car);
        gc_mark(obj->cdr);
    }
    if (is_pair(obj)) {
        gc_mark(obj->first);
        gc_mark(obj->second);
    }
    // ... 其他类型
}

void gc_mark_all_roots() {
    // 标记栈上的对象
    for each pointer in stack:
        gc_mark(*pointer);

    // 标记全局变量
    for each global:
        gc_mark(global);
}

// 清除阶段
void gc_sweep() {
    GCObject** p = &gc_all_objects;
    while (*p) {
        if (!(*p)->marked) {
            // 未标记，释放
            GCObject* dead = *p;
            *p = dead->next;
            free(dead);
        } else {
            // 已标记，清除标记
            (*p)->marked = false;
            p = &(*p)->next;
        }
    }
}

// 主GC
void gc_collect() {
    gc_mark_all_roots();
    gc_sweep();
}
```

### 5.4.4 复制收集

**思想**：把堆分成两半，从一半复制到另一半

```
初始：
┌──────────┬──────────┐
│ From     │ To       │
│ 空间     │ 空间     │
│          │ (空)     │
└──────────┴──────────┘

分配：
┌──────────┬──────────┐
│ From     │ To       │
│ A B C D  │ (空)     │
└──────────┴──────────┘

GC后：
┌──────────┬──────────┐
│ From     │ To       │
│ (空)     │ A B C    │
└──────────┴──────────┘
  D被回收

再分配：
┌──────────┬──────────┐
│ From     │ To       │
│ E F      │ A B C    │
└──────────┴──────────┘
```

**优点**：
- 只处理活对象（垃圾自动丢弃）
- 消除碎片

**缺点**：
- 需要复制（慢）
- 浪费一半空间

### 5.4.5 分代收集

**观察**：大多数对象死得早

```
代0（年轻代）：
  新创建的对象
  频繁GC
  复制收集

代1（老年代）：
  存活多次的对象
  偶尔GC
  标记-清除
```

**为什么有效？**

```
典型程序的寿命分布：
  90% 对象在创建后很快死亡
  9% 对象活得中等久
  1% 对象几乎永远活着

分代GC优化：
  主要在年轻代快速GC
  老年代很少触发
```

---

## 5.5 实战：从C到汇编

### 5.5.1 简单函数

**C代码**：

```c
int add(int a, int b) {
    return a + b;
}
```

**编译输出**（`gcc -S -O0`）：

```asm
add:
    push    rbp              ; 保存旧帧指针
    mov     rbp, rsp         ; 设置新帧指针
    mov     dword [rbp-4], edi   ; a
    mov     dword [rbp-8], esi   ; b

    mov     edx, dword [rbp-4]   ; 加载a
    mov     eax, dword [rbp-8]   ; 加载b
    add     eax, edx             ; a + b

    pop     rbp              ; 恢复帧指针
    ret                      ; 返回
```

**优化后**（`gcc -S -O2`）：

```asm
add:
    lea     eax, [rdi+rsi]   ; 直接计算
    ret
```

**栈帧对比**：

```
-O0 (无优化):
┌────────────┐
│ b          │  RBP - 8
├────────────┤
│ a          │  RBP - 4
├────────────┤
│ 旧的RBP    │  RBP
├────────────┤
│ 返回地址   │  RBP + 8
├────────────┤
│ a (RDI)    │
│ b (RSI)    │
└────────────┘

-O2 (优化):
┌────────────┐
│ 旧的RBP    │  RBP（可能省略）
├────────────┤
│ 返回地址   │
├────────────┤
│ a (RDI)    │  直接用寄存器
│ b (RSI)    │
└────────────┘
```

### 5.5.2 内联优化

**C代码**：

```c
static inline int square(int x) {
    return x * x;
}

int sum_of_squares(int a, int b) {
    return square(a) + square(b);
}
```

**不内联**：

```asm
sum_of_squares:
    push    rbp
    mov     rbp, rsp
    ; 调用 square(a)
    mov     edi, dword [rbp-4]   ; a
    call    square
    mov     dword [rbp-12], eax  ; 保存结果
    ; 调用 square(b)
    mov     edi, dword [rbp-8]   ; b
    call    square
    ; 相加
    add     eax, dword [rbp-12]
    pop     rbp
    ret
```

**内联后**：

```asm
sum_of_squares:
    lea     eax, [rdi+rdi]    ; a * a
    mov     edx, eax
    lea     eax, [rsi+rsi]    ; b * b
    add     eax, edx          ; a*a + b*b
    ret
```

**内联的代价**：
- 代码变大
- 可能影响缓存

**内联的好处**：
- 消除调用开销
- 更优化的机会

---

## 本章小结

### 编译器的工作

| 阶段 | 输入 | 输出 | C对应 |
|------|------|------|-------|
| **词法分析** | 源代码 | Token流 | `flex` |
| **语法分析** | Token流 | AST | `bison` |
| **语义分析** | AST | 带类型的AST | 类型检查 |
| **代码生成** | AST | 机器码 | `gcc` |

### 函数调用的本质

```
调用 = 保存状态 + 传递参数 + 跳转

栈帧 = 局部变量 + 保存的寄存器 + 返回地址
```

### 优化技术

1. **尾调用优化**：复用栈帧
2. **内联**：消除调用开销
3. **寄存器分配**：减少内存访问
4. **死代码消除**：删除无用代码

### 垃圾回收

| 算法 | 优点 | 缺点 | 用途 |
|------|------|------|------|
| **引用计数** | 简单 | 循环引用 | 简单脚本 |
| **标记-清除** | 完整 | 暂停时间长 | 通用 |
| **复制收集** | 快 | 浪费空间 | 年轻代 |
| **分代** | 高效 | 复杂 | 现代VM |

### 理解底层的好处

1. **性能优化**：知道代码的代价
2. **调试**：理解栈跟踪
3. **安全**：理解栈溢出、缓冲区溢出
4. **设计**：知道语言特性的实现

---

## 全书总结

### 从C到函数式编程

| 方面 | C思维 | 函数式思维 |
|------|-------|-----------|
| **数据** | 可变、结构 | 不可变、递归 |
| **控制** | 循环、跳转 | 递归、高阶函数 |
| **抽象** | 函数、结构体 | 闭包、模块 |
| **并发** | 锁、线程 | 不变性、消息 |

### 编程语言的本质

```
编程语言 = 语法 + 语义 + 实现

语法：如何写
语义：什么意思
实现：如何工作

理解实现 → 掌握语言
```

### 继续学习

1. **实践**：用C写一个完整的解释器
2. **阅读**：《编译原理》（龙书）
3. **研究**：LLVM、JVM源码

---

## 练习

### 思考题

1. 为什么JIT比解释快，但比AOT慢？
2. 如何实现增量GC？
3. 栈和堆的区别是什么？

### 实践题

1. 扩展第4章的解释器，支持let
2. 实现简单的字节码编译器
3. 用引用计数实现一个简单的GC

---

**恭喜你完成了教程！**

你现在已经理解了：
- 函数式编程的核心思想
- 数据抽象和模块化
- 状态和副作用的本质
- 解释器如何工作
- 编译器如何生成代码

这些知识将伴随你的整个编程生涯。继续探索！
