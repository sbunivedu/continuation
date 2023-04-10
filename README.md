What is a continuation?
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

```
> (fact 4 (lambda (x) x))
24
> (list-length '(a b c) (lambda (x) x))
3
> (fib 6 (lambda (x) x))
8
```

Resources:
* [Introduction to Continuations (YouTube video)](https://youtu.be/DW3TEyAScsY)
* [Continuations: The Swiss Army Knife of Flow Control (YouTube video)](https://youtu.be/Ju3KKu_mthg)
* [Continuations (YouTube video)](https://youtu.be/K-AhJgjb-8s)
* [What is a continuation?](https://youtu.be/zB5LTkaJaqk)



