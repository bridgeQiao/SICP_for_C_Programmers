# 第2章：数据抽象

## 引言：C中的数据表示困境

你用C时可能遇到过这些问题：

```c
// 问题1：到处暴露内部结构
typedef struct {
    int numerator;
    int denominator;
} Rational;

Rational r = {1, 2};
printf("%d/%d\n", r.numerator, r.denominator);
// 用户必须知道内部是numerator和denominator
// 如果改成用double存储，所有用户代码都要改！

// 问题2：void*的噩梦
void* data = create_something();
// 这个data是什么？怎么用？
// 必须查文档，或者...

int* int_data = (int*)data;  // 假设它是int
// 假设错了？崩溃！

// 问题3：多态很难实现
void print(void* data, Type type) {
    switch(type) {
        case INT: printf("%d", *(int*)data); break;
        case STRING: printf("%s", (char*)data); break;
        // 每加新类型都要修改这个函数！
    }
}
```

**数据抽象解决这些问题**：
- 隐藏内部表示
- 提供清晰的接口
- 支持多种表示

---

## 2.1 抽象屏障

### 2.1.1 理想情况：有理数

**C的"直接暴露"方式**：

```c
// 用户代码直接访问结构成员
typedef struct {
    int num;
    int den;
} Rational;

Rational r = {1, 2};
Rational s = {1, 3};

// 加法：用户必须知道内部
Rational sum;
sum.num = r.num * s.den + s.num * r.den;
sum.den = r.den * s.den;

// 问题：如果想优化（如化简），所有用户代码都要改
```

**更好的C方式：函数接口**：

```c
// rational.h - 只暴露接口
typedef struct Rational* Rational;  // 不透明类型

Rational make_rational(int num, int den);
int numerator(Rational r);
int denominator(Rational r);
Rational add_rational(Rational a, Rational b);

// rational.c - 隐藏实现
struct Rational {
    int num;
    int den;
};

Rational make_rational(int num, int den) {
    Rational r = malloc(sizeof(struct Rational));
    r->num = num;
    r->den = den;
    return r;
}

// 内部可以随意改变表示，用户代码不受影响
int numerator(Rational r) {
    return r->num;
}
```

**Scheme的数据抽象**：

```scheme
;; 构造器
(define (make-rat n d)
  (let ((g (gcd n d)))
    (cons (/ n g) (/ d g))))  ; 自动化简

;; 选择器
(define (numer r) (car r))
(define (denom r) (cdr r))

;; 操作
(define (add-rat a b)
  (make-rat (+ (* (numer a) (denom b))
               (* (numer b) (denom a)))
            (* (denom a) (denom b))))

;; 使用
(define r (make-rat 1 2))
(define s (make-rat 1 3))
(add-rat r s)  ; 结果自动是最简分数
```

**关键区别**：

| 特性 | C的struct | Scheme的抽象 |
|------|----------|-------------|
| 封装 | 需要手动设计（不透明指针） | 语言级支持（cons/car/cdr） |
| 改变表示 | 需要重新编译 | 用户代码不变 |
| 内存管理 | 手动malloc/free | 自动GC |

### 2.1.2 抽象屏障分层

```
┌─────────────────────────────────────┐
│ 用户代码                              │
│ (add-rat r s)                       │
├─────────────────────────────────────┤ ← 屏障1：操作
│ make-rat, numer, denom              │
├─────────────────────────────────────┤ ← 屏障2：表示
│ cons, car, cdr                      │
├─────────────────────────────────────┤ ← 屏障3：底层
│ 内存管理、指针                       │
└─────────────────────────────────────┘
```

**每个屏障内部可以随便改，不影响上层**

---

## 2.2 层次数据结构

### 2.2.1 序列（链表）

**C的实现（你熟悉的）**：

```c
typedef struct Node {
    int value;
    struct Node* next;
} Node;

// 创建：很麻烦
Node* create_list(int* values, int len) {
    Node head = {0, NULL};
    Node* tail = &head;
    for (int i = 0; i < len; i++) {
        Node* new = malloc(sizeof(Node));
        new->value = values[i];
        new->next = NULL;
        tail->next = new;
        tail = new;
    }
    return head.next;
}

// 操作：很容易出错
int length(Node* head) {
    int len = 0;
    while (head != NULL) {
        len++;
        head = head->next;  // 别忘了这个！
    }
    return len;
}
```

**Scheme的列表**：

```scheme
;; 创建：简单
(define lst '(1 2 3 4))

;; 等价于
(define lst (cons 1 (cons 2 (cons 3 (cons 4 '())))))

;; 操作：不会出错
(length lst)  ; 4
(car lst)     ; 1
(cdr lst)     ; (2 3 4)
```

**为什么Scheme更简单？**

1. **内置列表类型**：不需要手动定义结构
2. **自动内存管理**：不需要malloc/free
3. **模式匹配**：用car/cdr解构

### 2.2.2 树结构

**C的二叉树**：

```c
typedef struct TreeNode {
    int value;
    struct TreeNode* left;
    struct TreeNode* right;
} TreeNode;

// 创建：很多malloc
TreeNode* make_tree(int val, TreeNode* left, TreeNode* right) {
    TreeNode* node = malloc(sizeof(TreeNode));
    node->value = val;
    node->left = left;
    node->right = right;
    return node;
}

// 计算：递归（小心内存泄漏！）
int sum_tree(TreeNode* root) {
    if (root == NULL) return 0;
    return root->value
         + sum_tree(root->left)
         + sum_tree(root->right);
}
```

**Scheme的树（就是嵌套列表）**：

```scheme
;; 树：(1 (2 () ()) (3 () ()))
;; 简化表示：(1 (2) (3))

(define tree '(1 (2) (3 (4) ())))

;; 递归操作很自然
(define (tree-sum tree)
  (if (null? tree)
      0
      (+ (car tree)
         (tree-sum (cadr tree))
         (tree-sum (caddr tree)))))
```

**闭包性质**：

```
数据结构可以包含自身

树：节点包含子树（也是树）
列表：元素可以是列表

这就是"递归数据结构"
```

### 2.2.3 映射树结构

**问题：遍历并对每个节点应用操作**

**C代码（写两次循环）**：

```c
// 遍历列表
void map_list(Node* head, void (*func)(int)) {
    for (Node* p = head; p != NULL; p = p->next) {
        func(p->value);
    }
}

// 遍历树（又要写递归）
void map_tree(TreeNode* root, void (*func)(int)) {
    if (root == NULL) return;
    func(root->value);
    map_tree(root->left, func);
    map_tree(root->right, func);
}
```

**Scheme（统一的map）**：

```scheme
;; 列表map
(map square '(1 2 3))  ; (1 4 9)

;; 树map（需要自己定义）
(define (map-tree func tree)
  (if (null? tree)
      '()
      (list (func (car tree))
            (map-tree func (cadr tree))
            (map-tree func (caddr tree)))))

(map-tree square '(1 (2) (3)))
; (1 (4) (9))
```

---

## 2.3 符号数据

**C的问题：只能处理数字**

```c
// 无法直接表达符号
// 只能用字符串或枚举

enum Symbol { ADD, SUB, MUL, DIV };

struct Expression {
    enum Symbol op;
    struct Expression* left;
    struct Expression* right;
};

// 很繁琐！
```

**Scheme的符号**：

```scheme
;; 符号就是标识符
'different   ; 符号 different
(define x 'symbol)
x  ; symbol

;; 符号求值
(define x 10)
x  ; 10 (求值后的值)

;; 引用阻止求值
'x  ; x (符号本身)
```

**应用：符号求导**

```scheme
;; 表达式：x + 3
;; 表示为：'(+ x 3)

;; 求导规则
(define (deriv exp var)
  (cond ((number? exp) 0)
        ((variable? exp)
         (if (same-variable? exp var) 1 0))
        ((sum? exp)
         (make-sum (deriv (addend exp) var)
                   (deriv (augend exp) var)))
        ((product? exp)
         (make-sum
          (make-product (multiplier exp)
                        (deriv (multiplicand exp) var))
          (make-product (deriv (multiplier exp) var)
                        (multiplicand exp))))))

;; 使用
(deriv '(+ x 3) 'x)     ; (+ 1 0)
(deriv '(* x y) 'x)     ; (+ (* x 0) (* y 1))
```

**C需要多少代码才能做到？**

---

## 2.4 多重表示

### 2.4.1 问题：同一数据，不同表示

**复数的两种表示**：

```c
// 方式1：直角坐标
typedef struct {
    double real;
    double imag;
} RectComplex;

// 方式2：极坐标
typedef struct {
    double mag;
    double angle;
} PolarComplex;

// 现在需要一个统一的接口...
typedef enum { RECT, POLAR } ComplexType;

typedef struct {
    ComplexType type;
    union {
        RectComplex rect;
        PolarComplex polar;
    } data;
} Complex;

// 操作变得复杂
double real_part(Complex c) {
    switch(c.type) {
        case RECT: return c.data.rect.real;
        case POLAR: return c.data.polar.mag * cos(c.data.polar.angle);
    }
}

// 每加新表示，所有函数都要修改！
```

**Scheme的数据导向编程**：

```scheme
;; 标签类型
(define (attach-tag type-tag contents)
  (cons type-tag contents))

(define (type-tag datum)
  (if (pair? datum)
      (car datum)
      (error "Bad tagged datum")))

(define (contents datum)
  (if (pair? datum)
      (cdr datum)
      (error "Bad tagged datum")))

;; 直角坐标包
(define (install-rectangular-package)
  ;; 内部函数
  (define (real-part z) (car z))
  (define (imag-part z) (cdr z))
  (define (make-from-real-imag x y) (cons x y))
  (define (magnitude z)
    (sqrt (+ (square (real-part z))
             (square (imag-part z)))))
  (define (angle z)
    (atan (imag-part z) (real-part z)))
  (define (make-from-mag-ang r a)
    (cons (* r (cos a)) (* r (sin a))))

  ;; 接口：注册到全局表
  (define (tag x) (attach-tag 'rectangular x))
  (put 'real-part '(rectangular) real-part)
  (put 'imag-part '(rectangular) imag-part)
  (put 'magnitude '(rectangular) magnitude)
  (put 'angle '(rectangular) angle)
  (put 'make-from-real-imag 'rectangular
       (lambda (x y) (tag (make-from-real-imag x y))))
  (put 'make-from-mag-ang 'rectangular
       (lambda (r a) (tag (make-from-mag-ang r a))))
  'done)

;; 极坐标包（类似）

;; 通用操作：自动查找
(define (real-part z)
  (apply-generic 'real-part z))

;; 使用
(define z (make-from-real-imag 3 4))
(real-part z)  ; 自动调用rectangular包
```

**核心思想**：

```
操作名 + 类型标签 → 具体实现

(type-table 'real-part 'rectangular) → rect-real-part函数
(type-table 'real-part 'polar) → polar-real-part函数

添加新类型：只需注册新包，不改通用代码
```

**C中的类似模式**：

```c
// 虚函数表（C++的方式）
struct ComplexOps {
    double (*real_part)(void*);
    double (*imag_part)(void*);
    double (*magnitude)(void*);
};

struct RectComplex {
    struct ComplexOps* ops;
    double real, imag;
};

struct PolarComplex {
    struct ComplexOps* ops;
    double mag, angle;
};

// 调用
double real_part(void* complex) {
    struct ComplexOps* ops = ((struct RectComplex*)complex)->ops;
    return ops->real_part(complex);
}
```

---

## 2.5 实战：用C实现数据抽象

### 2.5.1 不透明指针模式

```c
// stack.h - 用户接口
#ifndef STACK_H
#define STACK_H

typedef struct Stack Stack;  // 不完整类型

Stack* stack_create(void);
void stack_push(Stack* s, int value);
int stack_pop(Stack* s);
int stack_empty(Stack* s);
void stack_destroy(Stack* s);

#endif
```

```c
// stack.c - 实现
#include "stack.h"
#include <stdlib.h>

struct Stack {  // 完整定义只在.c中
    int* data;
    int size;
    int capacity;
};

Stack* stack_create(void) {
    Stack* s = malloc(sizeof(Stack));
    s->data = malloc(10 * sizeof(int));
    s->size = 0;
    s->capacity = 10;
    return s;
}

void stack_push(Stack* s, int value) {
    if (s->size >= s->capacity) {
        s->capacity *= 2;
        s->data = realloc(s->data, s->capacity * sizeof(int));
    }
    s->data[s->size++] = value;
}

// 内部可以随意改，用户代码不受影响
```

### 2.5.2 多态容器（void*模式）

```c
// generic_list.h
typedef struct List List;

List* list_create(void);
void list_append(List* l, void* data);
void* list_get(List* l, int index);
void list_foreach(List* l, void (*func)(void*));

// 使用示例
void print_int(void* data) {
    printf("%d ", *(int*)data);
}

int main() {
    List* l = list_create();

    int a = 1, b = 2, c = 3;
    list_append(l, &a);
    list_append(l, &b);
    list_append(l, &c);

    list_foreach(l, print_int);  // 1 2 3

    return 0;
}
```

**问题**：类型不安全！

```c
float f = 3.14;
list_append(l, &f);  // 编译器不报错，但运行会错
list_foreach(l, print_int);  // 崩溃！
```

**Scheme的优势**：类型安全 + 多态

```scheme
;; 列表可以是任何类型
(define lst '(1 2 3))
(define lst2 '("a" "b" "c"))
(define lst3 '(1 "two" 'three))

;; 操作自动适配
(map square lst)    ; (1 4 9)
(map string-length lst2)  ; (1 1 1)
```

---

## 2.6 模式匹配

**现代语言（Rust, Haskell）支持模式匹配**：

```rust
match value {
    None => 0,
    Some(x) => x,
    Tree(left, val, right) => val,
}
```

**Scheme的等价物**：

```scheme
(cond ((null? lst) 'empty)
      ((eq? (car lst) 'special) 'special-case)
      (else (process (cdr lst))))
```

**C的switch无法匹配结构**：

```c
// 无法做到！
switch(tree) {
    case NULL: ...
    case Node(NULL, value, NULL): ...
    case Node(left, value, right): ...
}
```

---

## 本章小结

### 数据抽象的关键

| 概念 | C的方式 | Scheme的优势 |
|------|---------|-------------|
| **封装** | 不透明指针（手动） | 语言级cons/car/cdr |
| **多态** | void*（不安全） | 类型安全的多态 |
| **层次数据** | 手动管理指针 | 递归定义 |
| **符号** | 枚举或字符串 | 原生符号类型 |
| **多重表示** | 虚函数表 | 数据导向编程 |

### 设计原则

1. **隐藏表示**：用户不应该知道内部结构
2. **提供操作**：通过函数访问数据
3. **分层设计**：每层只依赖下一层的接口
4. **支持扩展**：加新类型不需要改旧代码

### 在C中应用

1. **使用不透明指针**：隐藏内部结构
2. **函数指针表**：实现多态
3. **const正确性**：标记只读操作
4. **文档约定**：明确哪些是内部接口

---

## 练习

### 思考题

1. C++的类如何解决C的数据抽象问题？
2. 为什么C不提供内置的列表和字典？
3. void*的多态有什么风险？

### 实践题

1. 用C实现一个不透明的哈希表
2. 设计一个可以存储不同类型的容器（类型安全）
3. 实现一个简单的"虚拟函数表"系统

---

**下一章预告**：状态和副作用——为什么全局变量是魔鬼，如何用闭包管理状态
