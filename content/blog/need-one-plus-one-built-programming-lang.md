---
title: "Needed 1+1, Built a Functional Programming Language"
date: 2026-09-16T06:12:00+05:30
description: "A deep dive into why I spent a weekend building an AST parser from scratch just todo basic math."
tags: ["compilers", "c", "weekend-project", "functional programming", "graph reduction"]
categories: ["programming", "tech", "PL"]
draft: false
---

### THE DATA STRUCTURES ASSIGNMENT

So, My prof taught us how to convert an expression into a binary tree.
Naturally, I wondered how we would evaluate such trees. I concluded you 
would recursively check the root node, the rootnodes are operators and left
and right are its operands. once you evaluate stuff and you can replace the 
current root node with evaluated value.

So I assigned myself to work of writing an evaluator for such expressions.
I constructed the tree for 1+1+1 which could translate to 

```

     (+)
    /   \
  (+)   (1)
  / \
(1) (1)
```

And so I needed to solve this. pretty easy you would think. and then the functional programming brain virus took over.

(´･ω･`)

so then I realized we are just performing actions based on the current expressions so I defined them as a sum type of all the arithematic operations

```
Expr ::= Add Expr Expr
       | Sub Expr Expr
       | Mul Expr Expr
       | Div Expr Expr
       | Val Expr Expr
```

and this Expr takes left and right as arguments. if left is a literal it is taken as an argument and if its an operator it is evaluated to a literal first
And then. 

I had an epipheny 
(|'o'|).

We dont need ADD/SUB/MUL/DIV the evaluator doesnt need to know what it is evaluating as long as it takes two Exprs and returns an Expr

so now it can be rewritten as:

```
Expr ::= Func Expr Expr
       | Val 
```

So now our evaluator's job is pretty simple. just:


- See Node
- If its a func, go to left
- If left is a literal use that as arg
- If not evaluate till you get literal
- See right
- Do the same
- Both are literals 
- Replace root func with literal

ヽ(´ー｀)ﾉ
### Oh wait. you can add vars to this. it wouldnt be a big change 
( ﾟヮﾟ)	

Should be a one tiny addition no problem whatsoever. I mean variables are just a hashtable lookup that hold an Expr



oh wait. C doesnt have built in hashtables.

hmmm.（´-`）.｡oO( ... )

> Lets just implement a hashtable. its small change! m9(・∀・)

soo....how does that work? I never implemeneted it before.
I look it up on google like a caveman and find this amazing text

[How to implement a hash table (in C)](https://benhoyt.com/writings/hash-table-in-c/)

It was pretty easy to implement. Its not that difficult once you get it. 
(tbf my implementation is pretty basic I dont have those Red Black self balacncing BST its just a linkedlist on collisions, Though I gotta add that to my todo I will implement those)

So now we just got a little change in the Expr:

```
Expr ::= Func Expr Expr
       | Val 
       | Var 
```

---

### Actually implementing it in C

alright  then time to code in C with this plan, seems simple enough. just a tagged union.


```C
typedef enum {
    LITERAL,
    VAR,
    CALL,
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

But heres the thing. when you malloc() a node malloc adds a header which takes up **16 bytes** of memory!
bringing our total per node to 32(node size) + 16 (malloc header) = **48 bytes!**

if we add recursion or any sort of more complex functions other than 1+1 we will be looking at hundreds of thousands to even billions of nodes.

malloc also sends a request to the operating system everythime we want to allocate a node. if we are doing this billions of times this is just not sustainable.

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

I wrote the allocator and then defined a c func:

Once our simple allocator was done we could easily allocate nodes by calling
```C 
Node *allocNode();
```

It was great! Now moving on to actually calling the functions.
Then I realized

> We can store the vars and funcs into the same struct to get higherorder functions for free! (°◇°)

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
When a user types code into our language, our parser
reads text. It can't magically compile that text into
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

so now we can create our env entry as
```C 
typedef struct EnvEntry {
    char *key;
    Node *val;
    struct EnvEntry *next;
} EnvEntry;
```

we obtain the key using a very simple hashing function called djb2 
obtain the key, look inside and get the val. simple.

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

SO. we have:

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

we can keep track of this by using a current and a first!(basically a head and a tail of an LL)

we have:

- A Chunk Memory Allocator (new) 

- A Environment table that holds both our vars, c funcs and userdefined funcs

- An evaluator that just walks down the the program recursively and reduces them using env lookup

> lets execute our first program again

and...


**It works!**

we get the result 5! 3+2 is 5. 

but something weird happened. it was using 1.32 mb of memory.
Thats weird, because fib(5) isnt a complex operation.

so I rean fib(10)
it took 40 mb of ram !!
so I had to test it out. I ran a benchmark.

```
RAM (GB) vs fib(n)

12.29 GB ┤  
11.34 GB ┤    				             ╭───
10.40 GB ┤			              ╭──────╯
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

fib(40) literally took 12+ GIGABYTES of memory. why? because it spawns approximately 1 Billion nodes.

The problem is. we are allocating nodes but we are never freeing them once their use is over. This is causing us to allocate more and more memory.

To tackle this problem I had to build a garbage collector.

---
### BUILDING THE GARBAGE COLLECTOR

what does it mean to collect garbage?
when we evaluate 1+1+1 the evaluator does this:

- 1. Builds the ast
```
    (+)
    / \
   (1)(+)
      / \
     (1)(1)
```

- 2. Evaluates left. Left is already a literal. moves on to right

- 3. Right is a function. so it gets evaluated first. and we mutate the tree.

```
    (+)
    / \
  (1) (2)
```

> But what happens to the two 1s?

They are left on the allocator! they are never freed till the end of the program! that is exactly the problem.

what we need our allocator to do is just walk through our chunk and mark what needs to be preserved and what dont.

Now if you notice, when a node is a literal its children dont need to be preserved! we can reuse these free nodes!

what we could do is once we mutate to a literal we just null out both the children and so those are never reached by the garbage collector never giving them a chance to be marked!

this is called the mark phase, the garbage collector starts from root and starts marking everything it encounters.

once marking is done we have the garbage collector go through our chunks and just check if a node is not marked and add it to a linked list called free list.

this free list is used when allocating a new node!


if freeList is empty ONLY THEN will we allocate a new node on top.
otherwise we just pop the head, and allocate the node at the memory address of head!

This lets us recycle our nodes effectively!!



