# 第1章：函数式编程基础

## 引言：C程序员的困惑

你写C代码时可能遇到过这些问题：

```c
// 问题1：递归太难理解
int factorial(int n) {
    // 什么时候用递归？会不会栈溢出？
}

// 问题2：函数指针很丑
int (*compare)(const void*, const void*)
// 为什么要这么复杂的语法？

// 问题3：全局变量到处都是
int g_counter = 0;  // 哪些函数会修改它？
int g_state = 0;
```

本章学习：**函数式编程如何解决这些问题**

---

## 1.1 表达式和求值

### C的语句 vs Scheme的表达式

**C风格（语句导向）**：
```c
int x = 5;
int y = 10;
int result = x + y;
printf("%d\n", result);  // 语句，没有返回值
```

**Scheme风格（表达式导向）**：
```scheme
(+ 5 10)           ; 一切都是表达式，都有返回值
; 结果：15

(+ (* 3 5) (- 10 6))
; 结果：19

(if (> x 0)
    x
    (- x))
; if也是表达式，返回值！
```

**为什么这很重要？**

在C中，你必须这样做：
```c
int abs_x;
if (x > 0) {
    abs_x = x;
} else {
    abs_x = -x;
}
return abs_x;
```

在Scheme中：
```scheme
(return (if (> x 0) x (- x)))  ; 假设有return关键字
```

**核心思想**：表达式可以自由组合，语句不能。

---

## 1.2 递归 vs 迭代

### C中的困境

你学递归时，老师可能说过："递归效率低，容易栈溢出，用循环代替。"

**为什么？看这个例子：**

```c
// 递归阶乘
int factorial(int n) {
    if (n == 0) return 1;
    return n * factorial(n - 1);  // ← 问题在这里
}

// 调用 factorial(5) 的过程：
// factorial(5)
//   → 5 * factorial(4)
//     → 5 * 4 * factorial(3)
//       → 5 * 4 * 3 * factorial(2)
//         → 5 * 4 * 3 * 2 * factorial(1)
//           → 5 * 4 * 3 * 2 * 1 * factorial(0)
//             → 5 * 4 * 3 * 2 * 1 * 1
// 栈深度：O(n)，n大就溢出！
```

### 迭代版本

```c
int factorial_iter(int n) {
    int result = 1;
    for (int i = 1; i <= n; i++) {
        result *= i;
    }
    return result;
}
// 栈深度：O(1)，不会溢出
```

**但是，递归真的这么糟糕吗？**

### 尾递归优化

**关键问题**：为什么递归会栈溢出？

因为需要保存"待完成的乘法"：
```c
return n * factorial(n - 1);
//      ^^^^^^^^^^^^^^^^^^^^^ 必须先算出右边的值
//      所以要保存n，等递归返回后再乘
```

**如果重写成这样呢？**

```c
// 尾递归版本（C编译器通常不会优化，但Scheme会）
int factorial_tail(int n, int accumulator) {
    if (n == 0) return accumulator;
    return factorial_tail(n - 1, n * accumulator);
    //  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //  递归调用是最后操作，不需要保存n
}

// 调用：
factorial_tail(5, 1)
// → factorial_tail(4, 5)
//   → factorial_tail(3, 20)
//     → factorial_tail(2, 60)
//       → factorial_tail(1, 120)
//         → factorial_tail(0, 120)
//           → 120
// 栈深度：O(1)！编译器会把它优化成循环
```

**Scheme版本**：

```scheme
(define (factorial n)
  (define (iter counter accumulator)
    (if (> counter n)
        accumulator
        (iter (+ counter 1) (* counter accumulator))))
  (iter 1 1))

;; 这会被编译器优化成类似C的for循环
;; 没有栈溢出风险！
```

**对比总结**：

| 特性 | C递归 | C循环 | Scheme递归 |
|------|-------|-------|------------|
| 栈空间 | O(n) | O(1) | O(1) 如果是尾递归 |
| 可读性 | 较差 | 中等 | 好 |
| 表达力 | 有限 | 有限 | 强大（递归数据结构） |

**实用的建议**：

1. **在C中**：避免深度递归，用循环
2. **理解模式**：学会识别尾递归
3. **思维转变**：递归描述"是什么"，迭代描述"怎么做"

---

## 1.3 高阶函数：C的函数指针 vs 闭包

### 1.3.1 函数作为值

**C的函数指针（你熟悉的）**：

```c
int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }

int compute(int (*op)(int, int), int x, int y) {
    return op(x, y);
}

// 使用：
compute(add, 3, 4);  // 7
compute(mul, 3, 4);  // 12

// 但语法很丑：
int (*ops[2])(int, int) = {add, mul};
```

**Scheme（函数是一等公民）**：

```scheme
(define (add x y) (+ x y))
(define (mul x y) (* x y))

(define (compute op x y)
  (op x y))  ; 简单多了！

;; 函数可以作为值
(define ops (list + - * /))
(list-ref ops 0)  ; 获取 +

;; 动态调用
((list-ref ops 0) 3 4)  ; 7
```

**为什么C的语法这么复杂？**

因为C的设计历史：函数指针是后来加入的，不是核心特性。

### 1.3.2 Lambda表达式（匿名函数）

**C语言（没有lambda）**：

```c
// 必须提前定义函数
int is_positive(int x) {
    return x > 0;
}

int* filter(int* arr, int len, bool (*pred)(int), int* out_len) {
    // ...
}

// 使用：
int result_len;
int* result = filter(arr, len, is_positive, &result_len);
```

**C++11（有lambda）**：

```cpp
auto result = filter(arr, len, [](int x) { return x > 0; });
```

**Scheme（原生支持lambda）**：

```scheme
(define (filter pred lst)
  (cond ((null? lst) '())
        ((pred (car lst))
         (cons (car lst) (filter pred (cdr lst))))
        (else
         (filter pred (cdr lst)))))

;; 使用时定义lambda
(filter (lambda (x) (> x 0))
        '(-1 2 -3 4 -5))
; 结果：(2 4)

;; 不需要命名函数！
```

**优势**：函数定义紧跟使用位置，更易理解。

### 1.3.3 闭包：函数 + 环境

**这是C程序员最不熟悉的概念！**

**问题：C的函数指针无法捕获环境**

```c
int make_adder(int n) {
    // C无法返回一个"记住n的函数"
    // 只能返回函数指针，但无法携带n
    return 0;  // 不可能！
}
```

**C++的闭包**：

```cpp
std::function<int(int)> make_adder(int n) {
    return [n](int x) { return x + n; };  // 捕获n
}

auto add3 = make_adder(3);
add3(10);  // 13
```

**Scheme的闭包（原生支持）**：

```scheme
(define (make-adder n)
  (lambda (x) (+ x n)))

(define add3 (make-adder 3))
(add3 10)  ; 13

(define add10 (make-adder 10))
(add10 5)  ; 15
```

**闭包的本质**：

```
闭包 = 函数代码 + 定义时的环境变量

make-adder(3) 创建：
  代码：(lambda (x) (+ x n))
  环境：n = 3

当调用 (add3 10) 时：
  用 x=10, n=3 执行代码
  返回 13
```

**C无法直接做到这一点！**（除非用GCC扩展）

---

## 1.4 高阶函数的威力

### 1.4.1 map, filter, reduce

这些是函数式编程的"循环抽象"。

**C风格（手写循环）**：

```c
// 对每个元素加1
for (int i = 0; i < len; i++) {
    arr[i] = arr[i] + 1;
}

// 过滤正数
int j = 0;
for (int i = 0; i < len; i++) {
    if (arr[i] > 0) {
        result[j++] = arr[i];
    }
}

// 求和
int sum = 0;
for (int i = 0; i < len; i++) {
    sum += arr[i];
}
```

**Scheme风格（高阶函数）**：

```scheme
;; map：对每个元素应用函数
(map (lambda (x) (+ x 1))
     '(1 2 3 4))
; 结果：(2 3 4 5)

;; filter：保留满足条件的元素
(filter (lambda (x) (> x 0))
        '(-1 2 -3 4 -5))
; 结果：(2 4)

;; reduce：折叠列表
(fold-left + 0 '(1 2 3 4))
; 结果：10
```

**为什么这样更好？**

1. **意图清晰**：一眼看出是"映射"、"过滤"、"折叠"
2. **不易出错**：不需要管理索引、边界
3. **可组合**：可以链式调用
   ```scheme
   (filter
     (lambda (x) (even? x))
     (map
       (lambda (x) (* x x))
       '(1 2 3 4)))
   ; 先平方，再过滤偶数
   ; 结果：(4 16)
   ```

### 1.4.2 捕获通用模式

**C程序员经常重复写类似的代码**：

```c
// 求数组和
int sum(int* arr, int len) {
    int s = 0;
    for (int i = 0; i < len; i++) {
        s += arr[i];
    }
    return s;
}

// 求数组积
int product(int* arr, int len) {
    int p = 1;
    for (int i = 0; i < len; i++) {
        p *= arr[i];
    }
    return p;
}

// 代码结构完全一样！只是操作不同
```

**函数式编程：提取模式**

```scheme
(define (fold op initial lst)
  (if (null? lst)
      initial
      (op (car lst)
          (fold op initial (cdr lst)))))

;; 现在求和、求积变得简单
(define (sum lst)
  (fold + 0 lst))

(define (product lst)
  (fold * 1 lst))

;; 甚至可以定义任意组合操作
(define (any-true lst)
  (fold (lambda (a b) (or a b))
        #f
        (map (lambda (x) (> x 0))
             lst)))
```

---

## 1.5 实战：从C思维到函数式思维

### 例子1：找出数组中的最大值

**C思维**：

```c
int max(int* arr, int len) {
    int m = arr[0];
    for (int i = 1; i < len; i++) {
        if (arr[i] > m) {
            m = arr[i];
        }
    }
    return m;
}
```

**函数式思维**：

```scheme
(define (max lst)
  (fold (lambda (current best)
          (if (> current best)
              current
              best))
        (car lst)
        (cdr lst)))
```

**更简洁（Scheme内置）**：

```scheme
(apply max '(1 5 3 9 2))  ; 9
```

### 例子2：链表处理

**C代码（你写过的）**：

```c
typedef struct Node {
    int value;
    struct Node* next;
} Node;

// 对链表每个元素加1
Node* map_add_one(Node* head) {
    Node* result = NULL;
    Node** tail = &result;
    for (Node* p = head; p != NULL; p = p->next) {
        Node* new_node = malloc(sizeof(Node));
        new_node->value = p->value + 1;
        *tail = new_node;
        tail = &new_node->next;
    }
    *tail = NULL;
    return result;
}
```

**Scheme（多么简单）**：

```scheme
(define (map-add-one lst)
  (map (lambda (x) (+ x 1))
       lst))

;; 或者直接
(map (lambda (x) (+ x 1))
     '(1 2 3 4))
; 结果：(2 3 4 5)
```

**关键区别**：
- C需要手动管理内存、指针
- Scheme的列表是**不可变**的，map返回新列表
- 不可变性 → 更容易推理、更安全

### 例子3：树的结构

**C定义二叉树**：

```c
typedef struct TreeNode {
    int value;
    struct TreeNode* left;
    struct TreeNode* right;
} TreeNode;

// 计算树的节点数
int count_nodes(TreeNode* root) {
    if (root == NULL) return 0;
    return 1 + count_nodes(root->left) + count_nodes(root->right);
}

// 翻转树
TreeNode* mirror(TreeNode* root) {
    if (root == NULL) return NULL;
    TreeNode* new = malloc(sizeof(TreeNode));
    new->value = root->value;
    new->left = mirror(root->right);
    new->right = mirror(root->left);
    return new;
}
```

**Scheme（树就是嵌套列表）**：

```scheme
;; 树：(1 (2 (4) (5)) (3 (6) (7)))

;; 计算节点数
(define (count-nodes tree)
  (if (null? tree)
      0
      (+ 1
         (count-nodes (car tree))
         (count-nodes (cdr tree)))))

;; 翻转树
(define (mirror tree)
  (if (null? tree)
      '()
      (cons (car tree)
            (mirror (cdr tree)))))
```

**模式相同，但Scheme更简洁！**

---

## 1.6 柯里化（Currying）

**这是什么鬼东西？**

简单说：**把多参数函数变成单参数函数的链**。

**C函数**：

```c
int add(int a, int b) {
    return a + b;
}

add(3, 4);  // 7
```

**柯里化版本（伪代码）**：

```c
// 无法在C中直接表达，用C++演示
auto add = [](int a) {
    return [a](int b) {
        return a + b;
    };
};

add(3)(4);  // 7
//      ^^^ 先调用 add(3)，返回一个函数
//              再调用这个函数 with 4
```

**Scheme原生支持**：

```scheme
(define (add a)
  (lambda (b)
    (+ a b)))

((add 3) 4)  ; 7

;; 更简洁
(define (add a b) (+ a b))
;; 可以部分应用
(define add3 (add 3))  ; 如果支持自动柯里化
(add3 4)  ; 7
```

**有什么用？**

```scheme
;; 创建专用函数
(define increment (add 1))
(define double (lambda (x) (* 2 x)))

(map increment '(1 2 3))  ; (2 3 4)
(map double '(1 2 3))     ; (2 4 6)
```

**C中的对应**：

```c
// 必须定义多个函数
int increment(int x) { return x + 1; }
int double(int x) { return x * 2; }
```

---

## 1.7 延迟求值（惰性求值）

**问题：不必要的计算**

```c
// C总是立即求值
int compute_expensive() {
    // 耗时计算...
    return result;
}

int x = compute_expensive();  // 立即执行
if (some_condition) {
    // 如果永远不进入这里，上面的计算白费了
    use(x);
}
```

**Scheme的惰性版本**：

```scheme
(define (delay-computation)
  (delay (expensive-operation)))  ; 不会立即执行

(define x (delay-computation))

;; 只在需要时才计算
(force x)
```

**实际应用：无限流**

```scheme
;; 所有自然数（无限的！）
(define naturals
  (cons-stream 1
               (stream-map +1 naturals)))

;; 取前10个
(stream-take naturals 10)
; (1 2 3 4 5 6 7 8 9 10)

;; 只会计算前10个，后面的不会计算
```

**C无法直接表达无限数据结构！**

---

## 本章小结

### 核心概念

| 概念 | C的局限 | Scheme的优势 |
|------|---------|-------------|
| **表达式** | 语句和表达式分离 | 一切都是表达式 |
| **递归** | 容易栈溢出 | 尾递归优化 |
| **函数** | 函数指针语法复杂 | 函数是一等公民 |
| **闭包** | 需要GCC扩展或C++ | 原生支持 |
| **高阶函数** | 需要手写 | map, filter, fold |
| **不可变数据** | 到处是可变状态 | 默认不可变 |

### 思维转变

**C程序员思维**：
- "怎么一步步解决问题？"
- 关注：循环、索引、状态修改
- 工具：for, while, if

**函数式编程思维**：
- "问题是什么？怎么组合简单的操作？"
- 关注：递归、组合、数据转换
- 工具：map, filter, fold, recursion

### 在C中应用这些思想

即使写C代码，也可以借鉴：

1. **避免全局变量**：用函数参数传递状态
2. **函数要小**：每个函数只做一件事
3. **使用const**：标记不修改的参数
4. **考虑尾递归**：某些情况编译器会优化
5. **函数指针数组**：替代大量if-else

---

## 练习

### 思考题

1. 为什么C不把函数指针做成一等公民？
2. 如果C有lambda，什么场景最有用？
3. 闭包在C中如何模拟？（提示：struct + 函数指针）

### 实践题

1. 用C实现一个简单的map函数
2. 把递归斐波那契改写成尾递归版本
3. 设计一个C的"闭包"结构（包含函数指针+环境）

---

**下一章预告**：数据抽象——如何用C的void*和函数指针实现"对象"
