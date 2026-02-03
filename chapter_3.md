# 第3章：状态和副作用

> **阅读前提示**：如果你还不熟悉Scheme的基本语法（变量、函数、流程控制），请先阅读[第0章：预备知识](./chapter_0.md)

## 引言：C程序员最头疼的问题

你调试时肯定遇到过：

```c
// 问题1：全局变量的副作用
int g_counter = 0;

int foo() {
    g_counter++;  // 偷偷修改全局变量
    return g_counter * 2;
}

int bar() {
    g_counter += 10;  // 又修改了
    return g_counter;
}

// foo()和bar()的返回值取决于调用顺序！
int x = foo();  // 2
int y = bar();  // 13
int z = foo();  // 28！不是2！

// 问题2：不知道函数是否会修改参数
void process(int* arr, int len) {
    // 会修改arr吗？必须看源码才知道！
    for (int i = 0; i < len; i++) {
        arr[i] *= 2;  // 哦，原来会修改
    }
}

// 问题3：多线程噩梦
int balance = 100;

void deposit(int amount) {
    balance += amount;  // 竞态条件！
}

// 两个线程同时调用：
// Thread1: 读取balance (100)
// Thread2: 读取balance (100)
// Thread1: 写入110
// Thread2: 写入110
// 应该是120，实际是110！
```

**这些问题来自"可变状态"**

---

## 3.1 为什么可变状态这么麻烦？

### 3.1.1 引用透明性丢失

**理想情况**（函数式）：

```scheme
;; 同样的输入永远产生同样的输出
(define (square x) (* x x))

(square 5)  ; 25
(square 5)  ; 25（永远）
(square 5)  ; 25

;; 可以安全替换
(+ (square 5) (square 5))
;; = (+ 25 25)
;; = 50
```

**有状态的情况**：

```c
int counter = 0;

int next() {
    return counter++;
}

next();  // 0
next();  // 1（不一样了！）
next();  // 2

// 不能替换
next() + next()
// 可能是 0 + 1 = 1
// 也可能是 1 + 2 = 3
// 取决于求值顺序！
```

**引用透明性**：表达式可以随时用它的值替换，不改变程序行为。

有状态 → 引用透明性丢失 → 难以推理

### 3.1.2 时间变成重要因素

**函数式**：`f(x) = x²` 永远成立

**有状态**：`balance` 的值取决于"什么时候看"

```c
balance = 100;
deposit(50);   // 现在 balance = 150
withdraw(20);  // 现在 balance = 130
// balance的值依赖于历史操作
```

**这就是"时间"被引入程序**

```
无状态：函数是数学关系， timeless
有状态：变量随时间变化， time-sensitive
```

---

## 3.2 局部状态：用set!引入状态

### 3.2.1 基本的赋值

**C的局部变量**：

```c
int factorial(int n) {
    int result = 1;  // 局部状态
    for (int i = 1; i <= n; i++) {
        result *= i;  // 修改状态
    }
    return result;
}
```

**Scheme的set!**：

```scheme
(define (factorial n)
  (let ((result 1)
        (i 1))
    (define (iter)
      (if (> i n)
          result
          (begin
            (set! result (* result i))  ; 修改变量
            (set! i (+ i 1))
            (iter))))
    (iter)))

;; 但这不是函数式风格！
```

**函数式版本（无状态）**：

```scheme
(define (factorial n)
  (define (iter i acc)
    (if (> i n)
        acc
        (iter (+ i 1) (* acc i))))
  (iter 1 1))

;; 没有set!，通过参数传递"状态"
```

### 3.2.2 银行账户例子

**C的面向对象风格**：

```c
typedef struct {
    double balance;
} BankAccount;

BankAccount* create_account(double initial) {
    BankAccount* acc = malloc(sizeof(BankAccount));
    acc->balance = initial;
    return acc;
}

void deposit(BankAccount* acc, double amount) {
    acc->balance += amount;
}

double get_balance(BankAccount* acc) {
    return acc->balance;
}

// 使用
BankAccount* my_acc = create_account(100.0);
deposit(my_acc, 50.0);
double b = get_balance(my_acc);  // 150.0
```

**Scheme的闭包风格**：

```scheme
(define (make-account balance)
  (define (withdraw amount)
    (if (>= balance amount)
        (begin (set! balance (- balance amount))
               balance)
        "Insufficient funds"))

  (define (deposit amount)
    (set! balance (+ balance amount))
    balance)

  (define (dispatch m)
    (cond ((eq? m 'withdraw) withdraw)
          ((eq? m 'deposit) deposit)
          (else (error "Unknown request"))))

  dispatch)

;; 使用
(define acc (make-account 100))

((acc 'withdraw) 50)   ; 50
((acc 'withdraw) 60)   ; "Insufficient funds"
((acc 'deposit) 40)    ; 90
```

**关键区别**：

| C | Scheme |
|---|---|
| struct + 函数指针 | 闭包 |
| 显式this指针 | 隐式的环境 |
| 手动内存管理 | 自动垃圾回收 |
| 状态在struct字段 | 状态在闭包变量 |

**Scheme的实现细节**：

```
make-account 创建：
  闭包1: {代码: withdraw, 环境: balance = 100}
  闭包2: {代码: deposit, 环境: balance = 100}
  dispatch函数：返回对应闭包

调用 (acc 'withdraw)：
  返回 withdraw 闭包
  调用 ((acc 'withdraw) 50)：
    在 balance=100 的环境中执行 withdraw
    set! 修改这个环境的 balance
    返回 50
```

---

## 3.3 闭包的本质：函数+环境

### 3.3.1 C无法直接表达闭包

**问题：创建"记住"值的函数**

```c
// 想要这样的函数
Function make_adder(int n) {
    // 返回一个"记住n的函数"
    // 无法实现！
}
```

**C++的Lambda（接近了）**：

```cpp
#include <functional>

std::function<int(int)> make_adder(int n) {
    return [n](int x) { return x + n; };  // 捕获n
}

auto add3 = make_adder(3);
add3(10);  // 13
```

**Scheme的原生闭包**：

```scheme
(define (make-adder n)
  (lambda (x) (+ x n)))

(define add3 (make-adder 3))
(add3 10)  ; 13
```

### 3.3.2 闭包实现原理（伪C）

```c
// 闭包 = 函数指针 + 环境
typedef struct Closure {
    void (*func)(struct Closure*, int);
    int env;  // 捕获的变量
} Closure;

// make-adder "编译"后的函数
void adder_func(Closure* closure, int x) {
    int n = closure->env;  // 从环境读取
    return x + n;
}

// make-adder 本身
Closure* make_adder(int n) {
    Closure* closure = malloc(sizeof(Closure));
    closure->func = adder_func;
    closure->env = n;  // 保存到环境
    return closure;
}

// 使用
Closure* add3 = make_adder(3);
int result = add3->func(add3, 10);  // 13
```

**这就是C编译器实现lambda的方式！**

---

## 3.4 可变数据结构

### 3.4.1 C的可变链表

```c
typedef struct Node {
    int value;
    struct Node* next;
} Node;

// 修改链表
void insert_after(Node* node, int value) {
    Node* new = malloc(sizeof(Node));
    new->value = value;
    new->next = node->next;
    node->next = new;  // 修改原节点
}

// 问题：多个指针可能指向同一节点
Node* a = malloc(sizeof(Node));
Node* b = a;  // b和a指向同一节点
insert_after(a, 10);  // b也看到了变化！
```

### 3.4.2 Scheme的可变操作

```scheme
;; set-car! 和 set-cdr! 修改cons对
(define x (list 1 2 3))
(define y x)          ; y和x指向同一列表

(set-car! x 10)       ; 修改x的第一个元素

x  ; (10 2 3)
y  ; (10 2 3)  - y也变了！

;; 这和C指针的行为一样！
```

**共享导致的别名问题**：

```c
// C
int* a = malloc(sizeof(int));
int* b = a;
*a = 10;
printf("%d", *b);  // 10（b看到了a的修改）

// Scheme
(define cell (cons 1 '()))
(define alias cell)
(set-car! cell 10)
(car alias)  ; 10
```

### 3.4.3 队列的实现

**C的循环缓冲队列**：

```c
#define QUEUE_SIZE 100

typedef struct {
    int data[QUEUE_SIZE];
    int head, tail;
    int count;
} Queue;

void enqueue(Queue* q, int value) {
    if (q->count >= QUEUE_SIZE) {
        // 队列满
        return;
    }
    q->data[q->tail] = value;
    q->tail = (q->tail + 1) % QUEUE_SIZE;
    q->count++;
}

int dequeue(Queue* q) {
    if (q->count == 0) {
        // 队列空
        return -1;
    }
    int value = q->data[q->head];
    q->head = (q->head + 1) % QUEUE_SIZE;
    q->count--;
    return value;
}
```

**Scheme的队列（用cons）**：

```scheme
(define (make-queue)
  (cons '() '()))  ; (front-pair . rear-pair)

(define (front-ptr queue) (car queue))
(define (rear-ptr queue) (cdr queue))

(define (set-front-ptr! queue item)
  (set-car! queue item))

(define (set-rear-ptr! queue item)
  (set-cdr! queue item))

(define (empty-queue? queue)
  (null? (front-ptr queue)))

(define (insert-queue! queue item)
  (let ((new-pair (cons item '())))
    (cond ((empty-queue? queue)
           (set-front-ptr! queue new-pair)
           (set-rear-ptr! queue new-pair)
           queue)
          (else
           (set-cdr! (rear-ptr queue) new-pair)
           (set-rear-ptr! queue new-pair)
           queue))))

(define (delete-queue! queue)
  (cond ((empty-queue? queue)
         (error "DELETE! called with empty queue"))
        (else
         (set-front-ptr! queue (cdr (front-ptr queue)))
         queue)))
```

**复杂度相当，但Scheme更灵活**

---

## 3.5 并发：竞态条件和锁

### 3.5.1 问题的本质

**C的竞态条件**：

```c
int account = 100;

// 线程1和线程2同时调用
void deposit(int amount) {
    int temp = account;     // 读取：100
    temp += amount;         // 计算：100 + 50 = 150
    account = temp;         // 写入：150
}

// 可能的交错：
// T1: read account (100)
// T2: read account (100)
// T1: write 150
// T2: write 150
// 结果：150（应该是200！）
```

### 3.5.2 互斥锁

**C的pthread_mutex**：

```c
#include <pthread.h>

int account = 100;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void deposit(int amount) {
    pthread_mutex_lock(&lock);      // 加锁
    int temp = account;
    temp += amount;
    account = temp;
    pthread_mutex_unlock(&lock);    // 解锁
}

// 现在安全了：
// T1: lock (成功)
// T1: read/write
// T2: lock (等待...)
// T1: unlock
// T2: lock (成功)
// T2: read/write
// T2: unlock
```

**Scheme的串行化（概念相同）**：

```scheme
(define (make-account balance)
  (define (withdraw amount)
    (if (>= balance amount)
        (begin (set! balance (- balance amount))
               balance)
        "Insufficient funds"))

  (define (deposit amount)
    (set! balance (+ balance amount))
    balance)

  (define (dispatch m)
    (let ((serializer (make-serializer)))  ; 创建锁
      (cond ((eq? m 'withdraw)
             (serializer withdraw))  ; 串行化操作
            ((eq? m 'deposit)
             (serializer deposit))
            (else (error "Unknown request")))))
  dispatch)
```

### 3.5.3 死锁

**问题**：

```c
pthread_mutex_t lock1, lock2;

// 线程1
pthread_mutex_lock(&lock1);
pthread_mutex_lock(&lock2);
// ...
pthread_mutex_unlock(&lock2);
pthread_mutex_unlock(&lock1);

// 线程2（反向顺序）
pthread_mutex_lock(&lock2);  // 可能和线程1同时执行
pthread_mutex_lock(&lock1);  // 死锁！
```

**死锁条件**：
1. 互斥：资源不能共享
2. 持有并等待：持有锁的同时等待其他锁
3. 不可抢占：锁不能被强制释放
4. 循环等待：形成等待环

**解决方案**：统一加锁顺序

```c
// 总是按固定顺序加锁
pthread_mutex_lock(&lock1);
pthread_mutex_lock(&lock2);
```

---

## 3.6 惰性求值：延迟计算

### 3.6.1 问题：不必要的计算

**C总是立即求值**：

```c
int expensive_computation() {
    // 耗时操作...
    sleep(10);
    return 42;
}

int x = expensive_computation();  // 立即执行
if (some_condition) {
    use(x);
}
// 如果some_condition为false，10秒白费了
```

### 3.6.2 Scheme的delay和force

```scheme
(define (lazy-computation)
  (delay (expensive-operation)))

(define promise (lazy-computation))  ; 不会立即执行

;; 只在需要时才计算
(force promise)  ; 现在才执行

;; 再次force不会重复计算
(force promise)  ; 直接返回缓存的结果
```

**实现原理（伪代码）**：

```scheme
(define (delay exp)
  (let ((evaluated? #f)
        (value #f))
    (lambda ()
      (if (not evaluated?)
          (begin
            (set! value exp)
            (set! evaluated? #t)))
      value)))

(define (force promise)
  (promise))
```

### 3.6.3 无限流

**C无法表示无限数据**：

```c
// 想要"所有自然数"
// 必须限制大小
int naturals[1000];
for (int i = 0; i < 1000; i++) {
    naturals[i] = i + 1;
}
```

**Scheme的无限流**：

```scheme
;; 所有自然数（无限！）
(define naturals
  (cons-stream 1
               (stream-map +1 naturals)))

;; 只在访问时才计算
(stream-ref naturals 100)  ; 101
(stream-ref naturals 1000) ; 1001
;; 其他元素永远不计算

;; 流操作
(stream-take naturals 10)
; (1 2 3 4 5 6 7 8 9 10)

(stream-filter
  (lambda (x) (even? x))
  naturals)
; (2 4 6 8 10 ...) - 无限的偶数流
```

**威力**：

```scheme
;; 所有素数（无限）
(define primes
  (sieve (integers-from 2)))

;; 第100个素数
(stream-ref primes 99)

;; 小于1000的所有素数
(stream-take-while primes (lambda (p) (< p 1000)))
```

---

## 3.7 实战：状态管理的最佳实践

### 3.7.1 最小化可变状态

**不好**（全局变量）：

```c
int counter = 0;

int next_id() {
    return counter++;
}
```

**更好**（通过参数传递）：

```c
int next_id(int* counter) {
    return (*counter)++;
}

// 调用者控制counter
int main() {
    int my_counter = 0;
    int id1 = next_id(&my_counter);
    int id2 = next_id(&my_counter);
}
```

**最好**（完全无状态）：

```scheme
(define (next-ids n)
  (iota n))  ; (0 1 2 3 ... n-1)

;; 调用者管理"状态"
(define ids (next-ids 10))
```

### 3.7.2 不可变数据结构

**C的问题**：

```c
char* str = strdup("hello");
char* p = str;  // p和str指向同一内存
p[0] = 'H';     // 修改了p，str也变了！
printf("%s", str);  // "Hello"
```

**思路：每次返回新副本**：

```c
char* to_upper(char* s) {
    char* new = strdup(s);  // 复制
    for (int i = 0; new[i]; i++) {
        new[i] = toupper(new[i]);
    }
    return new;  // 返回新字符串
}

char* original = "hello";
char* upper = to_upper(original);
// original: "hello" (不变)
// upper: "HELLO" (新字符串)
```

**Scheme默认不可变**：

```scheme
(define original '(1 2 3))
(define modified (map (lambda (x) (* x 2)) original))

original   ; (1 2 3) - 不变
modified   ; (2 4 6) - 新列表
```

### 3.7.3 const正确性

**C的const（不够用）**：

```c
// 承诺不修改数据
void print_string(const char* s) {
    printf("%s", s);
}

// 但const可以"骗"
const char* s = "hello";
char* p = (char*)s;
p[0] = 'H';  // 未定义行为！
```

**更好的设计**：

```c
// 输入总是const
void process(const char* input);

// 输出是非const
char* transform(const char* input);

// 输入+输出分开
void transform_in_place(char* data);  // 明确会修改
```

---

## 本章小结

### 状态的两面性

| 优点 | 缺点 |
|------|------|
| 直观（模拟现实世界） | 难以推理 |
| 高效（原地修改） | 并发问题 |
| 必要（I/O、GUI） | 时间依赖 |

### 设计原则

1. **默认不可变**：除非必要，不要修改数据
2. **局部化状态**：不要用全局变量
3. **明确接口**：const标记只读操作
4. **小心并发**：用锁保护共享状态

### C vs Scheme

| 特性 | C | Scheme |
|------|---|--------|
| 可变性 | 默认可变 | 默认不可变 |
| 状态管理 | 全局/局部变量 | 闭包 |
| 并发 | pthread_mutex | serializer |
| 惰性求值 | 手动实现 | delay/force |

### 在C中应用

1. **减少全局变量**：用函数参数传递
2. **使用const**：标记不会修改的参数
3. **不可变API**：返回新对象而不是修改原对象
4. **文档约定**：明确哪些函数有副作用

---

## 练习

### 思考题

1. 为什么没有全局变量的程序更容易调试？
2. 什么时候必须使用可变状态？
3. 惰性求值如何影响程序性能？

### 实践题

1. 用C实现一个不可变的字符串库
2. 用闭包模拟"对象"（参见本章代码）
3. 实现一个线程安全的队列

---

**下一章预告**：解释器实现——用C手写一个Lisp解释器，理解编程语言的本质
