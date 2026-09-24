# C++ Day 9 — Pointers, references and lifetimes: a workout

**Goal:** find and prevent the pointer bugs that C++ *doesn't* protect you from. Smart pointers
solve ownership; they don't solve **lifetime**: a reference, raw pointer, iterator or view that
outlives the object it refers to. This day is all practice.

Compile every exercise with `-g -fsanitize=address,undefined`. ASan reports most of these bugs as
`heap-use-after-free`, `stack-use-after-return` or `stack-use-after-scope`.

## The lifetime rules in one table

| Handle | Dangles when... |
|---|---|
| `T &`, `const T &` | the object is destroyed (local out of scope, container element erased or moved) |
| `T *` (raw, non-owning) | same, and when the owner (`unique_ptr`, container) releases it |
| iterator | the container reallocates or erases (rules differ per container — day 17) |
| `std::string_view` | the string is destroyed or modified (appending may reallocate) |
| `std::span` | the vector/array is destroyed or reallocates |
| lambda capturing by reference `[&]` | the lambda is called after the captured locals die |
| `this` | the object is destroyed while a member function or a stored lambda still uses it |

## Part 1: Predict the output (or the bug)

For each: what does it print, or what's wrong? Answers at the end of the part.

1.
   ```cpp
   std::vector<int> v = {1, 2, 3};
   int *p = &v[1];
   int &r = v[2];
   *p = 20;
   r = 30;
   std::cout << v[0] << v[1] << v[2] << '\n';
   ```

2.
   ```cpp
   std::vector<int> v = {1, 2, 3};
   v.reserve(3);
   int &first = v[0];
   v.push_back(4);
   std::cout << first << '\n';
   ```

3.
   ```cpp
   auto a = std::make_unique<int>(5);
   int *raw = a.get();
   auto b = std::move(a);
   *raw = 7;
   std::cout << *b << ' ' << (a == nullptr) << '\n';
   ```

4.
   ```cpp
   std::string_view sv;
   {
       std::string s = "temporary text that is long enough to be on the heap";
       sv = s;
   }
   std::cout << sv << '\n';
   ```

5.
   ```cpp
   std::string name = "Ada";
   std::string_view first = name;
   name += " Lovelace, Countess of Lovelace, mathematician";
   std::cout << first << '\n';
   ```

6.
   ```cpp
   auto sp1 = std::make_shared<int>(1);
   auto sp2 = sp1;
   std::weak_ptr<int> w = sp1;
   sp1.reset();
   std::cout << sp2.use_count() << ' ' << w.expired() << '\n';
   sp2.reset();
   std::cout << w.expired() << '\n';
   ```

7.
   ```cpp
   const std::string &longer(const std::string &a, const std::string &b)
   {
       return a.size() >= b.size() ? a : b;
   }
   /* in main: */
   std::string x = "hello";
   const std::string &r1 = longer(x, "hi");
   std::cout << r1 << '\n';
   const std::string &r2 = longer("hi", "hello!");
   std::cout << r2 << '\n';
   ```

8.
   ```cpp
   std::function<int()> make_counter()
   {
       int count = 0;
       return [&count]() { return ++count; };
   }
   /* in main: */
   auto c = make_counter();
   std::cout << c() << '\n';
   ```

<details><summary>Answers</summary>

1. `12030` — no reallocation happened, so `p` and `r` are fine. (It prints 1, 20, 30 with no
   spaces.)
2. **Undefined behavior**. `reserve(3)` does nothing (capacity is already ≥ 3), so `push_back` must
   reallocate and `first` dangles. ASan: heap-use-after-free.
3. `7 1` — `raw` points to the object, which still lives (now owned by `b`). Moving a `unique_ptr`
   moves ownership, not the object. This one is fine.
4. **Undefined behavior**: `sv` refers to `s`'s buffer, destroyed at the `}`.
5. **Undefined behavior**: appending made `name` reallocate its buffer; `first` still points to the
   old one. (With a short string, the small-string optimization may hide the bug — never rely on
   that.)
6. `1 0` then `1` — `w` doesn't keep the int alive; it expires when the last `shared_ptr` is reset.
7. `hello` for `r1` (fine: `x` lives). `r2` is **undefined behavior**: both arguments were
   temporaries created from literals, destroyed at the end of the full statement, and `r2` refers to
   one of them. Returning a reference to a parameter is a classic dangling trap.
8. **Undefined behavior**: the lambda captures `count` by reference, and `count` died when
   `make_counter` returned. Capture by value with a `mutable` lambda: `[count]() mutable { return ++count; }`.

</details>

## Part 2: Find and fix

Each program has one lifetime bug. Explain it, confirm with ASan, and fix it — preferably by
changing the *design*, not just patching the line.

1.
   ```cpp
   // BUG
   #include <iostream>
   #include <string>
   #include <vector>
   struct Player { std::string name; int score; };
   int main()
   {
       std::vector<Player> players = {{"Ann", 3}};
       Player &best = players[0];
       for (int i = 0; i < 100; ++i) {
           players.push_back({"bot", i});
       }
       std::cout << best.name << '\n';
   }
   ```

2.
   ```cpp
   // BUG
   #include <iostream>
   #include <vector>
   int main()
   {
       std::vector<int> v = {1, 2, 3, 4, 5, 6};
       for (auto it = v.begin(); it != v.end(); ++it) {
           if (*it % 2 == 0) {
               v.erase(it);
           }
       }
       for (int x : v) std::cout << x << ' ';
   }
   ```

3.
   ```cpp
   // BUG
   #include <iostream>
   #include <memory>
   #include <vector>
   struct Widget { int id; };
   int main()
   {
       std::vector<std::unique_ptr<Widget>> widgets;
       widgets.push_back(std::make_unique<Widget>(Widget{1}));
       Widget *selected = widgets[0].get();
       widgets.clear();
       std::cout << selected->id << '\n';
   }
   ```

4.
   ```cpp
   // BUG
   #include <iostream>
   #include <string>
   #include <string_view>
   std::string_view file_extension(const std::string &path)
   {
       return std::string_view{path}.substr(path.rfind('.'));
   }
   int main()
   {
       auto ext = file_extension(std::string{"report.pdf"});
       std::cout << ext << '\n';
   }
   ```

5.
   ```cpp
   // BUG
   #include <functional>
   #include <iostream>
   #include <vector>
   class Button {
   public:
       void on_click(std::function<void()> f) { handlers_.push_back(std::move(f)); }
       void click() { for (auto &h : handlers_) h(); }
   private:
       std::vector<std::function<void()>> handlers_;
   };
   class Counter {
   public:
       void attach(Button &b) { b.on_click([this] { ++clicks_; std::cout << clicks_ << '\n'; }); }
   private:
       int clicks_ = 0;
   };
   int main()
   {
       Button button;
       {
           Counter c;
           c.attach(button);
       }
       button.click();
   }
   ```

<details><summary>Answers</summary>

1. `best` refers into the vector, which reallocates during the pushes. Fix: store an index
   (`std::size_t best = 0;`) — indexes survive reallocation — or re-find the element after
   modifying the vector.
2. `erase` invalidates `it` (and everything after it), then `++it` uses it. Fix:
   `std::erase_if(v, [](int x) { return x % 2 == 0; });` (C++20) — or `it = v.erase(it);` without
   `++it` in that branch.
3. `clear()` destroyed the widget; `selected` dangles. The owner decides lifetime: either don't keep
   raw pointers past operations that can delete, or look the widget up by `id` when needed, or use
   `shared_ptr`/`weak_ptr` if the widget really has shared users.
4. The argument is a temporary `std::string`, destroyed at the end of the statement that called
   `file_extension`; `ext` points into it. Fix: return `std::string`, or take and return
   `std::string_view` and let callers keep the original alive (document it!).
5. The lambda captures `this`; `c` is destroyed, then the button calls the lambda: use-after-scope.
   Fix: detach in `Counter`'s destructor (needs a handle/ID for the handler), make the button not
   outlive the counter, or have the handler hold a `std::weak_ptr` to a shared counter and check
   it.

</details>

## Part 3: Write it

1. **A linked list with unique_ptr ownership and raw-pointer navigation.** Implement:

   ```cpp
   class IntList {
   public:
       void push_front(int v);
       void push_back(int v);        // keep a raw Node *tail_ for O(1): non-owning!
       bool remove(int v);           // careful: may invalidate tail_
       void reverse();
       std::size_t size() const;
       void print() const;
       ~IntList();                   // iterative, to avoid deep recursion
   private:
       struct Node { int value; std::unique_ptr<Node> next; };
       std::unique_ptr<Node> head_;
       Node *tail_ = nullptr;
   };
   ```

   The `unique_ptr`s own the nodes; `tail_` just observes. The hard part is `remove` and `reverse`
   keeping `tail_` correct, and moving `unique_ptr`s around with `std::move`. Test with ASan.

2. **A tree with parent pointers.** `struct TreeNode { int key; std::vector<std::unique_ptr<TreeNode>> children; TreeNode *parent = nullptr; };`
   Write `add_child`, `path_to_root` (follow `parent` pointers), and `remove_subtree`. Which
   pointers can dangle, and when?

3. **Observer with weak_ptr.** A `Newsletter` keeps `std::vector<std::weak_ptr<Subscriber>>`. On
   `publish`, it locks each; expired ones are removed. Subscribers are owned elsewhere with
   `shared_ptr`. Show that destroying a subscriber doesn't crash the newsletter.

4. **Ownership in signatures.** For each function, choose the parameter type and justify it:
   (a) print a `Document`; (b) add a `Page` to a `Document` that will own it; (c) set an optional
   highlight color; (d) let a cache share an `Image` with its users; (e) normalize a `Document`
   in place.

   <details><summary>Suggested answers</summary>

   (a) `const Document &` · (b) `std::unique_ptr<Page>` by value (or `Page` by value if it's
   movable and doesn't need to be on the heap) · (c) `const Color *` or `std::optional<Color>` ·
   (d) `std::shared_ptr<Image>` · (e) `Document &`

   </details>

## Check yourself

1. Smart pointers solve ownership. What problem do they not solve?
2. Why is an index into a vector often safer to keep than a reference or pointer?
3. How does `std::erase_if` avoid the iterator invalidation bug?
4. When is returning a `const T &` from a function safe?
5. What's dangerous about capturing `this` or `[&]` in a lambda that's stored?
