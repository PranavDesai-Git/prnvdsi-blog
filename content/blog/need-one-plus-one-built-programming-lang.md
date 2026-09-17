---
title: "Needed 1+1, Built a Functional Programming Language"
date: 2026-09-16T06:12:00+05:30
description: "A deep dive into why I spent a weekend building an AST parser from scratch just todo basic math."
tags: ["compilers", "c", "weekend-project", "functional programming", "graph reduction"]
categories: ["programming", "tech", "PL"]
draft: false
---

[graphLang](https://github.com/PranavDesai-Git/graphLang)


I was given a pretty straightforward data structures problem: convert an arithmetic expression into a binary tree.

Naturally, instead of stopping there, I decided to build an evaluator for it.

Then I had a thought.

What if the evaluator didn't actually need to know what the operations were?

A few days later I implemented closures, a garbage collector, and a custom memory allocator in C.

I had started with 1 + 1.

And... ended up building a functional programming language.

### THE DATA STRUCTURES ASSIGNMENT

The problem was: evaluate 1 + 1 + 1 to 3 using a binary tree.

How do we get there?

well we first form our tree for 1+1+1

```text
     (+)
     / \
   (+) (1)
   / \
 (1) (1)
```

The operator becomes the root, with its two operands as children.

Now lets evaluate this tree.

We first evaluate the left operand of the root. It's another + expression, so we have to collapse it to a value before the outer + can execute.

```text
    (+)
    / \
  (2) (1)
```

Then we evaluate again.

```
(3) <--- thats our result
```

we just performed the equivalent of
```lisp
(+ (+ 1 1) 1)
    |
    v
(+  2  1)
    |
    v
   (3)
```

But notice what the evaluator had to know to do this: what + means.

One way to represent this is to make every operation a different case in our expression type:

```text
Expr ::= Add Expr Expr
       | Sub Expr Expr
       | Mul Expr Expr
       | Div Expr Expr
       | Val
```

what do these different cases actually represent?

And does the evaluator really need to know the difference between Add and Sub?

Then I started implementing our sum types. And when I looked at the structure.
```
Add: Expr x Expr  → Expr
Sub: Expr x Expr  → Expr
Mul: Expr x Expr  → Expr
Div: Expr x Expr  → Expr
```

They all take two expressions and produce one expression

So why should the evaluator care whether the function is Add, Sub, Mul, or Div?

This is when I realized the evaluator doesnt need to know what the function does, it just needs to know what function to execute

so now we can just represent our expression as:
```text
Expr ::= Func Expr Expr
       | Val
```

The evaluator doesn't need to know what a function does. It only needs to know how to apply one.

### Oh wait. you can add vars to this. it wouldnt be a big change 

Should be a one tiny addition no problem whatsoever. I mean variables are just a hashtable lookup that hold an Expr

oh wait. C doesnt have built in hashtables.

hmmm.（´-`）.｡oO( ... )

> Lets just implement a hashtable. its small change! m9(・∀・)

soo....how does that work? I never implemeneted it before.
I look it up on google like a caveman and find this amazing text

[How to implement a hash table (in C)](https://benhoyt.com/writings/hash-table-in-c/)

So now we just got a little change in the Expr:

```
Expr ::= Func Expr Expr
       | Val 
       | Var 
```

Would you look at that! we have variables now that can be passed to functions once evaluated! just like our (+)

---

### Actually implementing it in C

alright  then time to code in C with this plan, seems simple enough. just a tagged union.


```C
typedef enum {
    LITERAL,
    VAR,
    FUNC,
} NodeType;

struct Node {
    struct Node *left;
    struct Node *right;

    union {
        int literal;
        char *var;
        char *func;
    } data;

    NodeType type;
};
```


Right now the mem size of each ( assuming 64 bit system) is:


```
+------------------------+----------+
| Field                  | Size     |
+------------------------+----------+
| Left Pointer           |  8 bytes |
| Right Pointer          |  8 bytes |
| Data                   |  8 bytes |
| Type                   |  4 bytes |
| Padding                |  4 bytes |
+------------------------+----------+
| Total                  | 32 bytes |
+------------------------+----------+
```



32 Bytes might not seem like a lot but we gotta think how this is being used. for evaluating 1+1 we would need 3 nodes.
- 1 for the operator
- 2 for the operands

That would be 32 x 3= **96 bytes** to evaluate 1+1.

But heres the thing. on my system, when you malloc() a node malloc adds a header which takes up **16 bytes** of memory!
bringing our total per node to 32(node size) + 16 (malloc header) = **48 bytes!**

And notice we will be doing a lot of little individual allocations. so, we will need a better way to allocate these nodes.

we clearly need a custom allocator.

So, I look up what allocator we can use, again like a caveman, And I decide I will be writing an Arena Allocator

---

### Arena Allocator


The idea of an arena allocator is pretty simple. all you do is at the start take a big chunk of memroy, allocate stuff yourself and then at then end just free the entire block

So my arena allocator would just be:

```C
    #define SIZE 1024
    Node arena[SIZE]
```

and when we allocate a node we can keep track of a top using
```C
    int top = 0;
```
when we want to allocate a node we just return 
```C 
    &arena[top++];
```

I wrote the allocator and defined a C function to allocate nodes:
```C 
Node *allocNode();
```

It was great! Now moving on to actually calling the functions.

Then I realized

> We can store vars and funcs in the same environment! That means functions can just be values too (°◇°)

Remember the hashtable we created earlier? its time to upgrade it.

---

### Making the Env Table


In the Env table we are storing two things. Vars and
functions.

But what are functions?
As far as our evaluator is concerned, it is a thing
that consumes arguments on one side and spits out a
result on the other.

```
    (func node)
    /         \
 (arg 1)    (arg 2)
```

The initial idea was they will just be pointers to c
funcs. But there is a huge problem with this: how do
users write their own functions?
When a user types code into our language, It can't magically compile that text into
a native C function pointer on the fly. It can only
build an AST (a tree of nodes).
Also, if functions are just C pointers, they
instantly evaluate to a literal. Soo this defeats the
purpose of higher order functions. If a function
returns another function, you can't just return a C
pointer back into the graph to be evaluated later.
Thus we need a type of node that tells the evaluator
"hey, I am a function, but my code isn't a C
pointer,it is this tree right here." It needs to
store the body of a function, which would evaluate
over multiple steps. And that is a closure.

Instead of a black-box C function, a closure is an
actual node in our graph. It holds the parameter on
one side, and the tree of operations (the body) on
the other.

```
         (closure)
         /        \
 (parameter)      (body)
                 /      \
              (math)  (literal)
```

By making the function an actual node, we can pass it
around, return it from other functions, and evaluate
it step-by-step whenever we want!

> Now we have functions user can define themselves without ever touching the c code!!!

If variables and functions are both values, the environment needs to map names to nodes.
so now we can create our env entry as
```C 
typedef struct EnvEntry {
    char *key;
    Node *val;
    struct EnvEntry *next;
} EnvEntry;
```

Now our hash table maps the names (key) to the Node * which is the value.

but what *is* val?

val is a Node! but Node needs new premitive types for our val to work.
so we update our Node to:

```C 
struct Node {
    struct Node *left;
    struct Node *right;

    union {
        int literal;
        char *var;
        int index;
        char *call;
        struct Node *closure;
        struct Node *nativeFunc;
    } data;

    NodeType type;
};
```


we have nativeFuncs which point to our c funcs and closures which are user defined funcs in our lang composed of nodes

native function = opaque C implementation
closure = language-level graph representation

SO. we have assembled our pieces:

- A Memory Allocator 

- A Environment table that holds both our vars, c funcs and userdefined funcs

- An evaluator that just walks down the the program recursively and reduces them using env lookup

> lets execute our first program!!

Right now we dont have a lexer/parser yet so we can just hand construct our AST.

what better progrm to test than fibonacci sequence!


So I wrote the program. typed in fib(5).

and...


**IT CRASHED**

---
### UPGRADING THE MEMORY ALLOCATOR

why? because our fib(5) spawned 13k nodes. but our allocator size is only 1024 nodes total!
You might think "okay its obvious, reallocate the block and grow the size. have it be a dynamic array"
and thats exactly where the problem is. the thing is, all of our nodes are pointing to these memory blocks.
when we perform a reallocation with an increased size sometimes the operating system assignes the block into a new slot!
Completely breaking all of our pointers causing a segfault!

how do we tackle this problem? 

> By chaining multiple allocators togeather!

we can build a linked list of our allocated blocks. when we run out of space in one block we just allocate a new one and point to it! we dont have to move any data!

this is called a chunk allocator! each of our blocks are defined as a chunk that store the memory and we can chian them togeather like a linkedlist with a next pointer.

so we had to upgrade our allocator to a chunk allocator. now a chunk can be defined like this:

```C
typedef struct Chunk {
    Node nodes[CHUNK_SIZE];
    struct Chunk *next;
} Chunk;
```

we can keep track of this by using a current and a first!

we have:

- A Chunk Memory Allocator (new) 

- An Environment table that holds both our vars, c funcs and userdefined funcs

- An Evaluator that just walks down the the program recursively and reduces them using env lookup

> lets execute our first program again

and...


**It works!**

we get the result 5! 3+2 is 5. 

but something weird happened. it was using 1.32 mb of memory.
Thats weird, because fib(5) isnt a complex operation.

so I ran fib(10)
it took 40 mb of ram !!
so I had to test it out. I ran a benchmark.

```text
RAM (GB) vs fib(n)

12.29 GB ┤  
11.34 GB ┤                               ╭───
10.40 GB ┤                        ╭──────╯
 9.45 GB ┤                  ╭─────╯
 8.51 GB ┤              ╭───╯
 7.56 GB ┤             ╭╯
 6.62 GB ┤            ╭╯
 5.67 GB ┤           ╭╯
 4.73 GB ┤         ╭─╯
 3.78 GB ┤        ╭╯
 2.84 GB ┤       ╭╯
 1.89 GB ┤      ╭╯
 0.95 GB ┤     ╭╯
 0.00 GB ┼─────╯
         -----------------------------------
         5    10        20                  40
```

fib(40) literally took 12+ GIGABYTES before hitting an OOM and crashing. 

why? because it spawns approximately 1.3 Billion nodes.

At 48 bytes per node, 1.3 billion simultaneously-live nodes would be ~62.4 GB

The problem is. we are allocating nodes but we are never freeing them once their use is over. This is causing us to allocate more and more memory.

To tackle this problem I had to build a garbage collector.

---
### BUILDING THE GARBAGE COLLECTOR

what does it mean to collect garbage?
when we evaluate 1+1+1 the evaluator does this:

- 1. Builds the ast
```text
      (+)
      / \
    (1) (+)
        / \
      (1) (1)
```

- 2. Evaluates left. Left is already a literal. moves on to right

- 3. Right is a function. so it gets evaluated first. and we mutate the tree.

```text
      (+)
      / \
    (1) (2)
```

> But what happens to the two 1s?

They are left on the allocator! they are never freed till the end of the program! that is exactly the problem.

what we need our garbage collector to do is just walk through our chunk and mark what needs to be preserved and what dont.

Now if you notice, when a node is a literal its children dont need to be preserved! we can reuse these free nodes!

what we could do is once we mutate to a literal we just null out both the children and so those are never reached by the garbage collector never giving them a chance to be marked!

this is called the mark phase, the garbage collector starts from root and starts marking everything it encounters.

once marking is done we have the garbage collector go through our chunks and just check if a node is not marked and add it to a linked list called free list.

this free list is used when allocating a new node!


if freeList is empty ONLY THEN will we allocate a new node on top.
otherwise we just pop the head, and allocate the node at the memory address of head!

This lets us recycle our nodes effectively!!



> lets run the bench mark after applying the gc:
```
  RAM w/ GC (MB) vs fib(n)
      1.7200 MB ┼
      1.5886 MB ┤			                	╭───
      1.4571 MB ┤                          ╭────╯
      1.3257 MB ┤                    ╭─────╯
      1.1942 MB ┤		        ╭────╯
      1.0628 MB ┤             ╭─╯
      0.9314 MB ┤            ╭╯
      0.7999 MB ┤          ╭─╯
      0.6685 MB ┤         ╭╯
      0.5371 MB ┤       ╭─╯
      0.4056 MB ┤      ╭╯
      0.2742 MB ┤     ╭╯
      0.1427 MB ┤ ╭───╯
      0.0113 MB ┼─╯
                 -----------------------------------
           	     5    10        20                40
```

**LOOK AT THAT!** our ram usage went down from 12 GIGABYTES -> 1.7 MEGABYTES for fib(40)

but the thing is not yet solved. 

> fib 40 took 6 MINUTES to evalueate

why?

- The mark and sweep garbage collector we just completed is a stop the world garbage collector

- And the algorithm fib is inherently exponential. 

we can tacle both of these issues.

- We can implement a concurrent garbage collector

---
### What to expect in the next parts
This text has gone on long enough so I decided to split it into parts. in the next parts I will talk about 

- how I tackled the speed issue using TCO

- Implemented a Lexer/parser 

- Implemented an FFI to completely decouple our core c funcs from the execution engine

- Implemented a REPL

- switched from pointer dereferencing to getters using Handle like V8

- Brought scuffed encapsulation into C

- Implemented lambda funcitons

- Added local vars

- Set up stuff for a cheney's copy allocator


### What have we achieved so far?

We accomplished quite a lot here actually.

- Realised operators and operands can be an Algebraic Data Type.

- I then realized the real sum types arent the individual functions themselves but the type of data i.e. funcs, vars, literals

- Then realized that vars and funcs are not two different things but both are just data

- Implemented a graph evaluator which mutates the current node after eval

- Implemented an Environment Table which stores our env entries as data using a custom hashtable 

- Implemented a custom allocator to allocate our nodes

- Realized we have too much garbage we are never using and implemented a mark and sweep garbagge collector

- Overall I built a Graph Reduction engine

### WAIT. BUT DOES IT EVALUATE 1+1

yeah...I mean now it does now that I have a lexer/parser but after the state of the project mentioned in the blog

...you would have to build your program by had using createFunction and stuff. but once you did it, it does it!

1+1 is infact 2 according to graphLang! (￣ー￣)


anyhow, I did a bunch of work not mentioned in this post as mentioned above. checkout the repo
