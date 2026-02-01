# 第4章：解释器实现

## 引言：解释器是魔法吗？

你觉得编程语言很神秘？

```c
int x = 5;
int y = x + 10;
return y;
```

这行代码如何变成机器指令？编译器做了什么？

**本章目标：用C写一个Lisp解释器**

理解了解释器，你就理解了：
- 编程语言如何工作
- 代码和数据的关系
- 作用域、闭包的本质
- eval和apply的通用模式

---

## 4.1 解释器的基本架构

### 4.1.1 核心循环

```
┌─────────────┐
│   读取代码   │  Read
└──────┬──────┘
       ↓
┌─────────────┐
│   求值      │  Eval
└──────┬──────┘
       ↓
┌─────────────┐
│   打印结果   │  Print
└──────┬──────┘
       ↓
   ┌──────┐
   │ 循环 │   Loop
   └──────┘
```

**简化的eval**：

```c
// 伪代码
Value* eval(Value* exp, Environment* env) {
    if (is_number(exp)) {
        return exp;  // 数字返回自身
    }
    if (is_symbol(exp)) {
        return lookup(env, exp);  // 查找变量
    }
    if (is_list(exp)) {
        Value* op = eval(car(exp), env);
        Value* args = eval_list(cdr(exp), env);
        return apply(op, args);  // 应用函数
    }
    error("Unknown expression");
}
```

**Scheme版本（更简洁）**：

```scheme
(define (eval exp env)
  (cond ((self-evaluating? exp) exp)
        ((variable? exp) (lookup-variable-value exp env))
        ((quoted? exp) (text-of-quotation exp))
        ((assignment? exp) (eval-assignment exp env))
        ((definition? exp) (eval-definition exp env))
        ((if? exp) (eval-if exp env))
        ((lambda? exp)
         (make-procedure (lambda-parameters exp)
                         (lambda-body exp)
                         env))
        ((application? exp)
         (apply (eval (operator exp) env)
                (list-of-values (operands exp) env)))
        (else (error "Unknown expression"))))
```

---

## 4.2 用C实现Lisp解释器

### 4.2.1 数据结构

**值的表示**：

```c
// lisp.h
typedef enum {
    TYPE_NUMBER,
    TYPE_SYMBOL,
    TYPE_LIST,
    TYPE_PRIMITIVE,
    TYPE_PROCEDURE,
    TYPE_ENVIRONMENT
} ValueType;

typedef struct Value Value;
typedef struct Environment Environment;

struct Value {
    ValueType type;
    union {
        double number;
        char* symbol;
        struct {
            Value* car;
            Value* cdr;
        } list;
        Value* (*primitive)(Value* args);
        struct {
            Value* params;
            Value* body;
            Environment* env;
        } procedure;
    } as;
};

struct Environment {
    Environment* parent;
    Value* vars;   // 变量名列表
    Value* vals;   // 对应值列表
};
```

**构造函数**：

```c
// 创建数字
Value* make_number(double x) {
    Value* v = malloc(sizeof(Value));
    v->type = TYPE_NUMBER;
    v->as.number = x;
    return v;
}

// 创建符号
Value* make_symbol(char* s) {
    Value* v = malloc(sizeof(Value));
    v->type = TYPE_SYMBOL;
    v->as.symbol = strdup(s);
    return v;
}

// 创建列表 (cons)
Value* make_list(Value* car, Value* cdr) {
    Value* v = malloc(sizeof(Value));
    v->type = TYPE_LIST;
    v->as.list.car = car;
    v->as.list.cdr = cdr;
    return v;
}
```

### 4.2.2 读取（Read）

**词法分析**：

```c
typedef enum {
    TOKEN_LPAREN,
    TOKEN_RPAREN,
    TOKEN_NUMBER,
    TOKEN_SYMBOL,
    TOKEN_EOF
} TokenType;

typedef struct {
    TokenType type;
    union {
        double number;
        char* symbol;
    } value;
} Token;

Token* tokenize(char* input) {
    Token* tokens = malloc(sizeof(Token) * 1000);
    int i = 0;
    char* p = input;

    while (*p) {
        if (isspace(*p)) {
            p++;
            continue;
        }
        if (*p == '(') {
            tokens[i].type = TOKEN_LPAREN;
            i++;
            p++;
            continue;
        }
        if (*p == ')') {
            tokens[i].type = TOKEN_RPAREN;
            i++;
            p++;
            continue;
        }
        if (isdigit(*p) || *p == '.') {
            tokens[i].type = TOKEN_NUMBER;
            tokens[i].value.number = strtod(p, &p);
            i++;
            continue;
        }
        // 符号
        char* start = p;
        while (!isspace(*p) && *p != '(' && *p != ')') {
            p++;
        }
        tokens[i].type = TOKEN_SYMBOL;
        tokens[i].value.symbol = strndup(start, p - start);
        i++;
    }
    tokens[i].type = TOKEN_EOF;
    return tokens;
}
```

**语法分析**：

```c
Value* read(Token** tokens) {
    Token* token = *tokens;

    if (token->type == TOKEN_NUMBER) {
        (*tokens)++;
        return make_number(token->value.number);
    }

    if (token->type == TOKEN_SYMBOL) {
        (*tokens)++;
        return make_symbol(token->value.symbol);
    }

    if (token->type == TOKEN_LPAREN) {
        (*tokens)++;
        Value* list = NULL;
        Value** tail = &list;

        while ((*tokens)->type != TOKEN_RPAREN) {
            Value* elem = read(tokens);
            *tail = make_list(elem, NULL);
            tail = &((*tail)->as.list.cdr);
        }
        (*tokens)++;  // 跳过 )
        return list;
    }

    error("Unexpected token");
}
```

### 4.2.3 求值（Eval）

**eval的骨架**：

```c
Value* eval(Value* exp, Environment* env) {
    switch (exp->type) {
        case TYPE_NUMBER:
            return exp;  // 数字返回自身

        case TYPE_SYMBOL:
            return env_lookup(env, exp);

        case TYPE_LIST: {
            Value* car_val = exp->as.list.car;
            Value* cdr_val = exp->as.list.cdr;

            // 空列表
            if (car_val == NULL) {
                return exp;
            }

            // 特殊形式
            if (is_symbol(car_val, "quote")) {
                return cdr_val->as.list.car;
            }

            if (is_symbol(car_val, "if")) {
                Value* test = eval(cdr_val->as.list.car, env);
                Value* consequence = cdr_val->as.list.cdr->as.list.car;
                Value* alternative = cdr_val->as.list.cdr->as.list.cdr->as.list.car;

                if (is_truthy(test)) {
                    return eval(consequence, env);
                } else {
                    return eval(alternative, env);
                }
            }

            if (is_symbol(car_val, "lambda")) {
                Value* params = cdr_val->as.list.car;
                Value* body = cdr_val->as.list.cdr;
                return make_procedure(params, body, env);
            }

            if (is_symbol(car_val, "define")) {
                Value* var = cdr_val->as.list.car;
                Value* val = eval(cdr_val->as.list.cdr->as.list.car, env);
                env_define(env, var, val);
                return val;
            }

            // 函数应用
            Value* op = eval(car_val, env);
            Value* args = eval_list(cdr_val, env);
            return apply(op, args);
        }

        default:
            error("Cannot evaluate");
    }
}

Value* eval_list(Value* list, Environment* env) {
    if (list == NULL) {
        return NULL;
    }
    return make_list(
        eval(list->as.list.car, env),
        eval_list(list->as.list.cdr, env)
    );
}
```

### 4.2.4 应用（Apply）

```c
Value* apply(Value* proc, Value* args) {
    switch (proc->type) {
        case TYPE_PRIMITIVE:
            return proc->as.primitive(args);

        case TYPE_PROCEDURE: {
            Environment* env = env_extend(
                proc->as.procedure.env,
                proc->as.procedure.params,
                args
            );
            Value* body = proc->as.procedure.body;
            Value* result = NULL;
            while (body != NULL) {
                result = eval(body->as.list.car, env);
                body = body->as.list.cdr;
            }
            return result;
        }

        default:
            error("Not a procedure");
    }
}
```

### 4.2.5 环境模型

```c
Environment* env_extend(Environment* parent, Value* vars, Value* vals) {
    Environment* env = malloc(sizeof(Environment));
    env->parent = parent;
    env->vars = vars;
    env->vals = vals;
    return env;
}

Value* env_lookup(Environment* env, Value* var) {
    while (env != NULL) {
        Value* vars = env->vars;
        Value* vals = env->vals;

        while (vars != NULL) {
            if (strcmp(vars->as.list.car->as.symbol,
                       var->as.symbol) == 0) {
                return vals->as.list.car;
            }
            vars = vars->as.list.cdr;
            vals = vals->as.list.cdr;
        }
        env = env->parent;
    }
    error("Undefined variable");
}

void env_define(Environment* env, Value* var, Value* val) {
    env->vars = make_list(var, env->vars);
    env->vals = make_list(val, env->vals);
}
```

### 4.2.6 基本操作

```c
// 加法
Value* primitive_add(Value* args) {
    double result = 0;
    while (args != NULL) {
        result += args->as.list.car->as.number;
        args = args->as.list.cdr;
    }
    return make_number(result);
}

// 减法
Value* primitive_sub(Value* args) {
    if (args->as.list.cdr == NULL) {
        return make_number(-args->as.list.car->as.number);
    }
    double result = args->as.list.car->as.number;
    args = args->as.list.cdr;
    while (args != NULL) {
        result -= args->as.list.car->as.number;
        args = args->as.list.cdr;
    }
    return make_number(result);
}

// 乘法
Value* primitive_mul(Value* args) {
    double result = 1;
    while (args != NULL) {
        result *= args->as.list.car->as.number;
        args = args->as.list.cdr;
    }
    return make_number(result);
}

// 注册基本操作
Environment* make_global_env() {
    Environment* env = malloc(sizeof(Environment));
    env->parent = NULL;
    env->vars = NULL;
    env->vals = NULL;

    env_define(env, make_symbol("+"), make_primitive(primitive_add));
    env_define(env, make_symbol("-"), make_primitive(primitive_sub));
    env_define(env, make_symbol("*"), make_primitive(primitive_mul));

    return env;
}
```

### 4.2.7 主循环

```c
void repl(Environment* env) {
    char input[1024];

    while (1) {
        printf("> ");
        if (fgets(input, sizeof(input), stdin) == NULL) {
            break;
        }

        Token* tokens = tokenize(input);
        Value* exp = read(&tokens);
        Value* result = eval(exp, env);
        print_value(result);
        printf("\n");
    }
}

int main() {
    Environment* global = make_global_env();
    repl(global);
    return 0;
}
```

---

## 4.3 理解eval和apply

### 4.3.1 递归的互调用

```
eval: 处理表达式
  ↓
  遇到函数调用 (f arg1 arg2)
  ↓
  eval f (得到函数)
  eval arg1, arg2 (得到参数值)
  ↓
  apply: 应用函数到参数
  ↓
  如果是复合函数:
    创建新环境
    ↓
    eval: 执行函数体
  ↓
  返回结果
```

**这个循环是所有解释器的核心！**

### 4.3.2 例子：求值 (+ 1 2)

```
输入: (+ 1 2)

eval((+ 1 2), env)
  ↓ 识别为函数调用
  eval(+, env)
    → + 是基本操作
  eval(1, env)
    → 1 (自求值)
  eval(2, env)
    → 2 (自求值)
  ↓
  apply(+, (1 2))
    → 1 + 2 = 3
  ↓
  返回 3
```

**例子：求值 (lambda (x) (+ x 1))**

```
eval((lambda (x) (+ x 1)), env)
  ↓ 识别为lambda
  创建procedure对象:
    params: (x)
    body: ((+ x 1))
    env: 当前环境
  ↓
  返回procedure对象
```

**例子：应用 ((lambda (x) (+ x 1)) 5)**

```
eval(((lambda (x) (+ x 1)) 5), env)
  ↓ 函数应用
  eval((lambda (x) (+ x 1)), env)
    → procedure{params=(x), body=(+ x 1), env}
  eval(5, env)
    → 5
  ↓
  apply(procedure, (5))
    创建新环境: {x: 5, parent=procedure.env}
    eval((+ x 1), 新环境)
      eval(+, 新环境) → + (基本操作)
      eval(x, 新环境) → 5 (从环境查表)
      eval(1, 新环境) → 1
      apply(+, (5 1)) → 6
  ↓
  返回 6
```

---

## 4.4 闭包的实现

### 4.4.1 问题：函数记住环境

```scheme
(define (make-adder n)
  (lambda (x) (+ x n)))

(define add3 (make-adder 3))
(add3 5)  ; 8
```

**在C中，如何实现？**

```c
// make-adder 创建时：
Value* make_adder_proc = eval(make_adder_code, global_env);
// make_adder_proc 是一个procedure对象:
//   params: (n)
//   body: ((lambda (x) (+ x n)))
//   env: global_env

// 调用 (make-adder 3) 时:
Value* add3_proc = apply(make_adder_proc, list(3));
// add3_proc 是:
//   params: (x)
//   body: ((+ x n))
//   env: {n: 3, parent=global_env}  ← 关键！

// 调用 (add3 5) 时:
Value* result = apply(add3_proc, list(5));
// 创建新环境: {x: 5, parent=add3_proc.env}
// eval((+ x n), 这个环境):
//   eval(+, ...) → +
//   eval(x, ...) → 5
//   eval(n, ...) → 查找x的环境，找到父环境中的n=3
//   apply(+, (5 3)) → 8
```

**环境链实现了闭包！**

### 4.4.2 环境链可视化

```
调用 (add3 5) 时的环境结构:

global_env:
  +: primitive_add
  make-adder: procedure{...}

   ↑ parent

  env1 (make-adder 3创建):
    n: 3
    make-adder返回的procedure.env指向这里

     ↑ parent

    env2 (add3 5创建):
      x: 5

      查找 n 时:
      1. 查env2: 没有
      2. 查env1: 找到 n=3
      3. 返回 3
```

---

## 4.5 特殊形式

### 4.5.1 为什么需要特殊形式？

**问题**：`(if (= x 0) 1 (/ 1 x))`

如果`if`是普通函数：
```
先求值所有参数:
  (= x 0) → #t 或 #f
  1 → 1
  (/ 1 0) → 错误！除零
```

**解决方案：if是特殊形式，控制求值顺序**

```c
// 在eval中特殊处理
if (is_symbol(car_val, "if")) {
    Value* test = eval(cdr_val->as.list.car, env);
    // 先求值test

    if (is_truthy(test)) {
        Value* consequence = cdr_val->as.list.cdr->as.list.car;
        return eval(consequence, env);  // 只求值consequent
    } else {
        Value* alternative = cdr_val->as.list.cdr->as.list.cdr->as.list.car;
        return eval(alternative, env);  // 只求值alternative
    }
    // 不会同时求值两者！
}
```

### 4.5.2 其他特殊形式

**define**：不求值变量名

```c
if (is_symbol(car_val, "define")) {
    Value* var = cdr_val->as.list.car;  // 不求值！
    Value* val = eval(cdr_val->as.list.cdr->as.list.car, env);
    env_define(env, var, val);
    return val;
}
```

**lambda**：不求值参数和body

```c
if (is_symbol(car_val, "lambda")) {
    Value* params = cdr_val->as.list.car;  // 不求值！
    Value* body = cdr_val->as.list.cdr;    // 不求值！
    return make_procedure(params, body, env);
}
```

---

## 4.6 完整例子：计算阶乘

**输入代码**：

```scheme
(define factorial
  (lambda (n)
    (if (= n 0)
        1
        (* n (factorial (- n 1))))))

(factorial 5)
```

**求值过程**：

```
1. eval((define factorial (lambda ...)), global)
   → 创建procedure对象
   → env_define(global, factorial, procedure)

2. eval((factorial 5), global)
   → eval(factorial, global) → procedure对象
   → eval(5, global) → 5
   → apply(procedure, (5))

3. apply时，创建环境 {n: 5, parent=global}
   → eval((if (= n 0) 1 (* n (factorial (- n 1)))), env)

4. eval(if):
   → eval((= n 0), env)
     → eval(=, env) → primitive_=
     → eval(n, env) → 5
     → eval(0, env) → 0
     → apply(=, (5 0)) → #f
   → #f是false，求值alternative
   → eval((* n (factorial (- n 1))), env)

5. eval(*):
   → eval(*, env) → primitive_*
   → eval(n, env) → 5
   → eval((factorial (- n 1)), env)  ← 递归调用！

6. 重复步骤3-5，但n=4, 3, 2, 1, 0

7. n=0时，if返回1

8. 递归返回:
   factorial(1) = (* 1 1) = 1
   factorial(2) = (* 2 1) = 2
   factorial(3) = (* 3 2) = 6
   factorial(4) = (* 4 6) = 24
   factorial(5) = (* 5 24) = 120

9. 返回 120
```

---

## 本章小结

### 解释器的核心

| 组件 | 作用 | C实现 |
|------|------|-------|
| **Read** | 解析代码 | tokenize + parse |
| **Eval** | 求值表达式 | eval函数 |
| **Apply** | 应用函数 | apply函数 |
| **Environment** | 变量绑定 | 环境链 |

### eval-apply循环

```
eval(exp) → apply(proc, args) → eval(body) → ...
```

这个循环是**所有解释器的基础**

### 闭包的本质

```
闭包 = procedure对象 + 环境

环境链让函数"记住"定义时的变量
```

### 特殊形式

**为什么需要**：控制求值顺序
- `if`：条件求值
- `define`：不求值变量名
- `lambda`：延迟求值body

### 实现技巧

1. **统一数据结构**：所有值都是Value*
2. **环境链**：实现作用域和闭包
3. **递归**：eval和apply互相调用
4. **类型标记**：区分不同类型的值

---

## 练习

### 思考题

1. 为什么Python比C的解释器更复杂？
2. 如何添加let特殊形式？
3. 垃圾回收如何实现？

### 实践题

1. 扩展解释器，支持cond
2. 实现基本比较操作（<, >, =）
3. 添加set!操作，支持赋值
4. 实现简单的垃圾回收（引用计数）

---

**下一章预告**：编译器原理——如何将高级代码翻译成机器码，理解栈帧、调用约定
