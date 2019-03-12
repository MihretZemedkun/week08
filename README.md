





# CSCI 3366 Programming Languages

### Spring 2018

**R. Muller**

------

### Lecture Notes: Week 08

#### This Week

1. Exam Review; Schedule Review
2. Implementing Block-Structured Recursive Functions
3. Introduction to the MiniC compiler

---

### 1. Exam Review; Schedule Review

A key to the first midterm exam can be found in `resources/exams/`.

### 2. Implementing Block-Structured Recursive Functions

>  NB: there is more detail on this topic in the writeup `earth.pdf`. 

Functions/procedures are the primary abstraction mechanism in virtually all programming languages. Since functions and procedures can accept input arguments and have local storage, some storage space must be allocated for the duration of the activation of the function. This storage area is typically called an *activation record* or *call frame*. 

If functions and procedures are prohibited from making recursive calls, their activation records can be allocated in static memory — typically the `.data` segment of the image. But virtually all modern programming languages support recursion. This means that multiple activations of the same function or procedure can exist at the same time. For this reason, activation records are usually allocated in a stack, the *call stack*. The code compiled for programs in the language manage a special register, the *frame pointer* which points to a fixed position in the activation record. Input parameters and local storage can then be accessed via indirect addressing off of the frame pointer.

This is all reasonably simple and well-understood. However, if the language supports recursion *and* is *block-structured*, some extra plumbing is required to access variables that might be free in a nested function. Using Python notation, consider the following:

```
                                  +---+-------+
  === - static chain              | h | e : 4 +======+
                                  +-+-+-------+      |  Looking up b in the body of h
   |  - dynamic chain               V                |  requires 2 hops of the static
   v                              +---+-------+<=====+  chain.
                                  | g | c : 0 |
                                  +-+-+ d : 12+======+  Lexical Address (SC-hops, offset)
                                    | +-------+      |                                  
                                    v                |
                                  +---+-------+      |
                                  | g | c : 1 |      |
                                  +-+-+ d : 8 +=====>+
                                    | +-------+      |
def f(a, b):                        v                |
    def g(c, d):                  +---+-------+      |  Variables as Lexical Addresses
        def h(e):                 | g | c : 2 |      |
            print b + d + e       +-+-+ d : 4 +=====>+   print (2, 1) + (1, 1) + (0, 0)
        if (c == 0):                | +-------+      |   if ((0, 0) == 0)
            h(4)                    v                |
        else:                     +---+-------+<=====+
            g(c - 1, b + d)       | f | a : 0 |          g((0, 0) - 1, (1, 1) + (0, 1))
    if (a == 0):                  +-+-+ b : 4 +======+   if ((0, 0) == 0)
    	g(2, b)                     | +-------+      |   g(2, (0, 1))
    else:                           v                =
        f(a - 1, b + 2)           +---+-------+          f((0, 0), (0, 1))
                                  | f | a : 1 |
f(2, 0)                           +-+-+ b : 2 +======+
                                    | +-------+      |
                                    v                =
                                  +---+-------+ 
                                  | f | a : 2 |
                                  +---+ b : 0 +======+
                                      +-------+      |
                                                     =
```

A lookup of a non-local variable can involve some number of hops of the static chain. Each hop entails a `LOD` instruction which is likely to be relatively slow. For this reason, compilers for languages with this combination of features would sometimes implement a *display*. A display is an area of an activation record containing local copies of non-local variables. Building a display on procedure entry is time consuming. And copying display entries back to their host activation records on function exit is time consuming too.

#### Variables Escaping the Scope of their Binding Occurrences

The problem and solution described above doesn't work for more modern programming languages with functions as first-class values. For example, consider the following fragment of JavaScript code

```javascript
const k = (x) => ((y) => x)

const sixFun = k(6)                    +---+-------+
                                       | k | x : 6 |
                                       +---+-------+
```

In this example, function `k` is called with argument `6`. A stack frame is created and a function `(y) => x` is returned and stored in variable `sixFun`. Of course, after `k` returns its result, the activation record containing the binding of `x` is popped off the stack. How can `sixFun` work if the binding occurrence of `x` is gone? There are a number of different solutions including [closures](https://en.wikipedia.org/wiki/Closure_(computer_programming), [lambda lifting](https://en.wikipedia.org/wiki/Lambda_lifting) and [defunctionalization](https://en.wikipedia.org/wiki/Defunctionalization).

We'll use the following OCaml simple code as an example to motivate how closure data structures work.

```ocaml
let g = 
  let a = 4 in
  let f x = x + a
  in
  f
in
g 6
```

The body of `f` has a free occurrence of variable `a`. However, when `f` (aka `g`) is applied in `g 6`, the variable `a` is gone. We can package up a function with it's defining environment using a closure record `{code; env}`. Consider the follow alternate representation

```ocaml
let g =
  let a = 4 in
  let f = { code = (fun z -> z.arg + z.env.a)
          ; env  = {a = a}
          }
  in
  f
in
g.code {arg = 6; env = g.env}
```

Note that each of the 3 record creations above would entail memory allocations (mallocs) and each of the 5 record projections (i.e., dots) above would give rise to `LOD` instructions. So closure creation and application are both relatively expensive as compared to a standard function call.

### 3. Introduction to the MiniC compiler

See the `.ml` and `.mli` files in the `src/` folder for problem sets 7.