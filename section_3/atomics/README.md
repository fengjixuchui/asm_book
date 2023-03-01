# Section 1 / Atomics

![Gurney Hallek](./battle_pug.jpeg)

## Threads

Suppose you run two copies of the same program at the same time. They
both have a variable named `i`. Can a change to `i` made by one copy of
the program impact the value of `i` in the other running copy of the
program? Of course not.

Think of running two copies of a program at the same time as having
two identical, but distinct, homes. What happens inside one house does
not impact what happens inside the house next door.

Threads are a different way of getting more than one "copy" of a program
to run at the same time. Threads are different, however, in that they
all live within the same household. All of the housemates share the
living space and all housemates have access to any global or shared
resource. This makes for great gains in performance for a broad class
of problems but also introduces great hazards.

Suppose you buy a carton of milk and place it in the fridge. If you were
the only member of the household you would expect that when we next went
to the fridge your milk would still be there, right? If you share the
household with other people, this might not be the case.

Consider the following program:

```c++
#include <iostream>                                           // 1 
#include <thread>                                             // 2 
#include <atomic>                                             // 3 
#include <vector>                                             // 4 
                                                              // 5 
using std::cout;                                              // 6 
using std::endl;                                              // 7 
using std::atomic;                                            // 8 
using std::vector;                                            // 9 
using std::thread;                                            // 10 
                                                              // 11 
const uint32_t MAX_LOOPS = 10000;                             // 12 
const uint32_t NUM_THREADS = 16;                              // 13 
                                                              // 14 
/*  volatile is necessary if any use of the optimizer         // 15 
    is to be made.                                            // 16 
*/                                                            // 17 
volatile uint32_t naked_int = 0;                              // 18 
atomic<uint32_t> atomic_integer(0);                           // 19 
                                                              // 20 
void NakedWorker() {                                          // 21 
    extern volatile uint32_t naked_int;                       // 22 
                                                              // 23 
    for (uint32_t i = 0; i < MAX_LOOPS; i++) {                // 24 
        naked_int++;                                          // 25 
    }                                                         // 26 
}                                                             // 27 
                                                              // 28 
void AtomicWorker() {                                         // 29 
    extern atomic<uint32_t> atomic_integer;                   // 30 
                                                              // 31 
    for (uint32_t i = 0; i < MAX_LOOPS; i++) {                // 32 
        atomic_integer++;                                     // 33 
    }                                                         // 34 
}                                                             // 35 
                                                              // 36 
void DoNaked() {                                              // 37 
    vector<thread *> threads;                                 // 38 
                                                              // 39 
    for (uint32_t i = 0; i < NUM_THREADS; i++) {              // 40 
        threads.push_back(new thread(NakedWorker));           // 41 
    }                                                         // 42 
                                                              // 43 
    for (auto &t : threads) {                                 // 44 
        t->join();                                            // 45 
    }                                                         // 46 
}                                                             // 47 
                                                              // 48 
void DoAtomic() {                                             // 49 
    vector<thread *> threads;                                 // 50 
                                                              // 51 
    for (uint32_t i = 0; i < NUM_THREADS; i++) {              // 52 
        threads.push_back(new thread(AtomicWorker));          // 53 
    }                                                         // 54 
                                                              // 55 
    for (auto &t : threads) {                                 // 56 
        t->join();                                            // 57 
    }                                                         // 58 
}                                                             // 59 
                                                              // 60 
int main() {                                                  // 61 
                                                              // 62 
    DoNaked();                                                // 63 
    DoAtomic();                                               // 64 
                                                              // 65 
    cout << "Correct sum is: ";                               // 66 
    cout << NUM_THREADS * MAX_LOOPS << endl;                  // 67 
    cout << "Naked sum: " << naked_int << endl;               // 68 
    cout << "Atomic sum: " << atomic_integer << endl;         // 69 
                                                              // 70 
    return 0;                                                 // 71 
}                                                             // 72 
perrykivolowitz@DAEDALUS atomics %
```

This program will spawn 16 threads which will each loop 10,000 times,
adding one to a zero-initialized integer each loop. At the end, when all
the threads complete, the integer should have the value 160,000.

Alas, this is an example of the class "Hidden Update" bug. The shared
resource, the integer, will get clobbered in unpredictable ways.

For example, multiple runs might produce (snipped to show only the
output from `NakedWorker()`):

- Naked sum: 74291
- Naked sum: 79390
- Naked sum: 89115
- etc

Not only are the results wrong, they are wrong in a different way each
time.

## Serializing Access to Integer Types

C++11 introduced the notion of *atomic integers*. These do not glow.
Rather, access to them is guaranteed to be atomic... as in, cannot be
broken down.

The hidden update problem's root cause is that adding (for example) to a
value in memory involves three instructions at the assembly language
level. A load, an addition, and a store. A hidden update occurs when a
thread is yanked from the CPU in the middle of these instructions. When
the thread returns to the CPU, the store causes its stale data to
overwrite (hide) correct data.

There are many ways to avoid the hidden update problem including a large
array of synchronization mechanisms. Alternatively, one can avoid the
hidden update problem by ensuring the three instruction sequence isn't
interrupted. This can be done using atomic integer types.

## Using Atomics

First, make the appropriate include:

`#include <atomic>`

Next, make the appropriate declaration and initialize the variable.
Here, replace "integral type" with some integer type:

`atomic<integral type> atomic_integer(0);`

Notice how the initial value is provided to the atomic variable's
constructor.

Finally, use the atomic variable as you would any other integer.

## A General Implementation

The newer ARM architectures provide a single instruction solution for
addition, subtraction and various bitwise operations. These will be
described below.

For ARMv8 and for later ARM versions (to perform operations other than
those listed above), there is a general solution that isn't pretty. It
is an example of Load Locked / Store Conditional. It isn't pretty
because it involves a loop.

```asm
        .text                                                 // 1 
        .p2align    2                                         // 2 
                                                              // 3 
#if defined(__APPLE__)                                        // 4 
        .global     _LoadLockedStoreConditional               // 5 
_LoadLockedStoreConditional:                                  // 6 
#else                                                         // 7 
        .global     LoadLockedStoreConditional                // 8 
LoadLockedStoreConditional:                                   // 9 
#endif                                                        // 10 
1:      ldaxr       w1, [x0]                                  // 11 
        add         w1, w1, 1                                 // 12 
        stlxr       w2, w1, [x0]                              // 13 
        cbnz        w2, 1b                                    // 14 
        ret                                                   // 15 
```

Lines 1 and 2 are boilerplate.

The conditional assembly block from line 4 through line 10 declare the
label `LoadLockedStoreConditional` as global for both Linux and Apple
assemblers. The label itself is also stated.

It is worth explaining that labels marked as global must have an
underscore prefix for Apple assembly.

This function is passed the address of an `int32_t`.

Line 11 loads the value found at that address into `w1` and also marks
the address as needing watching (by the hardware).

Line 12 can be expanded and / or replaced with whatever operation needed
to be done on the value.

Line 13 puts the potato on the fork. It is a store condition with
release. The release means that after the instruction finishes, the
previously marked address will no longer be marked. The value returned
in `w2` will be 0 of the store actually took place.

Here's the cool bit (literally): If `w2` contains a 1 it means that
some executing agent (most likely another thread in the same process)
has attempted to change the location in memory since this thread marked
the location, the store did NOT actually take place.

Imagine the following:

| T1 | T2 |
| -- | -- |
| Executes line 11. Gets value 10. Location is marked by T1. | |
| Executes line 12. `w1` goes up to 11. | |
| Yanked from CPU | |
| | Executes line 11. Gets value 10. Location is marked by T2. |
| | `w1` goes up to 11. |
| | |

## Implementation of ARMv8.1A and Newer

Implementation of operations on atomic variables 