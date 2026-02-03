# 附录：C vs Scheme 速查表

本附录提供C和Scheme的快速对照参考，帮助你快速查找和转换代码。

---

## A.1 基础语法对照

### A.1.1 程序结构

| C | Scheme | 说明 |
|---|--------|------|
| `#include <stdio.h>` | `(import scheme)` | 导入库 |
| `int main() { ... }` | 直接写表达式 | 程序入口 |
| `// 注释` | `; 注释` | 单行注释 |
| `/* 多行注释 */` | `#\| ... \|#` | 多行注释 |
| `#define MACRO` | `(define-macro ...)` | 宏定义 |

### A.1.2 变量声明

| C | Scheme | 说明 |
|---|--------|------|
| `int x = 10;` | `(define x 10)` | 定义变量 |
| `const int x = 10;` | `(define x 10)` | 默认不可变 |
| `int x;` | `(define x)` | 未初始化 |
| `int arr[5];` | `(define arr (make-vector 5))` | 数组/向量 |
| `x = 20;` | `(set! x 20)` | 赋值 |
| `++x; x++;` | `(set! x (+ x 1))` | 自增 |

### A.1.3 作用域

| C | Scheme | 说明 |
|---|--------|------|
| `{ int x; ... }` | `(let ((x ...)) ...)` | 局部作用域 |
| `for(int i=0; ...)` | `(let loop ((i 0)) ...)` | 循环作用域 |
| `static int x;` | `(define x ...)` | 文件/全局作用域 |
| `extern int x;` | 无直接对应 | 外部变量 |

---

## A.2 运算符对照

### A.2.1 算术运算

| C | Scheme | 示例 |
|---|--------|------|
| `+` | `+` | `(+ 1 2)` → 3 |
| `-` | `-` | `(- 5 2)` → 3 |
| `*` | `*` | `(* 3 4)` → 12 |
| `/` | `/` | `(/ 10 2)` → 5 |
| `%` | `modulo` 或 `remainder` | `(modulo 10 3)` → 1 |
| `a += b` | `(set! a (+ a b))` | 复合赋值 |
| `++a` | `(set! a (+ a 1))` | 前置自增 |
| `a++` | `(begin (set! a (+ a 1)) (- a 1))` | 后置自增 |

**注意**：Scheme支持任意精度整数
```scheme
(+ 1 123456789012345678901234567890)  ;; C无法直接表示
```

### A.2.2 比较运算

| C | Scheme | 示例 |
|---|--------|------|
| `==` | `=` (数字) 或 `equal?` (通用) | `(= 3 3)` → #t |
| `!=` | `(not (= ...))` | `(/= 3 4)` (部分实现) |
| `<` | `<` | `(< 3 5)` → #t |
| `>` | `>` | `(> 5 3)` → #t |
| `<=` | `<=` | `(<= 3 5)` → #t |
| `>=` | `>=` | `(>= 5 3)` → #t |

**陷阱**：不要用 `eq?` 比较数字！
```scheme
(eq? 1000 1000)  ;; 可能是 #f（不同对象）
(= 1000 1000)    ;; #t（值相等）
```

### A.2.3 逻辑运算

| C | Scheme | 示例 |
|---|--------|------|
| `&&` | `and` | `(and (> x 0) (< x 10))` |
| `\|\|` | `or` | `(or (= x 0) (= y 0))` |
| `!` | `not` | `(not (> x 0))` |
| `a && b && c` | `(and a b c)` | 多参数 |
| `a \|\| b \|\| c` | `(or a b c)` | 多参数 |
| `!!` | 无直接对应 | 双重否定 |

**短路求值**：
```scheme
(and (> x 0) (/ 10 x))  ;; 如果x<=0，不会求值第二个表达式
```

### A.2.4 位运算

| C | Scheme | 示例 |
|---|--------|------|
| `&` | `bitwise-and` | `(bitwise-and 5 3)` → 1 |
| `\|` | `bitwise-ior` | `(bitwise-ior 5 1)` → 5 |
| `^` | `bitwise-xor` | `(bitwise-xor 5 1)` → 4 |
| `~` | `bitwise-not` | `(bitwise-not 5)` → -6 |
| `<<` | `bitwise-shift-left` | `(bitwise-shift-left 1 3)` → 8 |
| `>>` | `bitwise-shift-right` | `(bitwise-shift-right 8 3)` → 1 |

---

## A.3 流程控制

### A.3.1 条件语句

| C | Scheme | 说明 |
|---|--------|------|
| `if (cond) stmt` | `(if cond then else)` | if必须有else分支 |
| `if/else` | `(if cond then else)` | 两分支 |
| `if/else if/else` | `cond` | 多分支 |
| `switch/case` | `cond` 或 `case` | 分支选择 |
| `? :` | `if` | 三元运算符 |

**示例对比**：

```c
// C的两分支
if (x > 0) {
    return "positive";
} else {
    return "negative";
}
```

```scheme
;; Scheme的两分支
(if (> x 0)
    "positive"
    "negative")
```

```c
// C的多分支
if (x > 0) {
    printf("pos");
} else if (x < 0) {
    printf("neg");
} else {
    printf("zero");
}
```

```scheme
;; Scheme的多分支
(cond
  ((> x 0) (display "pos"))
  ((< x 0) (display "neg"))
  (else (display "zero")))
```

### A.3.2 循环

| C | Scheme | 说明 |
|---|--------|------|
| `for(init; cond; step)` | 命名let或递归 | 没有for循环 |
| `while(cond)` | 递归 | 没有while循环 |
| `do {...} while(cond)` | 特殊递归模式 | 先执行再判断 |
| `break` | 提前返回或额外条件 | 跳出循环 |
| `continue` | 条件跳过 | 跳过本次 |
| `goto` | 不支持 | 无标号跳转 |

**for循环转换示例**：

```c
// C
for (int i = 0; i < 10; i++) {
    printf("%d ", i);
}
```

```scheme
;; Scheme：命名let
(let loop ((i 0))
  (when (< i 10)
    (display i)
    (display " ")
    (loop (+ i 1))))
```

**while循环转换示例**：

```c
// C
while (x < 100) {
    x = x * 2;
}
```

```scheme
;; Scheme：尾递归
(define (double-until x)
  (if (< x 100)
      (double-until (* x 2))
      x))
```

---

## A.4 函数

### A.4.1 函数定义

| C | Scheme | 说明 |
|---|--------|------|
| `int add(int a, int b)` | `(define (add a b))` | 无类型声明 |
| `void func(void)` | `(define (func))` | 无参数 |
| `int arr[]` | `lst` | 列表代替数组 |
| `return value;` | 最后一个表达式 | 自动返回 |
| `;` (空语句) | 无 | 无空函数概念 |

**完整示例**：

```c
// C
int add(int a, int b) {
    return a + b;
}
```

```scheme
;; Scheme
(define (add a b)
  (+ a b))
```

### A.4.2 函数调用

| C | Scheme | 说明 |
|---|--------|------|
| `func(arg1, arg2)` | `(func arg1 arg2)` | 前缀表达式 |
| `func()` | `(func)` | 无参数也加括号 |
| `printf("%d", x)` | `(printf "~a" x)` | 格式化字符串 |

### A.4.3 高阶函数

| C | Scheme | 说明 |
|---|--------|------|
| 函数指针 `int (*fp)(int)` | 直接传递函数 | 函数是一等公民 |
| `qsort(arr, n, sizeof(int), compare)` | `(sort lst compare)` | 传递比较函数 |
| 回调函数 | lambda | 匿名函数 |

**示例**：

```c
// C：函数指针
int add(int a, int b) { return a + b; }
int (*op)(int, int) = add;
int result = op(3, 4);
```

```scheme
;; Scheme：函数作为值
(define (add a b) (+ a b))
(define op add)
(op 3 4)
```

```c
// C：回调
void foreach(int* arr, int n, void (*func)(int)) {
    for (int i = 0; i < n; i++) {
        func(arr[i]);
    }
}
```

```scheme
;; Scheme：高阶函数
(define (for-each func lst)
  (if (null? lst)
      (void)
      (begin
        (func (car lst))
        (for-each func (cdr lst)))))
```

---

## A.5 数据结构

### A.5.1 数组 vs 列表

| 操作 | C数组 | Scheme列表 | 说明 |
|------|-------|-----------|------|
| 创建 | `int a[] = {1,2,3}` | `'(1 2 3)` | 字面量 |
| 访问 | `a[0]` | `(car lst)` | 首元素 |
| 访问 | `a[i]` | `(list-ref lst i)` | 任意位置 |
| 切片 | `&a[1]` | `(cdr lst)` | 剩余部分 |
| 长度 | `sizeof(a)/sizeof(int)` | `(length lst)` | 获取长度 |
| 追加 | 需要新数组 | `(append lst1 lst2)` | 连接 |
| 修改 | `a[0] = 5` | `(set-car! lst 5)` | 修改首元素 |
| 迭代 | `for` 循环 | 递归或`for-each` | 遍历 |

### A.5.2 结构体 vs 记录

**C结构体**：

```c
typedef struct {
    int x;
    int y;
} Point;

Point p = {3, 4};
printf("%d", p.x);
```

**Scheme记录**（使用define-record-type）：

```scheme
(define-record-type point
  (fields x y))

(define p (make-point 3 4))
(point-x p)  ;; 3
```

**Scheme用序对模拟**：

```scheme
(define (make-point x y) (cons x y))
(define (point-x p) (car p))
(define (point-y p) (cdr p))

(define p (make-point 3 4))
(point-x p)  ;; 3
```

### A.5.3 联合/枚举

| C | Scheme | 说明 |
|---|--------|------|
| `enum {RED, BLUE}` | 使用符号 `'(red blue)` | 枚举 |
| `union` | 多种表示 | 类型标签 |

---

## A.6 类型系统

### A.6.1 类型检查函数

```scheme
(number? x)       ; 数字
(integer? x)      ; 整数
(real? x)         ; 实数
(rational? x)     ; 有理数
(complex? x)      ; 复数
(string? x)       ; 字符串
(char? x)         ; 字符
(boolean? x)      ; 布尔
(pair? x)         ; 序对
(list? x)         ; 列表
(null? x)         ; 空列表
(symbol? x)       ; 符号
(vector? x)       ; 向量
(hash-table? x)   ; 哈希表
(procedure? x)    ; 函数
(void? x)         ; 未定义值（部分实现）
```

### A.6.2 类型转换

```scheme
;; 数字转换
(exact->inexact 10)     ; 10.0
(inexact->exact 10.5)   ; 21/2

;; 字符串转换
(string->number "123")  ; 123
(number->string 123)    ; "123"
(string->list "abc")    ; (#\a #\b #\c)
(list->string '(#\a #\b #\c))  ; "abc"

;; 字符转换
(char->integer #\A)     ; 65
(integer->char 65)      ; #\A

;; 符号转换
(string->symbol "hello")  ; 'hello
(symbol->string 'hello)  ; "hello"
```

---

## A.7 字符串操作

| C | Scheme | 说明 |
|---|--------|------|
| `strlen(s)` | `(string-length s)` | 长度 |
| `strcpy(dst, src)` | `(string-copy src)` | 复制 |
| `strcat(s1, s2)` | `(string-append s1 s2)` | 连接 |
| `strcmp(s1, s2)` | `(string=? s1 s2)` | 比较 |
| `strstr(hay, needle)` | `(substring-index ...)` | 查找 |
| `sprintf(buf, "...")` | `(format "...")` | 格式化 |
| `s[i]` | `(string-ref s i)` | 访问字符 |
| `s[i] = c` | `(string-set! s i c)` | 修改字符 |

**示例**：

```c
// C字符串操作
char s[50];
strcpy(s, "Hello");
strcat(s, " World");
printf("%s\n", s);
printf("%d\n", strlen(s));
```

```scheme
;; Scheme字符串操作
(define s "Hello")
(set! s (string-append s " World"))
(display s)
(newline)
(display (string-length s))
```

---

## A.8 内存管理

### A.8.1 手动管理（C） vs 自动管理（Scheme）

| C | Scheme | 说明 |
|---|--------|------|
| `malloc(size)` | 自动分配 | 堆分配 |
| `free(ptr)` | GC自动回收 | 释放 |
| `ptr = &var` | N/A | 取地址 |
| `*ptr` | N/A | 解引用 |
| `ptr->field` | 访问函数 | 访问成员 |
| 内存泄漏 | 不可能（GC） | 常见错误 |

**示例对比**：

```c
// C：手动管理
int* arr = malloc(10 * sizeof(int));
arr[0] = 1;
free(arr);
```

```scheme
;; Scheme：自动管理
(define arr (make-vector 10))
(vector-set! arr 0 1)
;; 自动GC，无需手动释放
```

### A.8.2 可变性 vs 不可变性

| C | Scheme | 说明 |
|---|--------|------|
| 默认可变 | 默认不可变 | 设计理念 |
| `const` | 无直接对应 | 不可变标记 |
| 修改操作 | `set!`, `set-car!`等 | 显式修改（!表示副作用） |

---

## A.9 常用函数库

### A.9.1 列表操作

```scheme
;; 基础操作
(null? lst)           ; 是否为空
(pair? lst)           ; 是否序对
(list? lst)           ; 是否列表
(length lst)          ; 长度
(list-ref lst n)      ; 获取第n个元素（0-index）
(list-tail lst n)     ; 获取第n个之后的列表

;; 构造
(cons x lst)          ; 在头部添加
(list a b c)          ; 创建列表
(make-list n)         ; 创建n个元素的列表
(append lst1 lst2)    ; 连接列表
(reverse lst)         ; 反转

;; 修改（破坏性操作）
(set-car! pair x)     ; 修改car部分
(set-cdr! pair x)     ; 修改cdr部分

;; 高阶函数
(map func lst)        ; 映射
(filter pred lst)     ; 过滤
(fold-left func init lst)  ; 左折叠
(fold-right func init lst) ; 右折叠
(for-each func lst)   ; 遍历（副作用）
```

### A.9.2 数学函数

```scheme
;; 基础数学
(+ x y ...)           ; 加法
(- x y)               ; 减法
(* x y ...)           ; 乘法
(/ x y)               ; 除法
(quotient x y)        ; 整除商
(remainder x y)       ; 余数
(modulo x y)          ; 模
(expt x y)            ; 幂
(sqrt x)              ; 平方根
(abs x)               ; 绝对值
(max x y ...)         ; 最大值
(min x y ...)         ; 最小值

;; 三角函数
(sin x) (cos x) (tan x)
(asin x) (acos x) (atan x)

;; 其他
(log x)               ; 自然对数
(exp x)               ; e^x
(floor x)             ; 向下取整
(ceiling x)           ; 向上取整
(round x)             ; 四舍五入
(truncate x)          ; 截断
(random n)            ; 随机数（0到n）
```

### A.9.3 输入输出

```scheme
;; 输出
(display x)           ; 输出（无换行）
(newline)             ; 换行
(write x)             ; 输出（带引号字符串）
(print x)             ; 输出并换行
(format fmt arg ...)  ; 格式化输出

;; 输入
(read)                ; 读取表达式
(read-char)           ; 读取字符
(read-line)           ; 读取行

;; 文件IO
(call-with-input-file filename proc)
(call-with-output-file filename proc)
(with-input-from-file filename thunk)
(with-output-to-file filename thunk)
(open-input-file filename)
(open-output-file filename)
(close-port port)
```

---

## A.10 模式转换示例

### A.10.1 数组求和

```c
// C版本
int sum(int* arr, int len) {
    int s = 0;
    for (int i = 0; i < len; i++) {
        s += arr[i];
    }
    return s;
}
```

```scheme
;; Scheme版本
(define (sum-list lst)
  (if (null? lst)
      0
      (+ (car lst) (sum-list (cdr lst)))))

;; 或使用fold
(define (sum-list lst)
  (fold-left + 0 lst))
```

### A.10.2 链表反转

```c
// C版本
Node* reverse(Node* head) {
    Node* prev = NULL;
    Node* curr = head;
    while (curr != NULL) {
        Node* next = curr->next;
        curr->next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

```scheme
;; Scheme版本
(define (reverse-list lst)
  (if (null? lst)
      '()
      (append (reverse-list (cdr lst))
              (list (car lst)))))

;; 或使用内置函数
(reverse lst)
```

### A.10.3 快速排序

```c
// C版本
void quicksort(int* arr, int left, int right) {
    if (left >= right) return;
    int pivot = arr[(left + right) / 2];
    int i = left, j = right;
    while (i <= j) {
        while (arr[i] < pivot) i++;
        while (arr[j] > pivot) j--;
        if (i <= j) {
            int temp = arr[i];
            arr[i] = arr[j];
            arr[j] = temp;
            i++; j--;
        }
    }
    quicksort(arr, left, j);
    quicksort(arr, i, right);
}
```

```scheme
;; Scheme版本
(define (quicksort lst)
  (if (null? lst)
      '()
      (let ((pivot (car lst))
            (rest (cdr lst)))
        (append
          (quicksort (filter (lambda (x) (< x pivot)) rest))
          (list pivot)
          (quicksort (filter (lambda (x) (>= x pivot)) rest))))))
```

---

## A.11 调试技巧

### A.11.1 打印调试

```scheme
;; 简单打印
(display "x = ")
(display x)
(newline)

;; 格式化打印
(display (format "x = ~a, y = ~a~%" x y))

;; 调试宏（需要自己定义）
(define-macro (debug expr)
  `(begin
     (display ,(format "~a = " expr))
     (display ,expr)
     (newline)))

(debug (+ 1 2))  ; 输出：(+ 1 2) = 3
```

### A.11.2 追踪函数

```scheme
;; 某些实现支持trace
(trace function-name)
(untrace function-name)
```

### A.11.3 错误处理

```scheme
;; 抛出错误
(error "Something went wrong")
(error "Invalid argument: ~a" arg)

;; 捕获错误（如果实现支持）
(guard (exc
         ((error? exc)
          (display "Error: ")
          (display exc)))
  (risky-operation))
```

---

## A.12 常见陷阱

### A.12.1 括号匹配

```scheme
;; 错误：括号不匹配
(define (fact n)
  (if (= n 0)
      1
      (* n (fact (- n 1))  ; 少一个右括号

;; 正确
(define (fact n)
  (if (= n 0)
      1
      (* n (fact (- n 1)))))  ; 三个右括号
```

### A.12.2 前缀表达式

```scheme
;; 错误
(1 + 2)           ; 不能把数字当函数
(+ (1) (2))       ; 不需要内层括号

;; 正确
(+ 1 2)
```

### A.12.3 引用和求值

```scheme
(define x 10)

x          ; 10（求值）
'x         ; x（符号）
'(1 2 3)   ; (1 2 3)（列表）
(list 1 2 3) ; (1 2 3)（函数调用）
```

### A.12.4 可变性

```scheme
;; 列表默认不可变
(define lst '(1 2 3))
(set-car! lst 10)  ; 错误！字面量列表不可修改

;; 需要创建可变列表
(define lst (mcons 1 (mcons 2 (mcons 3 '()))))
(set-mcar! lst 10)  ; 正确
```

---

## A.13 学习资源

### A.13.1 在线资源

- **Racket官方文档**：https://racket-lang.org/
- **MIT Scheme**：https://www.gnu.org/software/mit-scheme/
- **SICP官方**：https://mitpress.mit.edu/sites/default/files/sicp/index.html
- **SICP视频**：https://www.youtube.com/playlist?list=PLE18841CABEA240D0

### A.13.2 推荐阅读

1. **The Scheme Programming Language** - R. Kent Dybvig
2. **Scheme and the Art of Programming** - George Springer
3. **Simply Scheme** - Brian Harvey

### A.13.3 练习网站

- **Exercism Scheme Track**：https://exercism.org/tracks/scheme
- **Codewars**：https://www.codewars.com/
- **LeetCode**（用Lisp/Scheme解算法题）

---

**返回[目录](./README.md)**
