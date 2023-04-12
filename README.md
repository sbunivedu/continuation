## What is a continuation?
* It is a **value** encoding a **saved return point** to resume.
* It is a **function** encoding the remainder of the program.
* It is a **function** that **never** returns (normally). When invoked on an input value,
it resumes a previous **return point** with that value and finishes the program
from that **return point** until it exits.

Continuation generalize all know control constructs: gotos, loops, return
statements, exceptions, threads/coroutines, etc.

In continuation passing style, every function accepts a continuation as a input
parameter and repalces the return value with a call to the continuation with the
return value. All calls to continuations are tail calls so the control jumps to
the continuations (functions) in their defining environment. There is no return,
therefore, no stack is needed.
The stack is implicit in those function calls.

```scheme
(define fact
  (lambda (n k)
    (if (= n 1)
        (k 1)
        (fact (- n 1) (lambda (x)
                        (k (* x n)))))))

(define list-length
  (lambda (lst k)
    (if (null? lst)
        (k 0)
        (list-length (cdr lst) (lambda (x)
                            (k (+ x 1)))))))

(define fib
  (lambda (n k)
    (if (or (= n 1) (= n 2))
        (k 1)
        (fib (- n 1) (lambda (x)
                       (fib (- n 2) (lambda (y)
                                      (k (+ x y)))))))))
```

```scheme
> (fact 4 (lambda (x) x))
24
> (list-length '(a b c) (lambda (x) x))
3
> (fib 6 (lambda (x) x))
8
```

```scheme
#lang scheme

(define member?-cps
  (lambda (item lst k)
    (cond
     ((null? lst) (k #f))
     ((eq? item (car lst)) (k #t))
     (else (member?-cps item
                        (cdr lst)
                        k)))))

(member?-cps 'a '(c b a) (lambda (x) x))

(define list-sum
  (lambda (lst k)
    (if (null? lst)
        (k 0)
        (list-sum (cdr lst)
                  (lambda (v)
                    (k (+ v (car lst))))))))

(list-sum '(1 2 3) (lambda (x) x))

(define power
  (lambda (n m k)
    (if (= m 0)
        (k 1)
        (power n
               (- m 1)
               (lambda (v)
                 (k (* v n)))))))

(power 2 3 (lambda (x) x))

(define map
  (lambda (f lst k)
    (if (null? lst)
        (k '())
        (map f
             (cdr lst)
             (lambda (v)
               (k (cons (f (car lst)) v)))))))

(map list '(1 2 3) (lambda (x) x))
```

```scheme
(define list-length
  (lambda (lst)
    (if (null? lst)
        0
        (+ 1 (list-length (cdr lst))))))

(list-length '(a b c))
(list-length '(a b . c))
```
Scheme allows the continuation of any expression to be captured with the procedure `call/cc`.
`call/cc` must be passed a procedure `p` of one argument. `call/cc` constructs a concrete
representation of the current continuation and passes it to `p`. The continuation itself
is represented by a procedure `k`. Each time `k` is applied to a value, it returns the value
to the continuation of the `call/cc` application. This value becomes, in essence, the value of
the application of `call/cc`.

```scheme
(call/cc
  (lambda (k)
    (* 5 4)))
;=> 20

(call/cc
  (lambda (k)
    (* 5 (k 4))))
;=> 4

(+ 2
   (call/cc
     (lambda (k)
       (* 5 (k 4)))))
;=> 6
```
[source](https://www.scheme.com/tspl4/further.html#g63)

## Use of call/cc to provide a nonlocal exit from a recursion

```scheme
(define product
  (lambda (ls)
    (call/cc
      (lambda (break)
        (let f ([ls ls])
          (cond
            [(null? ls) 1]
            [(= (car ls) 0) (break 0)]
            [else (* (car ls) (f (cdr ls)))]))))))

;(product '(1 2 3 4 5)) => 120
;(product '(7 3 8 0 1 9 5)) => 0
```
[source](https://www.scheme.com/tspl4/further.html#g63)

CPS also allows a procedure to take separate "success" and "failure" continuations,
which may accept different numbers of arguments. An example is `integer-divide` below,
which passes the quotient and remainder of its first two arguments to its third,
unless the second argument (the divisor) is zero, in which case it passes an error
 message to its fourth argument.
```scheme
(define integer-divide
  (lambda (x y success failure)
    (if (= y 0)
        (failure "divide by zero")
        (let ([q (quotient x y)])
          (success q (- x (* q y)))))))

;(integer-divide 10 3 list (lambda (x) x)) => (3 1)
;(integer-divide 10 0 list (lambda (x) x)) => "divide by zero"
```

Any program that uses `call/cc` can be rewritten in CPS without `call/cc`:
```scheme
(define product
  (lambda (ls k)
    (let ([break k])
      (let f ([ls ls] [k k])
        (cond
          [(null? ls) (k 1)]
          [(= (car ls) 0) (break 0)]
          [else (f (cdr ls)
                   (lambda (x)
                     (k (* (car ls) x))))])))))

;(product '(1 2 3 4 5) (lambda (x) x)) => 120
;(product '(7 3 8 0 1 9 5) (lambda (x) x)) => 0
```

## A catch and throw example

```scheme
; this is not valid Scheme code
#;(define (list-length lst)
  (catch 'exit
    (letrec ((list-length1
              (lambda (lst)
                (cond ((null? lst) 0)
                      ((pair? lst) (+ 1 (list-length1 (cdr lst))))
                      (else (throw 'exit 'improper-list))))))
      (list-length1 lst))))

(define (list-length lst)
  (call/cc
   (lambda (proc)
     (letrec ((list-length1
               (lambda (lst)
                 (cond ((null? lst) 0)
                       ((pair? lst) (+ 1 (list-length1 (cdr lst))))
                       (else (proc 'improper-list))))))
       (list-length1 lst)))))

;(list-length '(a b c)) => 3
;(list-length '(a b . c)) => 'improper-list
```
[source](https://homes.cs.aau.dk/~normark/pp/other-paradigms-continuations-slide-catch-throw-ex.html)

Resources:
* [Introduction to Continuations (YouTube video)](https://youtu.be/DW3TEyAScsY)
* [Continuations: The Swiss Army Knife of Flow Control (YouTube video)](https://youtu.be/Ju3KKu_mthg)
* [Continuations (YouTube video)](https://youtu.be/K-AhJgjb-8s)
* [What is a continuation?](https://youtu.be/zB5LTkaJaqk)



