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

これは、 `list_head` 型で変数定義を拡張している。

```C
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```

そして、 `LIST_HEAD_INIT` マクロでそれを初期化している。 name という変数のアドレスを使い、前と後のエントリーをセットしている。

```C
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

それでは、 `misc_register` という、いくつものデバイスを登録するい関数を見ていこう。
はじめに、それは `miscdevice->list` を `INIT_LIST_HEAD` 関数で初期化している。

```C
INIT_LIST_HEAD(&misc->list);
```

それは、`LIST_HEAD_INIT` マクロと同じことを行う。

```C
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

次に、 `device_create` 関数でデバイスが作られた後、それを以下のように、デバイスリストに追加している。

```
list_add(&misc->list, &misc_list);
```

カーネルの `list.h` はこのようにリストへのエントリを追加するAPIを提供している。
その実装を見てみよう。

```C
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}
```

それは、3つのパラメータを `__list_add` 関数に渡しているだけだ。

* new  - 新しいエントリ
* head - 新しい項目が挿入される前のリストの先頭
* head->next - リストの先頭の次の要素

`__list_add` 関数の実装は非常にシンプルだ。

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

ここでは、 `prev` と `next` の間に新しい項目をリクエストする追加している。
よって、 我々がはじめに `LIST_HEAD_INIT` マクロで定義した `misc` リストは `mscdevice->list` への前と後のポインタを持っている。

しかし、まだひとつ疑問が残る。
どうやって、リストの要素を取得するのかだ。これには、特別なマクロがある。

```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

これは3つのパラメータを受け取る。

* ptr - list_head構造体のポインタ
* type - 構造体の型
* member - 構造体の中のlist_headの名前

例をあげよう。

```C
const struct miscdevice *p = list_entry(v, struct miscdevice, list)
```

これで、 `p->minor` や `p->name` などから `miscdevice` にアクセス可能になる。
`list_entry` の実装を見てみよう。


```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

同じ引数で、 `container_of` マクロを呼んでいるだけだ。
`container_of` は、初めて見ると奇妙に見えるかもしれない。

```C
#define container_of(ptr, type, member) ({                      \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

まず最初に、{} の中に2つの式があることに注意しなければならない。
コンパイラは{}野中のブロックを評価し、最後の式の値を使用する。

例えば、

```
#include <stdio.h>

int main() {
	int i = 0;
	printf("i = %d\n", ({++i; ++i;}));
	return 0;
}
```

は、 `2` を表示する。

次のポイントは `typeof` だ。
名前からわかるように、それは与えられた変数の方を返却する。
私が初めて `container_of` マクロの実装を見た時最も奇妙に思えたのは、 `((type *)0)` の部分だった。
このポインタは魔法のように、構造体のアドレスから、与えられたフィールドへのオフセットを計算するのだが、
ここでは `0` となっているため、それはそのまま与えられたフィールドになる。簡単な例を見てみよう。

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

上記は `0x5` を表示する。

次の `offsetof` マクロは、構造体の初めから、与えられたフィールドへのオフセットを計算する。
その実装は、前のコードと非常に似ている。

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

`container_of` マクロについてまとめてみよう。
`container_of` マクロは、 `list_head` 構造体のフィールドのアドレス、 `list_head` 構造体のフィールドの名前、コンテナ構造体の型を素に、構造体自体のアドレスを返却する。
マクロの最初の行は `__mptr` を定義している。これは、 `ptr` 自体のポインタを持っている。
今、 `ptr` と `__mptr` は同じアドレスを指している。
厳密にはこの行は不要だが、型チェックに便利なのだ。最初の行は、与えられた構造体( `type` )が `member` と呼ばれるメンバを持っていることを確かめている。
2行目で、 `offsetof` 膜とでフィールドへのオフセットを計算し、それを構造体の型アドレスから減算する。以上だ。

当然、`<linux/list.h>` には `list_add` と `list_entry` 以外の関数もある。双方向連結リストの実装は以下のAPIを提供している。

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

他にも多数。
