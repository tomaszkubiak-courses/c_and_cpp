# C Day 22 — Stacks, queues, ring buffers and doubly linked lists

**Goal:** implement the classic container types, understand their operations' costs, and use
them to solve problems.

## Concepts

### Abstract data types

A **stack** and a **queue** are defined by their *operations*, not by how they're stored:

| Type | Add | Remove | Order |
|---|---|---|---|
| Stack | `push` (top) | `pop` (top) | last in, first out (LIFO) — a pile of plates |
| Queue | `enqueue` (back) | `dequeue` (front) | first in, first out (FIFO) — a line at a shop |
| Deque | both ends | both ends | double-ended queue |

Each can be built on an array or a linked list. Choose by cost: all these operations should be
O(1).

### Stack on a dynamic array

The top of the stack is the end of the array, so push and pop never shift anything:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    char *items;
    size_t size, capacity;
} CharStack;

int stack_push(CharStack *s, char c)
{
    if (s->size == s->capacity) {
        size_t cap = s->capacity ? s->capacity * 2 : 16;
        char *p = realloc(s->items, cap);
        if (p == NULL) return -1;
        s->items = p;
        s->capacity = cap;
    }
    s->items[s->size++] = c;
    return 0;
}

int stack_pop(CharStack *s, char *out)
{
    if (s->size == 0) return -1;
    *out = s->items[--s->size];
    return 0;
}

// Returns 1 if every (, [ and { is closed in the right order.
int balanced(const char *text)
{
    CharStack s = {0};
    int ok = 1;
    for (const char *p = text; *p && ok; p++) {
        char open;
        switch (*p) {
        case '(': case '[': case '{':
            if (stack_push(&s, *p) != 0) ok = 0;
            break;
        case ')': ok = stack_pop(&s, &open) == 0 && open == '('; break;
        case ']': ok = stack_pop(&s, &open) == 0 && open == '['; break;
        case '}': ok = stack_pop(&s, &open) == 0 && open == '{'; break;
        }
    }
    ok = ok && s.size == 0;
    free(s.items);
    return ok;
}

int main(void)
{
    const char *tests[] = {"(a[b]{c})", "(]", "((", "f(x[i]) + {y}"};
    for (size_t i = 0; i < 4; i++) {
        printf("%-16s %s\n", tests[i], balanced(tests[i]) ? "balanced" : "NOT balanced");
    }
    return 0;
}
```

Stacks are behind function calls (the call stack), undo, expression evaluation, and depth-first
search.

### Queue on a linked list with a tail pointer

Enqueue at the tail, dequeue at the head — both O(1) if you keep a pointer to each end:

```c
typedef struct QNode {
    int value;
    struct QNode *next;
} QNode;

typedef struct {
    QNode *head, *tail;
} Queue;

int enqueue(Queue *q, int value)
{
    QNode *n = malloc(sizeof *n);
    if (n == NULL) return -1;
    n->value = value;
    n->next = NULL;
    if (q->tail != NULL) q->tail->next = n;
    else q->head = n;          // queue was empty
    q->tail = n;
    return 0;
}

int dequeue(Queue *q, int *out)
{
    if (q->head == NULL) return -1;
    QNode *n = q->head;
    *out = n->value;
    q->head = n->next;
    if (q->head == NULL) q->tail = NULL;   // queue became empty
    free(n);
    return 0;
}
```

### Ring buffer (circular buffer)

A fixed-size array used as a queue, with two indexes that wrap around. No allocation per element —
it's how audio buffers, keyboard input and network drivers work:

```text
 capacity 8, head = 6, count = 4:
 index:  0   1   2   3   4   5   6   7
       [ c ][ d ][   ][   ][   ][   ][ a ][ b ]
                                     ^head        the next element goes to (head + count) % 8 = 2
```

```c
#define RB_CAP 8

typedef struct {
    int items[RB_CAP];
    size_t head;    // index of the oldest element
    size_t count;
} RingBuffer;

int rb_push(RingBuffer *rb, int value)
{
    if (rb->count == RB_CAP) return -1;          // full
    rb->items[(rb->head + rb->count) % RB_CAP] = value;
    rb->count++;
    return 0;
}

int rb_pop(RingBuffer *rb, int *out)
{
    if (rb->count == 0) return -1;               // empty
    *out = rb->items[rb->head];
    rb->head = (rb->head + 1) % RB_CAP;
    rb->count--;
    return 0;
}
```

### Doubly linked list

Each node points both ways, so you can remove a node in O(1) given only a pointer to it, and walk
in both directions:

```c
typedef struct DNode {
    int value;
    struct DNode *prev, *next;
} DNode;
```

The standard trick to avoid special cases is a **sentinel** node: a dummy node that is always
there, whose `next` is the first real node and whose `prev` is the last. An empty list is a
sentinel pointing to itself. Then insert and remove never check for NULL:

```c
void dlist_insert_after(DNode *pos, DNode *n)
{
    n->prev = pos;
    n->next = pos->next;
    pos->next->prev = n;
    pos->next = n;
}

void dlist_unlink(DNode *n)
{
    n->prev->next = n->next;
    n->next->prev = n->prev;
}
```

Draw these four pointer assignments for inserting into an empty (sentinel-only) list and into a
list of two nodes. The order of the assignments matters in `insert_after` — try swapping two
lines on paper and see what breaks.

## Common mistakes

- Forgetting to reset `tail` when a queue becomes empty.
- Off-by-one in ring buffers: confusing "full" and "empty" when both have `head == tail`. Keeping
  a `count` avoids this.
- Popping an empty stack.
- In a doubly linked list, updating `next` but not `prev` (or in the wrong order).

## Exercises

1. Type in `balanced` and extend it to report the position of the first error.
2. **RPN calculator:** evaluate Reverse Polish Notation expressions like `3 4 + 2 *` (= 14) with an
   int stack. Tokens are separated by spaces; report errors (too few operands, leftovers).
3. Complete the linked-list `Queue` with `queue_free` and use it to simulate a print queue:
   read commands `add NAME` / `print` / `quit` from stdin.
4. Complete the ring buffer, and write `rb_push_overwrite` that drops the oldest element when full
   (how many logs do: "keep the last N"). Test the wrap-around carefully.
5. Implement a **stack with a linked list** instead of an array. Compare the two versions: which
   allocates more? Which is simpler?
6. Implement a doubly linked list with a sentinel: `dlist_init`, `push_front`, `push_back`,
   `pop_front`, `pop_back`, `print_forward`, `print_backward`, `free`.
7. **Undo:** keep a text line and a stack of previous versions. Commands: `set TEXT`, `undo`,
   `show`.
8. ★ Implement a **queue with two stacks** (push to one, pop from the other, moving everything
   across when the second is empty). Why is each operation still O(1) *on average*?

## Check yourself

1. What does LIFO mean, and which structure is LIFO?
2. Why does a linked-list queue need a tail pointer?
3. How does a ring buffer compute where the next element goes?
4. What's a sentinel node, and what special cases does it remove?
