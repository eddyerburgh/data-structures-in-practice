---
title: "Intrusive linked lists"
date: 2019-09-13T12:29:04+01:00
---

This post will teach you what intrusive linked lists are, and how they're used in Linux to manage processes.<!--more-->

## What are intrusive linked lists?

Intrusive linked lists are a variation of [linked lists]({{< ref "/linked-lists" >}}) where the links are embedded into the structure that's being linked.

In a typical linked list implementation, a list node contains a `data` pointer to the linked data and a `next` pointer to the next node in the list.

{{< figure src="/images/intrusive-linked-lists/linked-list-pointers.svg" title="Figure 1: A linked list" >}}

An intrusive list still contains a `next` pointer to the next list node, but there's no `data` pointer because the list is part of the linked data structure itself.

{{< figure src="/images/intrusive-linked-lists/intrusive-linked-list-pointers.svg" title="Figure 2: An intrusive linked list" >}}

So a `list` structure for an intrusive singly linked list only contains a `next` pointer to another list node:

```c
typedef struct list {
  struct list *next;
} list;
```

The `list` structure is then embedded in the structure that will be linked. For example, you might have a `item` structure with a `val` member:

```c
typedef struct item {
  int val;
  list items;
} item;
```

To add a new item `i2` to the list of `i1`, you set the `items.next` pointer of `i1` to the address of `i2.items`:

```c
item* i1 = create_item(16);
item* i2 = create_item(18);

i1->items.next = &i2->items;
```

You can access the object that contains a list node by first getting the address of the list object (e.g. the value of `i1.items.next`). You then subtract the offset of the list member from the address of the list object. The **offset** is the number of bytes a member is positioned from the beginning of its containing object.

{{< figure src="/images/intrusive-linked-lists/intrusive-list-memory-address.svg" title="Figure 3: The address of an object with an embedded list" >}}

Consider a `list` object inside an object `i2` at memory address 0x18. The `list` member is offset 8 bytes from the beginning of the `item` data structure. Therefore, the beginning address of the `i2` object is 0x18 - 0x08 (0x10).

In C compiled by GCC, you can subtract bytes from a pointer   by casting the pointer variable to a void pointer, which has a size of 1 byte when compiled with GCC. You can then subtract a number from the pointer value:

```c
item* _i2 = (void *)(i1->items.next) - 8;
```

_Note: Pointer arithmetic on a void pointer is illegal in C, but is supported by GCC. Linux is compiled using GCC, and so it can perform pointer arithmetic on void pointers._

Subtracting an absolute value isn't portable, because data types can be different sizes depending on the architecture. A better way is to make use of the `offsetof` macro. `offsetof` returns the offset of a member from its containing structure in bytes:

```c
item* _s2 = (void *)(i1->items.next) - (offsetof(item, items));
```

So that's the basics of intrusive linked lists. After all that pointer arithmetic, you're probably wondering what benefit there is to using intrusive linked lists.

## Why use intrusive linked lists?

There are two main reasons to use intrusive lists over non-intrusive linked lists:

* Fewer memory allocations.
* Less cache thrashing.

With non-intrusive linked lists, you must allocate memory for a list node each time you add a new object to a list. With intrusive linked lists, you don't. This means there are fewer error cases to handle when using intrusive lists.

Intrusive linked lists also suffer less from cache thrashing. Iterating through a non-intrusive list node requires dereferencing a list node, and then dereferencing the list data. Intrusive linked lists only require dereferencing the next list node.

## Doubly and circular linked lists

Doubly linked lists and circular linked lists are variations of singular linked lists. Linux uses circular doubly linked lists, so this section will cover what they both are before diving into the Linux implementation.

A **doubly linked list** is linked list that keeps pointers to both the next node and the previous node.

{{< figure src="/images/intrusive-linked-lists/doubly-linked-list.svg" title="Figure 4: A doubly linked list" >}}

The list structure would contain an extra `prev` pointer:

```c
typedef struct dlist {
  struct dlist *next;
  struct dlist *prev;
} dlist;
```

Doubly linked lists makes deletion and insertion easier, because you only need a reference to a single node.

For example, deletion:

```c
void dlist_delete(dlist *node) {
  if(node->next) {
    node->next->prev = node->prev;
  }
  if(node->prev) {
    node->prev->next = node->next;
  }
  node->prev = NULL;
  node->next = NULL;
}
```

Another variant of linked lists are **circular linked lists**. A circular linked list is a linked list that never points to a null value. Instead, the final node points to the first node. In a circular doubly linked list, the first node also points to the final node.

{{< figure src="/images/intrusive-linked-lists/circular-doubly-linked-list.svg" title="Figure 5: A circular doubly linked list" >}}

A circular linked list makes it easy to iterate through an entire list from any list node, without keeping a reference to a specific list head:

```c
void list_print_each(list* node) {
  list* start = node;
  do {
    printf("%d,", node->val);
    node = node->next;
  } while(node != start);
}
```

## Linked lists in Linux

Linux uses linked lists extensively. They are used for all kinds of tasks, from keeping track of free memory slabs, to iterating through each running process. A search for the `struct list_head` structure returns over 10,000 results in Linux 5.2.

Linux includes a few different list structures. The most popular is an intrusive circular doubly linked list.

### Implementing intrusive linked lists

The Linux circular doubly linked list is defined in [include/linux/list.h](https://elixir.bootlin.com/linux/v5.2/source/include/linux/list.h).

The list structure is `struct list_head`. It contains a `next` and `prev` pointer:

```c
struct list_head {
	struct list_head *next, *prev;
};
```

You create a linked list of objects by embedding `list_head` as a member on the struct that will be made into a list:

```c
struct atmel_sha_drv {
    struct list_head head;
    // ..
};
```

A new list can either be statically or dynamically initialized.

A statically allocated list can use the `LIST_HEAD_INIT` macro:

```c
static struct atmel_sha_drv atmel_sha = {
	.dev_list = LIST_HEAD_INIT(atmel_sha.dev_list),
	// ..
};
```

`LIST_HEAD_INIT` expands to set the `next` and `prev` pointers of the list node to point to itself:

```plain
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

To initiate a linked list dynamically, you use the `INIT_LIST_HEAD` macro. `INIT_LIST_HEAD` is called with a pointer to a `list` node. Again, the list's `next` and `prev` pointers are set to point to itself:

```c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	WRITE_ONCE(list->next, list);
	list->prev = list;
}
```

_Note: the `WRITE_ONCE` macro prevents unwanted compiler optimizations when assigning a value._

After a list has been initialized, new items can be added with `list_add`. `list_add` accepts a head node pointer, and a pointer to the node that should be inserted. It then calls the internal `__list_add` function to insert the new node between the `head` node, and `head->next`:

```c
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}
```

`__list_add` reassigns pointers to add the new list node:

```c
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	// ..

	next->prev = new;
	new->next = next;
	new->prev = prev;
	WRITE_ONCE(prev->next, new);
}
```

Linux also provides a `list_entry` macro to access the containing data structure of a list node.

`list_entry` expands to a `container_of` macro:

```plain
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

The `container_of` macro calculates the containing object's address by subtracting the offset of the list node from the address of the `list_head` object:

```plain
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	((type *)(__mptr - offsetof(type, member))); })
```

That's the basic implementation of intrusive lists in Linux.

### Tracking processes

In Posix, a process is an executing instance of a program. One of the kernel's key responsibilities is to create processes and schedule them so that each process runs for an appropriate amount of time.

Internally, Linux refers to processes as **tasks**. A task is represented with a `task_struct` structure.

A `task_struct` includes a `list_head` member named `tasks` which is a list of all existing tasks (known as the task list):

```c
struct task_struct {
  // ..
  pid_t	pid;
  // ..
  struct list_head tasks;
  // ..
};
```

The initial task is statically allocated as `init_task`, and the `tasks` field is initialized with itself as the head:

```c
struct task_struct init_task = {
  // ..
	.tasks		= LIST_HEAD_INIT(init_task.tasks),
};
```

Future tasks are added to this task list when they are created.

New tasks are created in Linux by forking. This is implemented in `copy_process`, which creates a new `task_struct` from the currently executing process (`current`) by calling `dup_task_struct`:

```c
struct task_struct *copy_process(
  // ..
)
{
  struct task_struct *p;
  // ..
  p = dup_task_struct(current, node);
  // ..
}
```

After a new task is created, it's added to the task list by calling `list_add_tail_rcu` with the address of `init_task.tasks`:

```c
struct task_struct *copy_process(
  // ..
)
{
  // ..
  list_add_tail_rcu(&p->tasks, &init_task.tasks);
}
```

`list_add_tail_rcu` is a variation of the `list_add` function from earlier. It uses RCU, which is a mechanism for ensuring concurrent reads (no need to go into the details). `list_add_tail_rcu` has the effect of adding the newly created task's task list node to the tail of the `init_task` task list.

The task list is mainly used when the Kernel needs to perform an action on each task. For example, freezing tasks when a computer is going into hibernate mode, swapping tasks to an updated version of the kernel during a live patch, or when a signal is sent to each process.

One of the times a signal is sent to each process is when the SysRq key and e are pressed together, which terminates all processes.

_Note: SysReq is a key that was added in the 80s, Linux adds default shortcuts you can use with it._

The kernel registers a handler function that's called when the SysReq + e keys are pressed. The handler calls `send_sig_all`, with `SIGTERM` which sends a `SIGTERM` signal to all processes apart from the `init` process and kernel tasks. It does this with the `for_each_process` macro. You can see from the code that it does nothing if the process is a kernel thread or the `init` task, otherwise it calls `do_send_sig_info`.

```c
static void send_sig_all(int sig)
{
	struct task_struct *p;

	// ..
	for_each_process(p) {
		if (p->flags & PF_KTHREAD)
			continue;
		if (is_global_init(p))
			continue;

		do_send_sig_info(sig, SEND_SIG_PRIV, p, PIDTYPE_MAX);
	}
	// ..
}
```

At this point it's macros all the way down.  The `for_each_process` macro expands into a `for` loop that loops over each item in the list by changing the reference of `p` to the next task, using the `next_task` macro, starting at `init_task`:

```plain
#define for_each_process(p) \
	for (p = &init_task ; (p = next_task(p)) != &init_task ; )
```

The `next_task` macro expands to `list_entry_rcu` to get the next `task_struct` of the list head pointer:

```plain
#define next_task(p) \
	list_entry_rcu((p)->tasks.next, struct task_struct, `tasks`)
```

`list_entry_rcu` is itself a macro that expands to the `container_of` macro, which then gets the base address of the containing structure.

It's worth noting that the `tasks` linked list isn't the only way Linux keeps reference to tasks. It also creates a dictionary data structure (an idr) that offers constant time access, which is used to quickly access a `task` object from a given pid. This is much more efficient way of accessing a single `task` struct.

## Conclusion

Intrusive linked lists are an interesting approach to improving the performance of linked lists by reducing cache thrashing and memory allocation.

Intrusive linked lists are used extensively by Linux, and you should understand them if you plan to do any kernel hacking.
