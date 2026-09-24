# C Day 25 — Hash tables

**Goal:** understand hashing and build a string → int hash table with separate chaining that
resizes itself. You'll reuse this code in the C++ course (C++ day 21).

## Concepts

### The idea

An array gives O(1) access by *index*. A hash table gives (average) O(1) access by *key* — like a
string — by turning the key into an index with a **hash function**:

```text
 "apple"  --hash-->  2894715  --% 8-->  bucket 3
```

Two different keys can land in the same bucket: a **collision**. With **separate chaining**, each
bucket holds a linked list of the entries that landed there:

```text
 buckets
 [0] NULL
 [1] --> ("pear", 2) --> NULL
 [2] NULL
 [3] --> ("apple", 5) --> ("kiwi", 1) --> NULL      two keys collided here
 [4] NULL
 ...
```

To look up a key: hash it, go to its bucket, walk the (short) list comparing keys with `strcmp`.

### A good hash function

It should be fast and spread similar keys across buckets. **FNV-1a** is simple and decent:

```c
#include <stdint.h>

uint64_t hash_string(const char *s)
{
    uint64_t h = 14695981039346656037u;       // FNV offset basis
    for (; *s; s++) {
        h ^= (unsigned char)*s;
        h *= 1099511628211u;                  // FNV prime
    }
    return h;
}
```

(Don't use such functions for keys chosen by an attacker — they can craft collisions. Hash tables
in servers use keyed hashes like SipHash.)

### Load factor and resizing

The **load factor** is `count / bucket_count`, the average chain length. Keep it below about 0.75
and chains stay very short. When it's exceeded, allocate twice as many buckets and **rehash** every
entry into them (its bucket index changes, since it's `hash % bucket_count`). Like `realloc`-doubling
arrays, this keeps inserts O(1) on average.

### The implementation

`hashmap.h`:

```c
#ifndef HASHMAP_H
#define HASHMAP_H

#include <stddef.h>

typedef struct HashMap HashMap;     // opaque: users can't see inside (more on day 27)

HashMap *hm_create(void);
void     hm_destroy(HashMap *m);
int      hm_put(HashMap *m, const char *key, int value);        // insert or update; 0 = OK
int      hm_get(const HashMap *m, const char *key, int *out);   // 1 = found
int      hm_remove(HashMap *m, const char *key);                // 1 = removed
size_t   hm_count(const HashMap *m);

#endif
```

`hashmap.c`:

```c
#include "hashmap.h"

#include <stdint.h>
#include <stdlib.h>
#include <string.h>

typedef struct Entry {
    char *key;              // owned copy
    int value;
    struct Entry *next;
} Entry;

struct HashMap {
    Entry **buckets;        // array of bucket_count list heads
    size_t bucket_count;
    size_t count;
};

static uint64_t hash_string(const char *s)
{
    uint64_t h = 14695981039346656037u;
    for (; *s; s++) {
        h ^= (unsigned char)*s;
        h *= 1099511628211u;
    }
    return h;
}

HashMap *hm_create(void)
{
    HashMap *m = malloc(sizeof *m);
    if (m == NULL) return NULL;
    m->bucket_count = 8;
    m->count = 0;
    m->buckets = calloc(m->bucket_count, sizeof *m->buckets);   // all NULL
    if (m->buckets == NULL) {
        free(m);
        return NULL;
    }
    return m;
}

void hm_destroy(HashMap *m)
{
    if (m == NULL) return;
    for (size_t i = 0; i < m->bucket_count; i++) {
        Entry *e = m->buckets[i];
        while (e != NULL) {
            Entry *next = e->next;
            free(e->key);
            free(e);
            e = next;
        }
    }
    free(m->buckets);
    free(m);
}

static int resize(HashMap *m, size_t new_count)
{
    Entry **fresh = calloc(new_count, sizeof *fresh);
    if (fresh == NULL) return -1;
    for (size_t i = 0; i < m->bucket_count; i++) {
        Entry *e = m->buckets[i];
        while (e != NULL) {                     // move each entry to its new bucket
            Entry *next = e->next;
            size_t b = hash_string(e->key) % new_count;
            e->next = fresh[b];
            fresh[b] = e;
            e = next;
        }
    }
    free(m->buckets);
    m->buckets = fresh;
    m->bucket_count = new_count;
    return 0;
}

int hm_put(HashMap *m, const char *key, int value)
{
    size_t b = hash_string(key) % m->bucket_count;
    for (Entry *e = m->buckets[b]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {         // key exists: update
            e->value = value;
            return 0;
        }
    }
    if ((m->count + 1) * 4 > m->bucket_count * 3) {   // load factor would exceed 0.75
        if (resize(m, m->bucket_count * 2) != 0) return -1;
        b = hash_string(key) % m->bucket_count;
    }
    Entry *e = malloc(sizeof *e);
    if (e == NULL) return -1;
    size_t len = strlen(key) + 1;
    e->key = malloc(len);
    if (e->key == NULL) {
        free(e);
        return -1;
    }
    memcpy(e->key, key, len);
    e->value = value;
    e->next = m->buckets[b];                    // push to the front of the chain
    m->buckets[b] = e;
    m->count++;
    return 0;
}

int hm_get(const HashMap *m, const char *key, int *out)
{
    size_t b = hash_string(key) % m->bucket_count;
    for (const Entry *e = m->buckets[b]; e != NULL; e = e->next) {
        if (strcmp(e->key, key) == 0) {
            *out = e->value;
            return 1;
        }
    }
    return 0;
}

int hm_remove(HashMap *m, const char *key)
{
    size_t b = hash_string(key) % m->bucket_count;
    for (Entry **link = &m->buckets[b]; *link != NULL; link = &(*link)->next) {   // day 21 trick
        if (strcmp((*link)->key, key) == 0) {
            Entry *doomed = *link;
            *link = doomed->next;
            free(doomed->key);
            free(doomed);
            m->count--;
            return 1;
        }
    }
    return 0;
}

size_t hm_count(const HashMap *m)
{
    return m->count;
}
```

A test, `main.c`:

```c
#include <assert.h>
#include <stdio.h>

#include "hashmap.h"

int main(void)
{
    HashMap *m = hm_create();
    assert(m != NULL);

    char key[16];
    for (int i = 0; i < 1000; i++) {
        snprintf(key, sizeof key, "key%d", i);
        assert(hm_put(m, key, i) == 0);
    }
    assert(hm_count(m) == 1000);

    int v;
    assert(hm_get(m, "key500", &v) == 1 && v == 500);
    assert(hm_put(m, "key500", -1) == 0 && hm_count(m) == 1000);   // update, not insert
    assert(hm_get(m, "key500", &v) == 1 && v == -1);
    assert(hm_remove(m, "key500") == 1);
    assert(hm_get(m, "key500", &v) == 0);
    assert(hm_count(m) == 999);

    hm_destroy(m);
    printf("all tests passed\n");
    return 0;
}
```

Build with `gcc -std=c17 -Wall -Wextra -g -fsanitize=address,undefined hashmap.c main.c -o hm`.

## Common mistakes

- Storing the caller's `key` pointer instead of a copy — it may be a stack buffer that's reused
  (like `key` in the test).
- Comparing keys with `==`.
- Forgetting to rehash on resize (indexes depend on the bucket count).
- Not freeing keys in `destroy`/`remove`.

## Exercises

1. Type in the hash map (all three files) and make the test pass cleanly under ASan.
2. Add `void hm_foreach(const HashMap *m, void (*fn)(const char *key, int value, void *user), void *user)`.
   The `void *user` lets the callback carry state — use it to sum all values.
3. **Word frequencies:** read a text file, split it into lowercase words, count each with the hash
   map, then copy the entries into an array, sort by count (`qsort`), and print the top 10. Try it
   on a big text from Project Gutenberg.
4. Add instrumentation: print the longest chain length and the number of empty buckets. Then
   replace FNV-1a with a bad hash (`return s[0];`) and compare. What happens to lookups?
5. **Timing:** insert 10⁶ keys, then look up 10⁶ keys; compare with a linear search in an array of
   structs for 10⁴ keys (don't try 10⁶ linear!).
6. ★ Make the value type generic: store `void *value`, and let `hm_create` take an optional
   `void (*free_value)(void *)` called on removal and destroy.
7. ★ Implement **open addressing** with linear probing instead of chaining: one array of entries;
   on collision, try the next slot. Removal needs "tombstones" — read about why.

## Check yourself

1. What's a collision, and how does chaining handle it?
2. What's the load factor, and why keep it below ~0.75?
3. Why must entries be rehashed when the table grows?
4. Why does the table copy the key string?
5. What's the average and the worst-case cost of a lookup?
