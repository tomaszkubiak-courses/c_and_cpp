# C Day 26 — Binary search trees and recursion

**Goal:** build a binary search tree, traverse it recursively, and free it — the last big pointer
data structure of the course.

## Concepts

### Trees

A **tree** is a set of nodes where each node has children and every node except the **root** has
exactly one parent. In a **binary tree** each node has at most two children, `left` and `right`.
Nodes without children are **leaves**. The **height** is the number of edges on the longest path
from the root down to a leaf.

A **binary search tree (BST)** adds one rule: for every node, all keys in its left subtree are
smaller, and all keys in its right subtree are larger.

```text
            8
          /   \
         3     10
        / \      \
       1   6      14
          / \    /
         4   7  13
```

Searching compares with the root and goes left or right — like binary search, but the structure
supports fast insertion and removal too. In a **balanced** tree the height is about log₂ n, so
search, insert and remove are O(log n). Inserting already-sorted keys makes a BST degenerate into a
linked list (height n, O(n) operations); self-balancing trees (AVL, red-black) fix that, and
`std::map` in C++ is one.

### The node

```c
typedef struct TreeNode {
    int key;
    struct TreeNode *left, *right;
} TreeNode;
```

### Insert, search, traverse, free

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct TreeNode {
    int key;
    struct TreeNode *left, *right;
} TreeNode;

// Inserts key (ignores duplicates). Takes TreeNode ** because the root may change.
int bst_insert(TreeNode **root, int key)
{
    TreeNode **link = root;
    while (*link != NULL) {                 // the day 21 trick, now with two directions
        if (key < (*link)->key) link = &(*link)->left;
        else if (key > (*link)->key) link = &(*link)->right;
        else return 0;                      // already present
    }
    TreeNode *n = malloc(sizeof *n);
    if (n == NULL) return -1;
    n->key = key;
    n->left = n->right = NULL;
    *link = n;
    return 0;
}

const TreeNode *bst_find(const TreeNode *node, int key)
{
    while (node != NULL && node->key != key) {
        node = (key < node->key) ? node->left : node->right;
    }
    return node;
}

void print_in_order(const TreeNode *node)   // left, node, right -> sorted order
{
    if (node == NULL) return;
    print_in_order(node->left);
    printf("%d ", node->key);
    print_in_order(node->right);
}

int height(const TreeNode *node)
{
    if (node == NULL) return -1;            // empty tree: -1, single node: 0
    int l = height(node->left), r = height(node->right);
    return 1 + (l > r ? l : r);
}

void bst_free(TreeNode *node)               // post-order: children before the parent
{
    if (node == NULL) return;
    bst_free(node->left);
    bst_free(node->right);
    free(node);
}

int main(void)
{
    TreeNode *root = NULL;
    int keys[] = {8, 3, 10, 1, 6, 14, 4, 7, 13};
    for (size_t i = 0; i < sizeof keys / sizeof keys[0]; i++) {
        bst_insert(&root, keys[i]);
    }
    print_in_order(root);                   // 1 3 4 6 7 8 10 13 14
    printf("\nheight %d, has 6: %s, has 5: %s\n", height(root),
           bst_find(root, 6) ? "yes" : "no", bst_find(root, 5) ? "yes" : "no");
    bst_free(root);
    return 0;
}
```

### Recursion fits trees

A tree is a recursive structure: a node plus two smaller trees. So most tree functions are three
lines: handle the empty tree (base case), recurse left, recurse right, combine. `height` and
`bst_free` above follow exactly that shape.

### Traversal orders

| Order | Visit | Use |
|---|---|---|
| **In-order** | left, node, right | BST keys in sorted order |
| **Pre-order** | node, left, right | copying a tree; printing its structure |
| **Post-order** | left, right, node | freeing a tree (children before parent); evaluating expression trees |
| **Level-order** | level by level, left to right | uses a **queue** (day 22), not recursion |

For the tree above: pre-order is `8 3 1 6 4 7 10 14 13`; post-order is `1 4 7 6 3 13 14 10 8`.

### Removing a node: three cases

1. **Leaf** — just unlink and free it.
2. **One child** — replace the node with its child.
3. **Two children** — find the **in-order successor** (the smallest key in the right subtree), copy
   its key into this node, then remove the successor from the right subtree (it has at most one
   child, so this is case 1 or 2).

With a `TreeNode **link` to the node being removed, cases 1 and 2 are a single assignment:
`*link = (n->left != NULL) ? n->left : n->right;`.

## Common mistakes

- Freeing a node before its children (you lose them) — use post-order.
- Taking `TreeNode *root` in `insert`, so inserting into an empty tree doesn't change the caller's
  root.
- Recursion without the NULL base case.
- Testing only with random input and never with sorted input (degenerate tree).

## Exercises

1. Type in the program, then add `print_pre_order` and `print_post_order`. Check them against the
   sequences above.
2. `int bst_min(const TreeNode *root)` and `bst_max` — iterative.
3. `size_t count_nodes(const TreeNode *n)` and `size_t count_leaves(const TreeNode *n)` —
   recursive.
4. `int bst_remove(TreeNode **root, int key)` — handle all three cases. Test removing a leaf, a
   node with one child, a node with two children, and the root.
5. `void print_tree(const TreeNode *n, int depth)` — print it sideways, right subtree on top,
   indenting by depth, so you can *see* the shape.
6. Insert 1..1000 in order and print the height. Then insert them in random order (shuffle an
   array first) and print the height again. Compare with log₂(1000) ≈ 10.
7. **Level-order traversal** using a queue of `TreeNode *`.
8. `int is_bst(const TreeNode *n, long min, long max)` — check the BST property of any binary tree.
9. ★ Turn your tree into a **word counter**: keys are strings (owned copies), values are counts,
   and in-order traversal prints the words alphabetically. Compare with your hash-table version
   from day 25: which one gives sorted output for free?
10. ★ Parlante's *Binary Trees* problems (<http://cslibrary.stanford.edu/110/>): `mirror`,
    `sameTree`, `printPaths`, `hasPathSum`.

## Check yourself

1. What's the BST property?
2. Why are BST operations O(log n) when balanced and O(n) in the worst case?
3. Which traversal gives sorted output? Which one must you use for freeing?
4. How do you remove a node with two children?
5. Why does level-order traversal need a queue?
