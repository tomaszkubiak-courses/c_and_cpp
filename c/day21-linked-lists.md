# C Day 21 — Linked lists

**Goal:** build a singly linked list, and use it to master pointers to structs and pointers to
pointers. This is the classic pointer training ground — take your time and draw every step.

## Concepts

### Nodes connected by pointers

An array stores elements side by side. A **linked list** stores each element in its own heap
block — a **node** — which holds a pointer to the next node:

```c
typedef struct Node {
    int value;
    struct Node *next;     // the struct refers to itself, so it needs the tag "Node"
} Node;
```

```text
 head
 [ ●-]--> [ 3 | ●-]--> [ 7 | ●-]--> [ 9 | NULL ]
```

The list is represented by a pointer to the first node, `head`. An empty list is `head == NULL`.
The last node's `next` is `NULL`.

Compared with an array: inserting or removing at a known position is O(1) (just relink pointers,
no shifting), but reaching the i-th element means walking i nodes, and every node costs an
allocation. In practice, arrays are faster for most jobs (cache locality — C++ day 17) — but
linked lists teach you pointers like nothing else, and they're everywhere in kernels and
embedded code.

### Building blocks

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int value;
    struct Node *next;
} Node;

Node *node_new(int value, Node *next)
{
    Node *n = malloc(sizeof *n);
    if (n == NULL) {
        return NULL;
    }
    n->value = value;
    n->next = next;
    return n;
}

// Push to the front: the caller's head changes, so we need Node **.
int push_front(Node **head, int value)
{
    Node *n = node_new(value, *head);
    if (n == NULL) {
        return -1;
    }
    *head = n;
    return 0;
}

void print_list(const Node *head)
{
    for (const Node *p = head; p != NULL; p = p->next) {
        printf("%d -> ", p->value);
    }
    printf("NULL\n");
}

void free_list(Node **head)
{
    Node *p = *head;
    while (p != NULL) {
        Node *next = p->next;   // save next BEFORE freeing p
        free(p);
        p = next;
    }
    *head = NULL;
}

int main(void)
{
    Node *head = NULL;
    for (int i = 1; i <= 4; i++) {
        push_front(&head, i * 10);
    }
    print_list(head);        // 40 -> 30 -> 20 -> 10 -> NULL
    free_list(&head);
    print_list(head);        // NULL
    return 0;
}
```

Trace `push_front(&head, 5)` on an empty list and then again on the result, drawing `head`, `n`
and the arrows at each line. Why does `push_front` take `Node **`? Because it must change `main`'s
`head` variable — the day 11 rule.

### Walking: the loop you'll write a hundred times

```c
for (Node *p = head; p != NULL; p = p->next) { /* use p->value */ }
```

### The pointer-to-pointer trick: no special cases

Removing a node normally needs a "previous" pointer and a special case for removing the head:

```c
// The long way: two cases
int remove_value_long(Node **head, int value)
{
    Node *prev = NULL, *cur = *head;
    while (cur != NULL && cur->value != value) {
        prev = cur;
        cur = cur->next;
    }
    if (cur == NULL) return 0;          // not found
    if (prev == NULL) *head = cur->next; // removing the first node
    else prev->next = cur->next;
    free(cur);
    return 1;
}
```

Look at what really changes: some **pointer variable** that currently points to the node — either
`head` or some node's `next` field — must be redirected. So walk a pointer to *that pointer*:

```c
// The elegant way: link points at the pointer that points at the current node
int remove_value(Node **head, int value)
{
    Node **link = head;
    while (*link != NULL && (*link)->value != value) {
        link = &(*link)->next;           // move to the next node's "next" field
    }
    if (*link == NULL) return 0;          // not found
    Node *doomed = *link;
    *link = doomed->next;                 // bypass the node — works for head or middle
    free(doomed);
    return 1;
}
```

```text
 head          node A              node B
 [ ●-]-->   [ 3 | next ●-]-->   [ 7 | next ●-]--> ...
   ^                  ^
   link starts here   after one step, link points at A's next field
```

This is worth understanding deeply. Draw it for removing the first, a middle, and the last node,
and for a value that isn't there. The same trick gives a clean `insert_sorted`.

### Reversing a list

```c
void reverse(Node **head)
{
    Node *prev = NULL, *cur = *head;
    while (cur != NULL) {
        Node *next = cur->next;   // 1. remember the rest
        cur->next = prev;         // 2. point this node backwards
        prev = cur;               // 3. advance both
        cur = next;
    }
    *head = prev;
}
```

Draw three nodes and run it by hand. Every pointer interview question starts here.

## Common mistakes

- Using a node after freeing it (`free(p); p = p->next;`) — save `next` first.
- Losing the rest of the list by overwriting `next` before saving it.
- Taking `Node *head` in functions that must change the head.
- Forgetting the empty-list case.
- `(*link)->next` vs `*link->next` — the second means `*(link->next)`, which doesn't compile here
  because `link` isn't a pointer to a struct.

## Exercises

Draw each operation before coding it, and test each on an empty list, a one-node list and a longer
list. Run everything under `-fsanitize=address`.

1. `size_t list_length(const Node *head)` — iterative, then recursive.
2. `int push_back(Node **head, int value)` — walk to the last `next` field with the `link` trick,
   then attach.
3. `int insert_sorted(Node **head, int value)` — keep the list ascending. Use the `link` trick.
4. `Node *find(Node *head, int value)`.
5. `int pop_front(Node **head, int *out)`.
6. `void reverse(Node **head)` — type it in and trace it. ★ Then write it recursively.
7. `int nth_value(const Node *head, size_t n, int *out)`.
8. `Node *middle(Node *head)` — using two pointers, one moving 1 step and one moving 2 steps.
9. `void remove_all(Node **head, int value)` — removes every occurrence, with the `link` trick.
10. `Node *merge_sorted(Node *a, Node *b)` — merge two sorted lists into one by relinking nodes (no
    new allocation).
11. ★ `int has_cycle(const Node *head)` — Floyd's "tortoise and hare": a slow and a fast pointer; if
    they ever meet, there's a cycle. Test by linking the last node back to the second one (and
    break the cycle before freeing!).
12. ★ Work through problems from Nick Parlante's *Linked List Problems*
    (<http://cslibrary.stanford.edu/105/>) — at least `CountTest`, `InsertNth`, `SortedInsert`,
    `RemoveDuplicates` and `FrontBackSplit`.

## Check yourself

1. How do you represent an empty list?
2. Why does `push_front` need a `Node **`?
3. In `remove_value`, what does `link` point to, exactly?
4. Why must you save `p->next` before `free(p)`?
5. What are the costs of accessing the i-th element in an array and in a linked list?
