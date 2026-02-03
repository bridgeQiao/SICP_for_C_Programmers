# 第0章：预备知识 - 从C到Scheme

## 本章目标

在深入SICP之前，先掌握Scheme的基础语法和概念，用你已经熟悉的C语言作为参照。

**阅读完本章后，你将能够**：
- 读懂基本的Scheme代码
- 理解C和Scheme的语法映射关系
- 掌握变量、函数、流程控制的基础用法

---

## 0.1 Hello World对比

### C版本

```c
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
```

### Scheme版本

```scheme
(display "Hello, World!")
(newline)
```

**语法对比**：

| C | Scheme | 说明 |
|---|--------|------|
| `printf("...")` | `(display "...")` | 输出函数 |
| `"\n"` | `(newline)` | 换行 |
| `语句;` | `(表达式)` | 一切都是表达式 |
| `{}` 语句块 | `(expr1 expr2 ...)` | 括号表示序列 |
| `// 注释` | `; 注释` | 注释语法 |

---

## 0.2 变量和作用域

### 0.2.1 变量声明

**C方式**：

```c
// 必须声明类型
int x = 10;
double y = 3.14;
char* s = "hello";

// 修改变量
x = 20;
```

**Scheme方式**：

```scheme
;; 不需要声明类型
(define x 10)
(define y 3.14)
(define s "hello")

;; 修改变量
(set! x 20)
```

**关键区别**：

| C | Scheme | 说明 |
|---|--------|------|
| `int x = 10;` | `(define x 10)` | 定义变量 |
| `const int x = 10;` | `(define x 10)` | Scheme默认不可变 |
| `x = 20;` | `(set! x 20)` | 显式修改操作（!表示副作用） |
| 作用域：`{}` 块级 | 作用域：函数或`let`表达式 | 作用域规则不同 |

### 0.2.2 局部变量

**C的块级作用域**：

```c
int main() {
    int x = 10;          // 外层变量

    {  // 新作用域
        int x = 20;      // 遮蔽外层x
        printf("%d\n", x);  // 20
    }

    printf("%d\n", x);      // 10
}
```

**Scheme的let表达式**：

```scheme
(define x 10)  ;; 全局变量

;; let创建局部作用域
(let ((x 20))  ;; 局部x遮蔽外层x
  (display x))  ;; 20

(display x)    ;; 10
```

**let vs let***（重要！）：

```scheme
;; let：所有绑定同时初始化（不能互相引用）
(let ((x 5)
      (y (+ x 1)))  ;; 错误！这里的x是外层的x，不是上面的5
  (list x y))

;; let*：顺序绑定（可以引用前面的变量）
(let* ((x 5)
       (y (+ x 1)))  ;; 正确！y可以用前面的x
  (list x y))  ;; (5 6)
```

**C的对应**：

```c
// let* 类似于
int x = 5;
int y = x + 1;  // y可以用x的值

// let 类似于（C不支持）
int x = 5;
int y = 外层的x + 1;  // 不是上面的5
```

---

## 0.3 数据类型

### 0.3.1 基本类型

**C的基本类型**：

```c
int i = 42;
double d = 3.14;
char c = 'A';
char* s = "hello";
int arr[] = {1, 2, 3};
```

**Scheme的基本类型**：

```scheme
;; 整数（任意精度！）
(define i 42)
(define big 123456789012345678901234567890)  ;; C无法直接表示

;; 浮点数
(define d 3.14)

;; 字符
(define c #\A)

;; 字符串
(define s "hello")

;; 列表（不需要固定大小）
(define lst (list 1 2 3))
```

**类型检查**：

```c
// C：编译时确定类型
int x = 10;
// x = "hello";  // 编译错误
```

```scheme
;; Scheme：运行时类型检查
(define x 10)
(number? x)    ;; #t（true）
(string? x)    ;; #f（false）

(define s "hello")
(string? s)    ;; #t
(number? s)    ;; #f
```

**类型判断函数**：

| Scheme函数 | 作用 | C对应 |
|-----------|------|-------|
| `(number? x)` | 是否数字 | 无法直接判断（编译时确定） |
| `(integer? x)` | 是否整数 | `x % 1 == 0`（对于浮点数） |
| `(string? x)` | 是否字符串 | 无对应（C字符串就是`char*`） |
| `(pair? x)` | 是否序对 | 无对应（需要自定义结构） |
| `(null? x)` | 是否空列表 | `ptr == NULL` |
| `(eq? x y)` | 是否同一对象 | `x == y`（指针比较） |

### 0.3.2 列表（C的数组+链表）

**C的数组**：

```c
int arr[] = {1, 2, 3, 4, 5};
int len = 5;

// 访问
arr[0];      // 1
arr[4];      // 5

// 遍历
for (int i = 0; i < len; i++) {
    printf("%d ", arr[i]);
}
```

**Scheme的列表**：

```scheme
(define lst '(1 2 3 4 5))  ;; ' 表示"引用"，不求值

;; 访问
(car lst)     ;; 1（第一个元素）
(cadr lst)    ;; 2（第二个元素的简写）
(car (cdr (cdr lst)))  ;; 3（第三个元素）

;; 遍历
(define (print-list lst)
  (if (null? lst)
      (newline)
      (begin
        (display (car lst))
        (display " ")
        (print-list (cdr lst)))))

(print-list lst)  ;; 1 2 3 4 5
```

**列表操作对照表**：

| 操作 | C数组 | Scheme列表 |
|------|-------|-----------|
| 创建 | `{1, 2, 3}` | `'(1 2 3)` 或 `(list 1 2 3)` |
| 访问首元素 | `arr[0]` | `(car lst)` |
| 访问剩余部分 | `&arr[1]` | `(cdr lst)` |
| 添加元素 | `arr[i] = value` | `(cons x lst)` |
| 检查是否为空 | `len == 0` | `(null? lst)` |
| 获取长度 | `sizeof(arr)/sizeof(int)` | `(length lst)` |

**重要概念：cons/car/cdr**

```
列表结构：(1 2 3)

可视化：
[1 | →] → [2 | →] → [3 | /]
   car      car      car
   cdr→     cdr→     cdr→空列表

(car '(1 2 3))  → 1
(cdr '(1 2 3))  → (2 3)
(cons 1 '(2 3)) → (1 2 3)
```

**C的链表对比**：

```c
struct Node {
    int value;
    struct Node* next;
};

// cons
Node* cons(int value, Node* next) {
    Node* node = malloc(sizeof(Node));
    node->value = value;
    node->next = next;
    return node;
}

// car
int car(Node* lst) {
    return lst->value;
}

// cdr
Node* cdr(Node* lst) {
    return lst->next;
}
```

---

## 0.4 流程控制

### 0.4.1 条件语句

**C的if-else**：

```c
if (x > 0) {
    printf("positive\n");
} else if (x < 0) {
    printf("negative\n");
} else {
    printf("zero\n");
}
```

**Scheme的if和cond**：

```scheme
;; 简单if
(if (> x 0)
    (display "positive")
    (display "non-positive"))

;; 多分支用cond
(cond
  ((> x 0) (display "positive"))
  ((< x 0) (display "negative"))
  (else    (display "zero")))
```

**关键区别**：

| C | Scheme | 说明 |
|---|--------|------|
| `if (cond) { ... }` | `(if cond then-expr else-expr)` | if是表达式，有返回值 |
| `else if` | 用`cond`的多个分支 | Scheme没有else if语法 |
| `{}` 块 | `(begin ...)` 或隐式 | if的每个分支只能有一个表达式 |
| `condition ? a : b` | `(if condition a b)` | 三元运算符 |

**if的返回值**：

```c
// C：if是语句，没有返回值
int abs_x;
if (x > 0) {
    abs_x = x;
} else {
    abs_x = -x;
}
```

```scheme
;; Scheme：if是表达式，有返回值
(define abs-x (if (> x 0) x (- x)))

;; 可以直接使用
(display (if (> x 0) "positive" "negative"))
```

**逻辑运算**：

| C | Scheme | 说明 |
|---|--------|------|
| `a && b` | `(and a b)` | 短路与 |
| `a \|\| b` | `(or a b)` | 短路或 |
| `!a` | `(not a)` | 非 |
| `a == b` | `(eq? a b)` 或 `(= a b)` | 相等（数字用=） |
| `a != b` | `(not (= a b))` | 不等 |

### 0.4.2 循环

**C的for循环**：

```c
for (int i = 0; i < 10; i++) {
    printf("%d ", i);
}
```

**Scheme的递归（没有for循环！）**：

```scheme
(define (print-numbers n)
  (if (< n 10)
      (begin
        (display n)
        (display " ")
        (print-numbers (+ n 1)))))

(print-numbers 0)  ;; 0 1 2 3 4 5 6 7 8 9
```

**命名let（模拟循环）**：

```scheme
;; 命名let是Scheme的"循环"
(let loop ((i 0))
  (if (< i 10)
      (begin
        (display i)
        (display " ")
        (loop (+ i 1)))))  ;; "递归"调用
```

**对应C的while**：

```c
int i = 0;
while (i < 10) {
    printf("%d ", i);
    i++;
}
```

**循环模式对照表**：

| C循环 | Scheme递归模式 |
|-------|--------------|
| `for(init; cond; step)` | `(define (loop var) (if cond (begin body (loop new-var))))` |
| `while(cond)` | 同上，但没有init |
| `do {...} while(cond)` | 先执行body，再递归 |
| `break` | 提前返回或用额外条件 |
| `continue` | 跳过本次递归 |

**实际例子：求和**：

```c
// C版本
int sum = 0;
for (int i = 1; i <= 10; i++) {
    sum += i;
}
```

```scheme
;; Scheme版本（尾递归）
(define (sum-range n)
  (let loop ((i 1) (sum 0))
    (if (> i n)
        sum
        (loop (+ i 1) (+ sum i)))))

(sum-range 10)  ;; 55
```

---

## 0.5 函数定义和调用

### 0.5.1 基本函数定义

**C的函数**：

```c
// 声明返回类型和参数类型
int add(int a, int b) {
    return a + b;
}

// 调用
int result = add(3, 4);
```

**Scheme的函数**：

```scheme
;; 定义（不需要类型声明）
(define (add a b)
  (+ a b))

;; 调用
(add 3 4)  ;; 7
```

**语法对比**：

| C | Scheme | 说明 |
|---|--------|------|
| `int add(int a, int b)` | `(define (add a b))` | 定义 |
| `return a + b;` | `(+ a b)` | 最后一个表达式自动返回 |
| `add(3, 4)` | `(add 3 4)` | 调用（前缀表达式） |
| `void func()` | `(define (func))` | 无返回值 |
| `int func()` | `(define (func))` | 有返回值 |

### 0.5.2 多返回值

**C没有多返回值**（需要用指针或结构）：

```c
// 用指针返回多个值
void divide(int a, int b, int* quotient, int* remainder) {
    *quotient = a / b;
    *remainder = a % b;
}

int q, r;
divide(10, 3, &q, &r);
```

**Scheme用列表返回多个值**：

```scheme
(define (divide a b)
  (list (quotient a b) (remainder a b)))

(define result (divide 10 3))
(car result)     ;; 3（商）
(cadr result)    ;; 1（余数）
```

**或者用值解构**：

```scheme
(define (divide a b)
  (values (quotient a b) (remainder a b)))

(define-values (q r) (divide 10 3))
q  ;; 3
r  ;; 1
```

### 0.5.3 可变参数

**C的可变参数**：

```c
#include <stdarg.h>

int sum(int count, ...) {
    va_list args;
    va_start(args, count);

    int total = 0;
    for (int i = 0; i < count; i++) {
        total += va_arg(args, int);
    }

    va_end(args);
    return total;
}

sum(3, 1, 2, 3);  // 6
```

**Scheme的可变参数**：

```scheme
(define (sum . nums)
  (apply + nums))

(sum 1 2 3)  ;; 6
(sum 1 2 3 4 5)  ;; 15
```

---

## 0.6 前缀表达式（重要！）

**C使用中缀表达式**：

```c
1 + 2           // 加法
3 * (4 + 5)     // 优先级用括号
a && b || c     // 运算符优先级
```

**Scheme使用前缀表达式**：

```scheme
(+ 1 2)              ;; 加法
(* 3 (+ 4 5))        ;; 括号明确表示嵌套
(or (and a b) c)     ;; 逻辑运算也是前缀
```

**转换规则**：

```
C:           operator operand1 operand2 ...
Scheme:      (operator operand1 operand2 ...)

例子：
1 + 2         → (+ 1 2)
1 + 2 * 3     → (+ 1 (* 2 3))
(1 + 2) * 3   → (* (+ 1 2) 3)
a && b || c   → (or (and a b) c)
```

**为什么用前缀？**

1. **统一性**：所有函数调用都是 `(func args ...)`
2. **可变参数**：`(+ 1 2 3 4 5)` 不需要特殊语法
3. **宏**：可以定义新的语法形式

**C中类似的前缀**：

```c
// 函数调用是前缀
printf("hello", arg1, arg2);

// 但运算符是中缀
1 + 2;  // 不是 +(1, 2)
```

---

## 0.7 代码块和序列

**C的代码块**：

```c
{
    int x = 1;
    int y = 2;
    printf("%d", x + y);
}
```

**Scheme的begin**：

```scheme
(begin
  (define x 1)
  (define y 2)
  (display (+ x y)))
```

**隐式序列**：

```scheme
;; if的每个分支隐式包含多个表达式
(if (> x 0)
    (begin
      (display "positive")
      (display x)
      (newline))
    (display "non-positive"))

;; cond的每个分支也是隐式序列
(cond
  ((> x 0)
   (display "positive")
   (newline))
  (else
   (display "other")))
```

**begin的返回值**：

```scheme
(begin
  (display "hello")
  (display " ")
  (+ 1 2))  ;; 最后一个表达式的值是begin的返回值
;; 输出：hello ，返回 3
```

---

## 0.8 作用域和变量遮蔽

**C的作用域**：

```c
int x = 10;

void func() {
    int x = 20;  // 遮蔽全局x
    {
        int x = 30;  // 遮蔽外层x
        printf("%d", x);  // 30
    }
    printf("%d", x);  // 20
}
printf("%d", x);  // 10
```

**Scheme的作用域**：

```scheme
(define x 10)

(define (func)
  (let ((x 20))  ;; 遮蔽全局x
    (let ((x 30))  ;; 遮蔽外层x
      (display x))  ;; 30
    (display x)))  ;; 20

(func)
(display x)  ;; 10
```

**动态作用域 vs 词法作用域**（重要概念）：

```c
/* C使用词法作用域（静态作用域）*/
int x = 10;

void func() {
    printf("%d", x);  // 使用定义时的x，即全局x
}

void main() {
    int x = 20;
    func();  // 输出10，不是20
}
```

```scheme
;; Scheme也使用词法作用域
(define x 10)

(define (func)
  (display x))  ;; 使用定义时的x

(let ((x 20))
  (func))  ;; 输出10，不是20
```

---

## 0.9 常见错误和调试

### 0.9.1 括号匹配

**C的问题**：

```c
if (x > 0)
    printf("positive");
    printf("done");  // 缩进欺骗你，这不在if中
```

**Scheme的问题**：

```scheme
(if (> x 0)
    (display "positive")
    (display "negative"))  ;; 忘记右括号

;; 正确
(if (> x 0)
    (display "positive")
    (display "negative"))  ;; 三个右括号匹配三个左括号
```

**括号计数技巧**：

```
从左到右数：
( ( ( ( ) ) ) )
1 2 3 4 3 2 1 0  ← 计数
            ↑ 回到0，匹配完成
```

### 0.9.2 前缀表达式习惯

**常见错误**：

```scheme
;; 错误：受中缀表达式影响
(1 + 2)           ;; 应该是 (+ 1 2)
(+ (1 2))         ;; 不需要内层括号

;; 错误：忘记前缀
x > 0 && y > 0    ;; 应该是 (and (> x 0) (> y 0))
```

### 0.9.3 引用和求值

```scheme
(define x 10)

x        ;; 10（求值）
'x       ;; x（符号本身，不求值）
'(1 2 3) ;; (1 2 3)（列表，不求值内部元素）
(list 1 2 3) ;; (1 2 3)（调用list函数）

;; 错误示例
(define lst (1 2 3))  ;; 错误：试图把1作为函数调用
(define lst '(1 2 3)) ;; 正确
```

---

## 0.10 速查表

### 0.10.1 语法对照

| C | Scheme | 说明 |
|---|--------|------|
| `// 注释` | `; 注释` | 单行注释 |
| `int x = 10;` | `(define x 10)` | 变量定义 |
| `x = 20;` | `(set! x 20)` | 变量赋值 |
| `const int x = 10;` | `(define x 10)` | Scheme默认不可变 |
| `{ }` | `(begin ...)` | 代码块 |
| `if (cond) stmt` | `(if cond then else)` | 条件 |
| `switch` | `cond` | 多分支 |
| `for/while` | 递归/命名let | 循环 |
| `return value;` | 最后一个表达式 | 返回值 |
| `function(args)` | `(function args)` | 函数调用 |

### 0.10.2 运算符对照

| C | Scheme | 说明 |
|---|--------|------|
| `+ - * / %` | `+ - * / modulo` | 算术 |
| `== !=` | `=, equal?` | 相等（数字用=） |
| `< > <= >=` | `< > <= >=` | 比较 |
| `&& \|\| !` | `and or not` | 逻辑 |
| `& \| ^ ~ << >>` | `bit-and bit-or ...` | 位运算（不同实现可能不同） |
| `a ? b : c` | `(if a b c)` | 三元 |

### 0.10.3 类型检查

```scheme
(number? x)      ; 是否数字
(integer? x)     ; 是否整数
(real? x)        ; 是否实数
(string? x)      ; 是否字符串
(char? x)        ; 是否字符
(boolean? x)     ; 是否布尔
(pair? x)        ; 是否序对
(list? x)        ; 是否列表
(null? x)        ; 是否空列表
(symbol? x)      ; 是否符号
(procedure? x)   ; 是否函数
```

---

## 本章小结

### 从C到Scheme的思维转变

1. **前缀表达式**：运算符在前，`(op a b)` 而不是 `a op b`
2. **括号即语法**：括号不是可选项，是语法结构的一部分
3. **一切皆表达式**：if、cond都有返回值
4. **递归代替循环**：没有for/while，用递归或命名let
5. **动态类型**：不需要声明类型，运行时检查
6. **不可变优先**：变量默认不可变，修改用`set!`

### 学习建议

1. **先读代码，再写代码**：熟悉括号匹配
2. **使用编辑器**：支持括号高亮和自动匹配
3. **练习转换**：把简单的C代码转成Scheme
4. **忽略优化**：先理解语义，再考虑性能

---

## 练习

### 基础练习

1. 把以下C代码转成Scheme：
   ```c
   int x = 10;
   if (x > 0) {
       printf("positive");
   } else {
       printf("non-positive");
   }
   ```

2. 用Scheme实现1到100的求和（用递归）

3. 定义一个函数，判断一个数是否是偶数

### 进阶练习

1. 实现一个函数，返回列表的最大值
2. 用命名let实现一个从1数到10的循环
3. 定义一个函数，接受可变参数，返回它们的和

---

**准备好了吗？让我们进入[第1章：函数式编程基础](./chapter_1.md)**
