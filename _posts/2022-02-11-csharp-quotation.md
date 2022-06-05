---
title: Цитирование в C#
date: "2022-02-11 19:00:00 +0300"
id: csharp-quotation
excerpt: "Использование деревьев выражений для символического вычисления производных функций."
mathjax: true
---

В этом году весенний DotNext перенесли на неопределённый срок. А я к нему готовлюсь.

В рамках доклада про разные интересности в C#, разработал небольшую библиотеку для генерации производных функций.

### Задача

Саму задачу я встретил, решая упражнения из книги [Структура и Интерпретация Компьютерных Программ](https://ru.wikipedia.org/wiki/Структура_и_интерпретация_компьютерных_программ#:~:text=«Структу́ра%20и%20интерпрета́ция%20компью́терных%20програ́мм,технологического%20института%20в%201985%20году.). Обычно её называют SICP (читается *сик-пи*) — это аббревиатура названия на английском языке.

Раздел 2.3 посвящён *цитированию* в LISP и *символическим вычислениям*.

Обычные — несимволические — вычисления сводятся к тому, что мы считаем какие-то величины с помощью арифметических операций. Например, если я попрошу вас вычислить производную функции
$x^2$ в точке $x = 17$, вы можете сделать это по формуле при каком-нибудь не очень большом значении $dx$.

$$
(x^2)' = \frac{(x+dx)^2-x^2}{dx}
$$

Подгоняя $dx$, мы можем вычислить производную с хорошей точностью.

$$
\frac{(17+0.0001)^2-17^2}{0.0001} = 34.0001000001
$$

Символические же вычисления позволяют нам применить правила выведения производных и получить значение абсолютно точно.

$$
(x^2)' = 2x
$$

При $x = 17$ значение производной будет абсолютно точно равно $34$.

### Реализация на Scheme

SICP предлагает вычислять производную с помощью *цитирования*. По-английски этот механизм называется *quotation*.

Если мы вводим в интерпретатор Scheme любое выражение, он вычисляет его сразу.

```scheme
(+ (/ 1 1) (/ 1 1) (/ 1 2) (/ 1 6) (/ 1 24) (/ 1 120) (/ 1 720) (/ 1 5040))
; => 2.7182539682539684
```

Но если мы предваряем его *кавычкой* (*quote*), Scheme сохраняет выражение в виде списка, не вычисляя.

```scheme
'(+ (/ 1 1) (/ 1 1) (/ 1 2) (/ 1 6) (/ 1 24) (/ 1 120) (/ 1 720) (/ 1 5040))
; => (+ (/ 1 1) (/ 1 1) (/ 1 2) (/ 1 6) (/ 1 24) (/ 1 120) (/ 1 720) (/ 1 5040))
```

Таким образом, мы получаем корректное выражение на LISP и можем обработать его, как любой другой список, в частности, преобразовать по правилам вычисления производной.

Вот простая функция, которая строит производную сумм и произведений.

```scheme
(define (variable? x) (symbol? x))
(define (same-variable? v1 v2)
  (and (variable? v1) (variable? v2) (eq? v1 v2)))
(define (make-sum a1 a2) (list '+ a1 a2))
(define (make-product m1 m2) (list '* m1 m2))
(define (sum? x)
  (and (pair? x) (eq? (car x) '+)))
(define (addend s) (cadr s))
(define (augend s) (caddr s))
(define (product? x)
  (and (pair? x) (eq? (car x) '*)))
(define (multiplier p) (cadr p))
(define (multiplicand p) (caddr p))

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
                        (multiplicand exp))))
        (else
         (error "Unknown expression type"))))
```

Очевидным недостатком функции является сложность получаемых выражений.

```scheme
(deriv '(+ x 3) 'x)
; => (+ 1 0)

(deriv '(* x y) 'x)
; => (+ (* x 0) (* 1 y))

(deriv '(* (* x y) (+ x 3)) 'x)
; => (+ (* (* x y) (+ 1 0)) (* (+ (* x 0) (* 1 y)) (+ x 3)))
```

Их надо упрощать, для чего может быть написана отдельная функция. Упрощение выражений также рассматривается в SICP.

### Реализация на F#

Цитирование на F# всё ещё похоже на цитирование.

```fsharp
let expSquare = <@ fun x -> x * x @>
// => val expSquare : Quotations.Expr<(int -> int)> = Lambda (x, Call (None, op_Multiply, [x, x]))
```

Чтобы получить вместо кода его представление в виде сложного объекта, заключим код в своеобразные кавычки — **<@** и **@>**.

Результатом будет значение типа `Expr`, с которым можно работать также, как с деревьями выражений в C#.
Вот простая функция, которая строит производную сумм и произведений.

```fsharp
open Microsoft.FSharp.Quotations
open Microsoft.FSharp.Quotations.Patterns
open Microsoft.FSharp.Quotations.DerivedPatterns

let  make_sum left right =
    let left = Expr.Cast<float> left
    let right = Expr.Cast<float> right 
    <@ %left + %right @> :> Expr
    
let make_prod left right =
    let left = Expr.Cast<float> left
    let right = Expr.Cast<float> right 
    <@ %left * %right @> :> Expr

let deriv (exp: Expr) =
    match exp with
    | Lambda(arg, body) ->
        let rec d exp =
            match exp with
            | Int32(_) ->
                Expr.Value 0.0
            | Var(var) ->
                if var.Name = arg.Name
                then Expr.Value 1.0
                else Expr.Value 0.0
            | Double(_) ->
                Expr.Value 0.0
            | SpecificCall <@ (+) @> (None, _, [left; right]) ->
                make_sum (d left) (d right)
            | SpecificCall <@ (*) @> (_, _, [left; right]) ->
                let left = Expr.Cast<float> left
                let right = Expr.Cast<float> left
                make_sum (make_prod left (d right)) (make_prod (d left) right)
            | _ -> failwith "Unknown expression type"

        d body
    | _ -> failwith "Expr.Lambda expected"

<@ fun (x: double) -> x * x @>
// => Lambda (x, Call (None, op_Multiply, [x, x]))

deriv <@ fun (x: double) -> x * x @>
// => Call (None op_Addition,
//          [Call (None, op_Multiply, [x, Value (1.0)]),
//           Call (None, op_Multiply, [Value (1.0), x])])
```

### Реализация на C#

В C# существует аналог *цитирования* — *деревья выражений*. В отличие от F#, здесь нет специальных кавычек для выделения кода. Вместо этого мы указываем тип выражения `Expression`, а всё остальное делает механизм *вывода типов*.

Обычные выражения вычисляются сразу.

```c#
Func<double, double> square = x => x * x;
sqaure(2) // 4
```

Выражения, которые приводятся к типу [`Expression`](https://docs.microsoft.com/en-us/dotnet/api/system.linq.expressions.expression), складываются в древовидную структуру, которую мы сможем потом обработать.

```c#
Expression<Func<double, double>> expSquare = x => x * x;
expSquare.Compile()(2) // 4
```

Деревья выражений хорошо знакомы многим программистам на C#, поскольку они применяются в библиотеке Entity Framework. Однако, с помощью их можно делать и более сложную обработку.

Вот функция, которая получает на вход лямбда-функцию и применяет её к самой себе.

```c#
static Expression<Func<double, double>> DoubleFunc(Expression<Func<double, double>> f)
{
    var parameter = Expression.Parameter(typeof(double));
    var inner = Expression.Invoke(f, parameter);
    var outer = Expression.Invoke(f, inner);
    return Expression.Lambda<Func<double, double>>(outer, parameter);
}

var expFourth = DoubleFunc(expSquare);
```

Если два раза применить функцию возведения в квадрат к какому-то числу, мы получим возведение в четвёртую степень:

```c#
expFourth.Compile()(2) // 16
```

Я разработал небольшой [пакет](https://www.nuget.org/packages/SySharp/), который умеет генерировать производные функции по деревьям выражений. [Исходный код](https://github.com/markshevchenko/sysharp)) пакета открыт.

```c#
Symbolic.Derivative(x => x * x).ToString()
// => x => ((x * 1) + (1 * x))
```

Там же реализован код для упрощения выражений.

```c#
Symbolic.Derivative(x => x * x).Simplify().ToString()
// => x => (2 * x)
```

В отличие от F#, в C# очень просто из дерева выражения получить работающий код.

```c#
var d = (Func<double, double>)Symbolic.Derivative(x => x * x).Compile();
d(17)
// => 34
```