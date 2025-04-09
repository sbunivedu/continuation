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
(require racket/trace)
(define product
  (lambda (lst)
    (call/cc
      (lambda (break)
        (define helper
          (lambda (lst)
            (if (null? lst)
                1
                (if (= 0 (car lst))
                    (break 0)
                    (* (car lst) (helper (cdr lst)))))))
        (trace helper)
        (helper lst)))))

;(product '(1 2 3 4 5)) => 120
;(product '(7 3 8 0 1 9 5)) => 0
```

Another example:
```scheme
(define (find-even lst)
  (call/cc
   (lambda (exit)
     (define (helper lst)
       (if (null? lst)
           #f
           (if (even? (car lst))
               (exit (car lst))
               (helper (cdr lst)))))
     (helper lst))))

;(find-even '(1 3 5 8 9)) => 8
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
        (success (quotient x y) (remainder x y)))))

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
; a list-length definition in direct style
(define list-length
  (lambda (lst)
    (if (null? lst)
        0
        (+ 1 (list-length (cdr lst))))))

;(list-length '(a b c)) => 3
;(list-length '(a b . c)) => cdr: contract violation
```
```scheme
; this is not valid Scheme code
(define (list-length lst)
  (catch 'exit
         (letrec ((helper
                   (lambda (lst)
                     (cond ((null? lst) 0)
                           ((pair? lst) (+ 1 (helper (cdr lst))))
                           (else (throw 'exit 'improper-list))))))
           (helper lst))))
```
```scheme
#lang scheme

(define (list-length lst)
  (call/cc
   (lambda (proc)
     (letrec ((helper
               (lambda (lst)
                 (cond ((null? lst) 0)
                       ((pair? lst) (+ 1 (helper (cdr lst))))
                       (else (proc 'improper-list))))))
       (helper lst)))))

;(list-length '(a b c)) => 3
;(list-length '(a b . c)) => 'improper-list
```

[source](https://homes.cs.aau.dk/~normark/pp/other-paradigms-continuations-slide-catch-throw-ex.html)

## Another example
```scheme
(define retry #f)

(define factorial
  (lambda (x)
    (if (= x 0)
        (call/cc (lambda (k) (set! retry k) 1))
        (* x (factorial (- x 1))))))
```
With this definition, factorial works as we expect factorial to work, except it has the side effect of assigning retry.
```
(factorial 4) => 24
(retry 1) => 24
(retry 2) => 48
```

The continuation bound to "retry" might be described as "Multiply the value by 1, then multiply this result by 2, then multiply
this result by 3, then multiply this result by 4." If we pass the continuation a different value, i.e., not 1, we will
cause the base value to be something other than 1 and hence change the end result.
```
(retry 2) => 48
(retry 5) => 120
```
This mechanism could be the basis for a breakpoint package implemented with `call/cc`; each time a breakpoint is encountered,
the continuation of the breakpoint is saved so that the computation may be restarted from the breakpoint (more than once, if desired).
[source](https://homes.cs.aau.dk/~normark/pp/other-paradigms-continuations-slide-catch-throw-ex.html)

## Simulate exceptions (try/catch)
```scheme
(define (try/catch try-block catch-block)
  (call/cc (lambda (throw)
             (try-block (lambda (err) (catch-block err throw))))))

(define (example)
  (try/catch
   (lambda (throw)
     (display "Enter a number: ")
     (let ((input (read)))
       (if (not (number? input))
           (throw "Invalid input!")      ;; Jump to catch block
           (display "Success!"))))
   (lambda (err throw)
     (display "Caught an error: ")
     (display err))))
```

## Use `call/cc` for multiple returns
You can store a continuation for later use, allowing functions to "jump back" to a saved state.

```scheme
(define saved-cont #f)

(define (test-cont)
  (call/cc (lambda (k)
             (set! saved-cont k)  ;; Save continuation
             0)))  ;; Initial return value

(test-cont)     => 0   ;; First call, returns 0
(saved-cont 10) => 10  ;; Jumps back and returns 10
(saved-cont 99) => 99  ;; Jumps back again, returns 99
```

## Implement backtracking
```scheme
#lang scheme

(define (find-path grid)
  (let ((rows (length grid))
         (cols (length (car grid)))
         (path '())  ;; Stores the successful path
         (failed-cont #f))  ;; Stores the continuation for backtracking

    (define (in-bounds? x y)
      (and (>= x 0) (< x rows) (>= y 0) (< y cols)))

    (define (is-open? x y)
      (and (in-bounds? x y) (= (list-ref (list-ref grid x) y) 0)))

    (define (move x y k)
      (if (is-open? x y)
          (call/cc (lambda (abort)
                     (set! failed-cont abort)  ;; Save backtrack point
                     (set! path (cons (list x y) path))  ;; Add position to path
                     (if (and (= x (- rows 1)) (= y (- cols 1)))
                         (k path)  ;; Found goal → return path
                         (begin
                           (move (+ x 1) y k)  ;; Try moving down
                           (move x (+ y 1) k)  ;; Try moving right
                           (move (- x 1) y k)  ;; Try moving up
                           (move x (- y 1) k)  ;; Try moving left
                           (set! path (cdr path))  ;; Remove failed move
                           (failed-cont 'fail))))) ;; Backtrack if all fail
          #f))

    (call/cc (lambda (k)
               (move 0 0 k)
               'no-path))))  ;; If no path is found

;; Example grid
(define grid '((0 0 1 0)
               (1 0 1 0)
               (1 0 0 0)
               (1 1 1 0)))

;(find-path grid) => ((3 3) (2 3) (2 2) (2 1) (1 1) (0 1) (0 0))
```

Resources:
* [Introduction to Continuations (YouTube video)](https://youtu.be/DW3TEyAScsY)
* [Continuations: The Swiss Army Knife of Flow Control (YouTube video)](https://youtu.be/Ju3KKu_mthg)
* [Continuations (YouTube video)](https://youtu.be/K-AhJgjb-8s)
* [What is a continuation?](https://youtu.be/zB5LTkaJaqk)



