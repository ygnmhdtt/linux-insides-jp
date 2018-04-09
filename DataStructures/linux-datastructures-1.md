Linuxカーネルにおけるデータ構造
================================================================================

双方向連結リスト
--------------------------------------------------------------------------------

Linuxカーネルは[include/linux/list.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/list.h)で、独自の双方向連結リストの実装を提供している。
我々は `Linuxカーネルにおけるデータ構造` を双方向連結リストから見ていくとしよう。
何故かと言うと、[こちら](http://lxr.free-electrons.com/ident?i=list_head)の通り、カーネル内で非常にポピュラーであるからだ。

まず第一に、[include/linux/types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/types.h)内のメインのデータ構造を見てみよう。

```C
struct list_head {
	struct list_head *next, *prev;
};
```

あなたが今まで見てきたであろう、双方向連結リストの実装とは異なっていると思わないだろうか。
例えば、[glib](http://www.gnu.org/software/libc/)では次のようになっている。

```C
struct GList {
  gpointer data;
  GList *next;
  GList *prev;
};
```

普通、連結リストは、アイテムへのポインタを保持している。しかし、Linuxカーネルのそれは、そのようになっていない。
一番の疑問は、 `リストがどこにデータを保持しているのか？` ということだ。
カーネルにおける連結リストの実際の実装は、 `割り込みリスト` なのだ。
割り込み連結リストはノードにデータを保持しない。ノードは「次のノード」と「前のノード」へのポインタと、
リストに追加されたデータの一部であるリストノードを含んでいる。
これによりデータ構造が一般的になり、エントリのデータ型については気にしない。

例を示す。

```C
struct nmi_desc {
    spinlock_t lock;
    struct list_head head;
};
```

`list_head` がカーネルでどのように使用されているのかを理解するために、いくつか例を見てみよう。
前述したように、カーネル内でリストが使われている場所は非常に多い。
試しに、miscキャラクタドライバを見てみよう。
[drivers/char/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/char/misc.c)のmiscキャラクタドライバのAPIは、
シンプルなハードウェアや仮想デバイスをハンドリングする小さなドライバをを書くのに使用される。
これらのドライバは、いくつかのメジャ番号を共有している。

```C
#define MISC_MAJOR              10
```

しかし、固有のマイナ番号も持っている。

```
ls -l /dev |  grep 10
crw-------   1 root root     10, 235 Mar 21 12:01 autofs
drwxr-xr-x  10 root root         200 Mar 21 12:01 cpu
crw-------   1 root root     10,  62 Mar 21 12:01 cpu_dma_latency
crw-------   1 root root     10, 203 Mar 21 12:01 cuse
drwxr-xr-x   2 root root         100 Mar 21 12:01 dri
crw-rw-rw-   1 root root     10, 229 Mar 21 12:01 fuse
crw-------   1 root root     10, 228 Mar 21 12:01 hpet
crw-------   1 root root     10, 183 Mar 21 12:01 hwrng
crw-rw----+  1 root kvm      10, 232 Mar 21 12:01 kvm
crw-rw----   1 root disk     10, 237 Mar 21 12:01 loop-control
crw-------   1 root root     10, 227 Mar 21 12:01 mcelog
crw-------   1 root root     10,  59 Mar 21 12:01 memory_bandwidth
crw-------   1 root root     10,  61 Mar 21 12:01 network_latency
crw-------   1 root root     10,  60 Mar 21 12:01 network_throughput
crw-r-----   1 root kmem     10, 144 Mar 21 12:01 nvram
brw-rw----   1 root disk      1,  10 Mar 21 12:01 ram10
crw--w----   1 root tty       4,  10 Mar 21 12:01 tty10
crw-rw----   1 root dialout   4,  74 Mar 21 12:01 ttyS10
crw-------   1 root root     10,  63 Mar 21 12:01 vga_arbiter
crw-------   1 root root     10, 137 Mar 21 12:01 vhci
```

それでは、miscデバイスドライバでリストがどのように使用されているのか見てみよう。まず、 `miscdevice` の構造体を見てみよう。

```C
struct miscdevice
{
      int minor;
      const char *name;
      const struct file_operations *fops;
      struct list_head list;
      struct device *parent;
      struct device *this_device;
      const char *nodename;
      mode_t mode;
};
```

`miscdevice` 構造体の4番目のフィールドに、 `list` があるのがわかる。これは、登録されたデバイスのリストだ。
ソースコードの先頭に、misc_listの定義がある。

```C
static LIST_HEAD(misc_list);
```

which expands to the definition of variables with `list_head` type:

```C
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```

and initializes it with the `LIST_HEAD_INIT` macro, which sets previous and next entries with the address of variable - name:

```C
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

Now let's look on the `misc_register` function which registers a miscellaneous device. At the start it initializes `miscdevice->list` with the `INIT_LIST_HEAD` function:

```C
INIT_LIST_HEAD(&misc->list);
```

which does the same as the `LIST_HEAD_INIT` macro:

```C
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

In the next step after a device is created by the `device_create` function, we add it to the miscellaneous devices list with:

```
list_add(&misc->list, &misc_list);
```

Kernel `list.h` provides this API for the addition of a new entry to the list. Let's look at its implementation:

```C
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}
```

It just calls internal function `__list_add` with the 3 given parameters:

* new  - new entry.
* head - list head after which the new item will be inserted.
* head->next - next item after list head.

Implementation of the `__list_add` is pretty simple:

```C
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}
```

Here we add a new item between `prev` and `next`. So `misc` list which we defined at the start with the `LIST_HEAD_INIT` macro will contain previous and next pointers to the `miscdevice->list`.

There is still one question: how to get list's entry. There is a special macro:

```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

which gets three parameters:

* ptr - the structure list_head pointer;
* type - structure type;
* member - the name of the list_head within the structure;

For example:

```C
const struct miscdevice *p = list_entry(v, struct miscdevice, list)
```

After this we can access to any `miscdevice` field with `p->minor` or `p->name` and etc... Let's look on the `list_entry` implementation:

```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

As we can see it just calls `container_of` macro with the same arguments. At first sight, the `container_of` looks strange:

```C
#define container_of(ptr, type, member) ({                      \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

First of all you can note that it consists of two expressions in curly brackets. The compiler will evaluate the whole block in the curly braces and use the value of the last expression.

For example:

```
#include <stdio.h>

int main() {
	int i = 0;
	printf("i = %d\n", ({++i; ++i;}));
	return 0;
}
```

will print `2`.

The next point is `typeof`, it's simple. As you can understand from its name, it just returns the type of the given variable. When I first saw the implementation of the `container_of` macro, the strangest thing I found was the zero in the `((type *)0)` expression. Actually this pointer magic calculates the offset of the given field from the address of the structure, but as we have `0` here, it will be just a zero offset along with the field width. Let's look at a simple example:

```C
#include <stdio.h>

struct s {
        int field1;
        char field2;
		char field3;
};

int main() {
	printf("%p\n", &((struct s*)0)->field3);
	return 0;
}
```

will print `0x5`.

The next `offsetof` macro calculates offset from the beginning of the structure to the given structure's field. Its implementation is very similar to the previous code:

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

Let's summarize all about `container_of` macro. The `container_of` macro returns the address of the structure by the given address of the structure's field with `list_head` type, the name of the structure field with `list_head` type and type of the container structure. At the first line this macro declares the `__mptr` pointer which points to the field of the structure that `ptr` points to and assigns `ptr` to it. Now `ptr` and `__mptr` point to the same address. Technically we don't need this line but it's useful for type checking. The first line ensures that the given structure (`type` parameter) has a member called `member`. In the second line it calculates offset of the field from the structure with the `offsetof` macro and subtracts it from the structure address. That's all.

Of course `list_add` and `list_entry` is not the only functions which `<linux/list.h>` provides. Implementation of the doubly linked list provides the following API:

* list_add
* list_add_tail
* list_del
* list_replace
* list_move
* list_is_last
* list_empty
* list_cut_position
* list_splice
* list_for_each
* list_for_each_entry

and many more.
