---
title: "Hash tables"
date: 2019-09-12T12:28:04+01:00
---

In this post you'll learn what hash tables are, why you would use them, and how they're used to implement dictionaries in CPython.<!--more-->

## What are hash tables?

Hash tables are a data structure used for storing (key,value) pairs.

A simple hash table stores items in an array. It calculates the index of an item using a **hashing function**.

Commonly, this is done in two steps:

```python
hash = hash_function(key)
index = hash % array_size
```

The main hash table operations area `set_item`, `get_item`, and `delet_item`.


It's possible that a key maps to the same index as another key. This is known as a **collision**. A big part of a hash table is collision resolution.

There are several collision resolution approaches. The most common techniques are:

1. Chaining.
2. Open addressing.

Chaining is where each item in the hash table array is a linked list. When an item is added to the index, it's added as a node on the list. This approach can cause problems where one linked list grows very large, leading to O(n) access. To solve this, hash tables that implement chaining add logic to make sure no one linked list grows too large. If it does, the table is dynmically resized, and the indexes are recomputed—leading to a more balanced table.

Open addressing is an alternative. In the case of a collision, elements in the array are checked in a deterministic sequence, until a free slot is found(this is known as **probing**). The sequence that determines probing is slightly complex. I'll cover it in detail in the next section.

Open addressing implementations are often more performant due to better cache locality. Many projects that implement hash tables have moved from chaining to open addressing, such as PHP, Python, and Ruby. But chaining is still used in popular languages (e.g. Go, [GCC C++ stdlib](https://github.com/gcc-mirror/gcc/blob/gcc_5_3_0_release/libstdc++-v3/include/bits/hashtable.h)).

## Why use hash tables?

Hash tables are powerful. In the average case and in practice they give O(1) access to key value pairs.

The worst case is generally O(n), but the average case is O(1):

| Operation | Average case | Worst case |
| --------- | ---------- | ---------- |
| Search    | O(1)       | O(n)       |
| Insertion | O(1)       | O(n)       |
| Deletion  | O(1)       | O(n)       |

In practice:

The downside of hash tables is that they consume more memory than a simple list or array.

## Hash tables in Python

Python is an interpreted programming language. It's defined in the [Python reference](https://docs.python.org/3/reference/index.html), and developers create their own implementations to run python.

The reference Python implementation is CPython. CPython is written in C and works like most interpreted languages: it parses code into an AST, compiles it into byte code, and runs it on a virtual (stack-based) machine.

One useful Python data structure is a dictionary. Dictionaries work like hash tables, you can set and access values by a key.

This is how you define a dictionary in Python:

```python
tel = {
    'alice' : 2025550143,
    'bob'   : 2025550196,
    'carol': 2025550191
}
```

And you access items from a dictionary like so:

```python
print tel['alice'] ## 2025550143
```

Python implements dictionaries using hash tables.

The rest of this blog post I'll show you how Python uses a hash table to implement the following code:

```python
d = {}
d['key'] = 'some value'
```

The first thing to do is to discuss hash tables at a high-level.

### Hash tables in CPython

There are three parts to CPython hash tables:

1. Generating an index
2. Handling collisions
3. Resizing the table

Python dictionary hash tables use an array to store items. It uses open addressing for conflict resolution.

Python optimizes hash tables into contained hash tables and split tables. For simplicity, I'll focus on the contained tables.

A combined hash table has three kinds of slots:

1. Unused. This doesn't hold a key value pair now and never has.
2. Active. Holds an active key value pair.
3. Dummy. Previously held a key value pair but was deleted.

A Python array contains less items than it can hold. This is to avoid collisions.

### Generating an index

A hash function needs to generate an index from a given key.

A hash function is a one-way function. It always produces the same output for a given input, but it's not possible to convert the output back to the input. The only way to determine the original input is through trial and error.

Different data types can be used as a key in Python. Each data type has its own hashing algorithm. For example, string uses one, and int32 (??) uses another. Commonly in Python the key is a unicode string. By default, Python uses the SipHash algorithm for unicode strings.

SipHash is a relatively fast, cryptographic hash function. Cryptographic means that it meets certain criteria, for more information check out ....

On a 64-bit machine, the SipHash returns a 64-bit unsigned number (`size_t` (CHECK)). It needs to be converted into an index to be used in an array. This is done by creating a bit mask rather than a modulo. The reason is that modulo is an expensive operation on most processors. Reminder that a bit mask is bits that are set to 1. For example:

```plain
00100110
00000111 3-bit bitmask
&
00000110 Result
```

The bit mask works by setting the size of the array to be a power of 2. Any number that is a power of 2 is a single bit:

```plain
2   0010
4   0100
8   1000
16 10000
```

This can easily be converted into a bit mask by subtracting 1 from the numbers:

```plain
1  0001
3  0011
7  0111
15 1111
```

By applying the bitmask to a larger value. As an example, consider 16-bit int. You have an array that contains 32 items, with index 0-31. You can use a mask to convert your 16-bit value into an index within the range:

```plain
1011 0011 1011 1001
0000 0000 0001 1111 mask

0000 0000 0001 1001
```

This is how Python converts an. It's important for the array size to be a power of two, because then you can get a bit mask by simply subtracting 1 from the total number of the map:

```c
size_t mask = (size_t)DK_SIZE(keys) - 1;

Py_hash_t hash = ep->me_hash;
size_t i = hash & mask;
```

So that's how Python generates the initial index `i`. If the element at index `i` is empty, then the new element can be added. However, if it's not then Python must resolve the conflict.

### Conflict resolution

Conflict resolution occurs when an element already exists at the index generated by the hash table. Python uses open addressing to resolve conflicts.

Collision resolution is important, because the Python hash function can compute similar hashes in certain cases (e.g. for large integers).

The simplest implementation of open addressing is to linearly search through the array in case of a conflict. This is known as a **linear probing sequence**:

```c
while(1) {
  if(table[i++] == EMPTY) {
    return i;
  }
}
```

First part of collision detection is to visit table indices using this recurrence: `j = ((5*j) + 1) mod 2**i`


Linear probing sequence works great for hash table implementations that have a good hashing algorithm that gives evenly distributed results. The Python hashing algorithm however, is not so good. It causes conflicts. A linear approach could often lead to large sections covered in. One solution would be to improve the hashing algorithm, but the Python devs instead chose a different probing sequence.

Has a perturb value (initially the hash). Perturb is shifted down 5 bits each iteration. Eventually perturb becomes 0, meaning i*5 + 1 is used. This is fine because it produces every int in range 0-2^n.

```c
#define PERTURB_SHIFT 5

// ..

perturb >>= PERTURB_SHIFT;
i = mask & (i*5 + perturb + 1);
```


_Perturbing_ in this context means adding a small shift to a value. That's why the value is called `peturb`. It means that all of the 64-bit hash randomness is used, even though the initially used value is normally much less.

After a while the peturb create 0. Once that happens, it is `j = ((5*j) + 1) mod 2**i` which will eventually create every value from 0-.2**i

Also, when elements are deleted they are not removed from the array. This is so that later searches will continue to be able to find (CHECK)

The final part of the hash table to discuss is resizing. Once array is 2/3 full, resized.


### Resizing the table

Python dictionaries are dynamic. You can continue to add items until you reach the limit (currently ??).

In order to handle this, Python resizes maps as new items are added to them. The resizing works on two premise:

1. Only allow 2/3 of max space to be used
2. Resize once 2/3 of space is reached

Python only allows 2/3 of an array to be filled with items. This is to keep the hash table array sparse and therefore to reduce collisions.

###  Representing dictionaries

The dictionary data structure can be confusing at first.

There are three important parts of a dictionary

* The dictionary struct, representing the entire dictionary
* A dictionary keys object, which contains keys with (?)
* A values array,

A dictionary is represented by several structures. These include:

`dk_indices` is the hash table. It stores the index of its entry in `dk_entries`. When a hash table is resized, dk_indices is updated, whereas dk_entries stay in same index (CHECK):

```plain
+---------------+
| dk_refcnt     |
| dk_size       |
| dk_lookup     |
| dk_usable     |
| dk_nentries   |
+---------------+
| dk_indices    |
|               |
+---------------+
| dk_entries    |
|               |
+---------------+
```

A quick note before looking at the structures that represent this. Objects in python are represented as a `PyObject`. These contain a header, and are then cast to the correct structure. The dictionary is represented as a `PyDictObject` struct, but it can be passed around as `PyObject`.

The `PyDictObject` structure contains the number of items used, a unique version tag (used for what?), and a keys object:

```c
typedef struct {
    PyObject_HEAD

    Py_ssize_t ma_used; // Number of items in dictionary

    uint64_t ma_version_tag; // Globally unique value, changes when dict is modified

    PyDictKeysObject *ma_keys; // Keys object

    PyObject **ma_values; // NULL for combined tables
} PyDictObject;
```

The keys object is important. It ...

```c
struct _dictkeysobject {
    Py_ssize_t dk_refcnt;

    /* Size of the hash table (dk_indices). It must be a power of 2. */
    Py_ssize_t dk_size;

    /* Function to lookup in the hash table (dk_indices)
    dict_lookup_func dk_lookup;

    /* Number of usable entries in dk_entries. */
    Py_ssize_t dk_usable;

    /* Number of used entries in dk_entries. */
    Py_ssize_t dk_nentries;

    /* Hash table of dk_size entries. It holds indices in dk_entries,
       or DKIX_EMPTY(-1) or DKIX_DUMMY(-2).

       Indices must be: 0 <= indice < USABLE_FRACTION(dk_size).

       The size in bytes of an indice depends on dk_size:

       - 1 byte if dk_size <= 0xff (char*)
       - 2 bytes if dk_size <= 0xffff (int16_t*)
       - 4 bytes if dk_size <= 0xffffffff (int32_t*)
       - 8 bytes otherwise (int64_t*)

       Dynamically sized, SIZEOF_VOID_P is minimum. */
    char dk_indices[];  /* char is required to avoid strict aliasing. */

    /* "PyDictKeyEntry dk_entries[dk_usable];" array follows:
       see the DK_ENTRIES() macro */
};
```
`dk_indices` contains `PyDictKeyEntry`. This contains the hash, key, and value for an item:

```c
typedef struct {
    Py_hash_t me_hash;
    PyObject *me_key;
    PyObject *me_value;
} PyDictKeyEntry;
```

`dk_entries` is an array of the entries (as a `PyDictKeyEntry`). `DK_ENTRIES(dk)` is used to get a pointer to the entries. DK_ENTRIES isn't included in a struct, but it's allocated and memset each time a new keys object is created ([see code](https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/dictobject.c#L568-L568))




Python works by parsing and compiling code into bytecode that runs on a stack. The function for intializing an empty dictionary is, and the function for adding items is.



```c
struct _dictkeysobject {
    Py_ssize_t dk_refcnt;
    Py_ssize_t dk_size;
    dict_lookup_func dk_lookup;
    Py_ssize_t dk_usable;
    Py_ssize_t dk_nentries;
    char dk_indices[];
};
```

* The dictobject struct
* A _dictkeysobject object (keys & hashes)
* A values array

The Python hash table

### Using a dictionary

Now it's time to see how dictionaries are implemented for a small snippet of Python code.

```python
x = {}
x['key'] = 'a'
x['key2'] = 'b'
```

The two opcodes that I'll show you are `BUILD_MAP`, and `STORE_SUBSCR`.

`BUILD_MAP` is used to initialize the map. That calls `_PyDict_NewPresized((Py_ssize_t)oparg)`. In the example code, oparg is 0 (the dictionary is empty). `_PyDict_NewPresized` just calls through `PyDict_New()` without any args to create a new dictionary:

```c
PyObject *
_PyDict_NewPresized(Py_ssize_t minused)
{
    // ..

    if (minused <= USABLE_FRACTION(PyDict_MINSIZE)) {
        return PyDict_New();
    }

   // ..
}
```

`PyDict_New` is a proxy function, and calls through to `new_dict()` after implementing refs.

```c
PyObject *
PyDict_New(void)
{
    dictkeys_incref(Py_EMPTY_KEYS);
    return new_dict(Py_EMPTY_KEYS, empty_values);
}
```

`new_dict()` allocates memory for a dict object and sets initial values for dict object.

```c
static PyObject *
new_dict(PyDictKeysObject *keys, PyObject **values)
{
    PyDictObject *mp;

    // ..

    mp = PyObject_GC_New(PyDictObject, &PyDict_Type);

    // ..

    mp->ma_keys = keys;
    mp->ma_values = values;
    mp->ma_used = 0;
    mp->ma_version_tag = DICT_NEXT_VERSION();
    ASSERT_CONSISTENT(mp);
    return (PyObject *)mp;
}
```

That's it. There's now a dict object assigned to a variable (code for assigning to a variable isn't shown).

The next opcode adds a value to the map. This is `STORE_SUBSCR`, short for store subscript (subscript notation is [], e.g. `x['key'] = 'hello'`).

`STORE_SUBSCR` is a more generic opcode. It can be called on many objects, like list. So `PyObject_SetItem` method is called with the map object. This then accesses the `tp_as_mapping` member of the map object. This object contains function pointers that implement different behaviors (like a set method). You can see this in the code:

```c
PyObject_SetItem(PyObject *o, PyObject *key, PyObject *value)
{
    PyMappingMethods *m;

    // ..
    m = o->ob_type->tp_as_mapping;
    if (m && m->mp_ass_subscript)
        return m->mp_ass_subscript(o, key, value);
    // ..
}
```

The registered dict `mp_ass_subscript` is the dubiously-named `dict_ass_sub`. `dict_ass_sub` deletes an item if there isn't any value to set, otherwise it calls the more sensibly named `PyDict_SetItem`:

```c
static int
dict_ass_sub(PyDictObject *mp, PyObject *v, PyObject *w)
{
    if (w == NULL)
        return PyDict_DelItem((PyObject *)mp, v);
    else
        return PyDict_SetItem((PyObject *)mp, v, w);
}
```

`PyDict_SetItem` is where a hash is set. Python checks to see if a hash has already been created for the string, if it's not then it calls `PyObject_Hash()` with the key:

```c
int
PyDict_SetItem(PyObject *op, PyObject *key, PyObject *value)
{
    Py_hash_t hash;
    // ..
    if (!PyUnicode_CheckExact(key) ||
        (hash = ((PyASCIIObject *) key)->hash) == -1)
    {
        hash = PyObject_Hash(key);
        // ..
    }
    // ..
}
```

`PyObject_Hash` gets the object type (`ob_type`) from the object. If an object type is hashable, it has a `tp_hash` function set in its type structure. See the [definition for the `PyUnicode_Type` structure](https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/unicodeobject.c#L15243-L15243). `PyObject_Hash` calls `tp_hash` on the `PyObject` it was called with to generate the hash. For unicode, this is `unicode_hash`.

`unicode_hash` calls `_Py_HashBytes` to generate a hash (check the source code if interested). It then sets a hash on the object that was hashed (caching it for later use).

```c
static Py_hash_t
unicode_hash(PyObject *self)
{
    Py_uhash_t x;  /* Unsigned for defined overflow behavior. */

    // ..

    x = _Py_HashBytes(PyUnicode_DATA(self),
                      PyUnicode_GET_LENGTH(self) * PyUnicode_KIND(self));
    _PyUnicode_HASH(self) = x;
    return x;
}
```

Once the hash has been generated, `PyDict_SetItem` can continue. `PyDict_SetItem`  can optimize for an empty dict. This creates a keys object with the minimum size (8). It also adds the function used to lookup items.

```c
int
PyDict_SetItem(PyObject *op, PyObject *key, PyObject *value)
{
    PyDictObject *mp;
    Py_hash_t hash;
    // ..

    mp = (PyDictObject *)op;

    // ..

    if (mp->ma_keys == Py_EMPTY_KEYS) {
        return insert_to_emptydict(mp, key, hash, value);
    }
    // ..
}
```

`insert_to_emptydict` creates a new keys object. Remember, a keys object . Look at that first.

```c
static int
insert_to_emptydict(PyDictObject *mp, PyObject *key, Py_hash_t hash,
                    PyObject *value)
{
    // ..
    PyDictKeysObject *newkeys = new_keys_object(PyDict_MINSIZE);
    // ..
}
```


Once the keys have been created, the entry can be added. This is done by calculating the hash (logical & the hash with the mask):

```c
static int
insert_to_emptydict(PyDictObject *mp, PyObject *key, Py_hash_t hash,
                    PyObject *value)
{
    // ..
    size_t hashpos = (size_t)hash & (PyDict_MINSIZE-1);
    // ..
}
```

Then the index is stored in the hash table. That's an interesting function.
```c
static int
insert_to_emptydict(PyDictObject *mp, PyObject *key, Py_hash_t hash,
                    PyObject *value)
{
    // ..
    dictkeys_set_index(mp->ma_keys, hashpos, 0);
    // ..
}
```

This determines the size of the keys based on the number of items that are stored. It gets the number of items from the keys, then it checks to see if the size is below a limit. For example, if the number of addressable items is less than 255 (0xff), then the indices only need to be 1 byte to store all possible addresses. There are other cases for 2 bytes and 4 bytes. You can see the other cases in the code [s](https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/dictobject.c#L359-L359).

```c
/* write to indices. */
static inline void
dictkeys_set_index(PyDictKeysObject *keys, Py_ssize_t i, Py_ssize_t ix)
{
    Py_ssize_t s = DK_SIZE(keys);

    // ..

    if (s <= 0xff) {
        int8_t *indices = (int8_t*)(keys->dk_indices);
        assert(ix <= 0x7f);
        indices[i] = (char)ix;
    }
    // ..
}
```

After `dictkeys_set_index` has stored the index of the key entry in the `dk_indices` array (memory?), the key entry is accessed. Since this is first item in the list, the entry is known to be at index 0 in entries list. The entry item values then get set as well as the dictionary values,:

```c
static int
insert_to_emptydict(PyDictObject *mp, PyObject *key, Py_hash_t hash,
                    PyObject *value)
{
 // ..
    PyDictKeyEntry *ep = &DK_ENTRIES(mp->ma_keys)[0];
    dictkeys_set_index(mp->ma_keys, hashpos, 0);
    ep->me_key = key;
    ep->me_hash = hash;
    ep->me_value = value;
    mp->ma_used++;
    mp->ma_version_tag = DICT_NEXT_VERSION();
    mp->ma_keys->dk_usable--;
    mp->ma_keys->dk_nentries++;
    return 0;
}
```

Success! The item with value of "" has been added to the dictionary.

The final part of the code to look at is what happens when a second item is added. In this case, you can see the code that potentially resizes and avoids collisions. That happens in `PyDict_SetItem` again. If the keys object exists on the dictionary that it was called with, it calls `insertdict`:

```c
int
PyDict_SetItem(PyObject *op, PyObject *key, PyObject *value)
{
    // ..
    return insertdict(mp, key, hash, value);
}
```

`insertdict` does several things:

- checks if needs resizing
- handles collisions
- adds item

First `insertdict` calls the maps `dk_lookup`, which is assigned depending on the key type.

```c
static int
insertdict(PyDictObject *mp, PyObject *key, Py_hash_t hash, PyObject *value)
{
    PyObject *old_value;
    // ..

    Py_ssize_t ix = mp->ma_keys->dk_lookup(mp, key, hash, &old_value);

    // ..
}
```

This calls the objects dk_lookup. Initially this is set to an optimized version that doesn't check need to handle dummy. Once an item is deleted, it's updated to `lookdict_unicode`.

`lookdict_unicode` uses the key hash and mask to generate an index for the item. If an item already exists at the index (there's a conflict!) it enters the conflict resolution, by changing the index using the algorithm shown earlier.

First, check out the happy path. ix is empty, and it can be returned.

```c
static Py_ssize_t _Py_HOT_FUNCTION
lookdict_unicode(PyDictObject *mp, PyObject *key,
                 Py_hash_t hash, PyObject **value_addr)
{
   // ..

    PyDictKeyEntry *ep0 = DK_ENTRIES(mp->ma_keys);
    size_t mask = DK_MASK(mp->ma_keys);
    size_t perturb = (size_t)hash;
    size_t i = (size_t)hash & mask;

    for (;;) {
        Py_ssize_t ix = dictkeys_get_index(mp->ma_keys, i);
        if (ix == DKIX_EMPTY) {
            *value_addr = NULL;
            return DKIX_EMPTY;
        }
       //
    }
    // ..
}
```

_Note: `dictkeys_get_index` performs the logic to get the correct memory address depending on the size of the map._

If it isn't, then the perturb is added to generate a new index:

```c
static Py_ssize_t _Py_HOT_FUNCTION
lookdict_unicode(PyDictObject *mp, PyObject *key,
                 Py_hash_t hash, PyObject **value_addr)
{
   // ..
    for (;;) {
        // ..
        if (ix >= 0) {
            PyDictKeyEntry *ep = &ep0[ix];
            assert(ep->me_key != NULL);
            assert(PyUnicode_CheckExact(ep->me_key));
            if (ep->me_key == key ||
                    (ep->me_hash == hash && unicode_eq(ep->me_key, key))) {
                *value_addr = ep->me_value;
                return ix;
            }
        }
        perturb >>= PERTURB_SHIFT;
        i = mask & (i*5 + perturb + 1);
    }
    // ..
}
```

That's how conflict resolution works. Back in insertdict, has `ix` (memory location of item in hash table).

Now, time to insert. But first, the map might have grown too large. That means a resize will need to be done.

```c
static int
insertdict(PyDictObject *mp, PyObject *key, Py_hash_t hash, PyObject *value)
{
    PyObject *old_value;
    PyDictKeyEntry *ep;
    // ..

    Py_ssize_t ix = mp->ma_keys->dk_lookup(mp, key, hash, &old_value);

    // ..

    if (ix == DKIX_EMPTY) {
        // ..
        if (mp->ma_keys->dk_usable <= 0) {
            /* Need to resize. */
            if (insertion_resize(mp) < 0)
                goto Fail;
        }
        // ..
    }
}
```

`insertion_resize` increases the dictionary hash table by the growth rate (3 by default):

```c
static int
insertion_resize(PyDictObject *mp)
{
    return dictresize(mp, GROWTH_RATE(mp));
}
```

Since the hash table must be a power of 2, dictresize calculaates the next lowest power of two that enables it to. This is done by bit shifting an 8-bit Py_ssize_t until the value is larger.

```c
static int
dictresize(PyDictObject *mp, Py_ssize_t minsize)
{
    Py_ssize_t newsize, numentries;
    // ..

    /* Find the smallest table size > minused. */
    for (newsize = PyDict_MINSIZE;
         newsize < minsize && newsize > 0;
         newsize <<= 1)
        ;
    // ..
}
```

Python gets a reference to the old keys object (`mp->ma_keys`). It allocates a new keys object table (`new_keys_object`). Then it copies over the old entries into the new entries location. Copies over `lookdict` values, etc.: Some other housekeeping, including freeing dictionary object.

Copies over entries. If there are entries that have been deleted (and their key is NULL), they aren't copied over:

```c
static int
dictresize(PyDictObject *mp, Py_ssize_t minsize)
{
    // ..
        if (oldkeys->dk_nentries == numentries) {
            memcpy(newentries, oldentries, numentries * sizeof(PyDictKeyEntry));
        }
        else {
            PyDictKeyEntry *ep = oldentries;
            for (Py_ssize_t i = 0; i < numentries; i++) {
                while (ep->me_value == NULL)
                    ep++;
                newentries[i] = *ep++;
            }
        }
    // ..
    return 0;
}
```

The final part is to rebuild the indices.

```c
build_indices(mp->ma_keys, newentries, numentries);
```

[`build_indices`](https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/dictobject.c#L1147-L1147) goes through the insertion process for each entry. Any items that had been deleted, won't be included (why??)\

Now that's it for the dictionary implementation.

### Deleting from dictionary

When items are deleted, they are put in a dummy state. This is because they need to exist in order for probing to be deterministic.

`dk_indices` is the hash table. It holds index in entries, or DKIX_EMPTY(-1) or DKIX_DUMMY(-2). When an item is deleted its entry is set to NULL, and its indices item is set to `DKIX_DUMMY`. Dummies can't be removed, because that would mess up probing for keys that had collisions. Instead they're kept in the map until the map is resized, at which point they aren't copied over (check).

## Questions

- What is use of ma_version_tag? Can be used for caching OPCODE
- Difference between ma_used and dk_usable
- What does incrementing ref do? (used as part of garabage collection)

## Conclusion

Hash tables are great, and hypothetically they are relatively simple. However, real-life implementations of hash tables can be complex. Especially in an interpreted language like Python.

The next blog post will take a look at a different kind of hash table: a chained hash table.


<!-- ### WHW

`tp_new` is called when new dictionary is created (WHEN??).

New `PyObject` created from calling type->`tp_alloc` (`PyType_GenericAlloc`). Values initialized (AKA ma_used set to 0 etc, unique ma_version_tag assigned, new keys object with min size created, ). self returned by `tp_new`.

`dict_init` called. `dict_update_common` called.

When item is added, `PyDict_SetItem` is called. -->


<!-- Expression is converted to `STORE_SUBSCR` (somehow).
In ceval: https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Python/ceval.c#L1836-L1836. `PyObject_SetItem(container, sub, v)` called

`tp_as_mapping` struct accessed. `mp_ass_subscript` called (`dict_ass_sub`). https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/abstract.c#L191-L191

`PyDict_SetItem` called: -->
<!-- 
`PyDict_SetItem`: validates input, creates a hash if one doesn't already exist (stored on `PyObject`)

Hash created in `PyObject_Hash`. `tp_hash` called on object that should be hashed. e.g. for string, https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/unicodeobject.c#L11738-L11738.

For me hash function is https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Python/pyhash.c#L367-L367. No need to dive into the details, but ultimately it generates a hash value (like what?). Uses SipHash. Produces 64 bit. https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Python/pyhash.c#L416-L416.

If first time, call optimized `insert_to_emptydict` (no need to check). Else call `insertdict`, which may resize dict.

`insertdict`. If table is split, keys are stored in ma_keys and values are stored in ma_values. If table is split, and key is not unicode object, `insertion_resize` called to resize entries and table.

`dk_lookup` to check for item. `lookdict` is one possibility (how?).

If is split table, resized, `ix` set to DKIX_EMPTY. do something

If `ix` is empty: if no usable, resize map (`insertion_resize`), call `find_empty_slot` to find position in hash table (). Get last entry in entries, set the index (`dictkeys_set_index`), key, hash, and value of entry (if combined). decrease usable, increase nentries, -->

<!-- 
### `lookdict`

Gets e0 in keys. Gets mask, `DK_MASK(dk)`, get index by masking hash with mask (`hash & mask`) and calls `dictkeys_get_index`. Loops until it finds free slot.

If ix >= 0, access entry at dk_entries[i], if entry has same key, then set value_address to entry->me_value and return `dk_entries[i]`.

If key not same, and hash same, increment object ref to avoid it being cleaned, compare key with entry key. If keys are not equal, do nothing (next loop). If they are, then set value_addr = ep->me_value and return ix. Else, shift with perturb. -->

<!-- 
### `insertion_resize`


`insertion_resize` will increase the hash table by the number of used keys * 3 (and must be power of 2). First it gets minimum possible size by shifting an int 8 left until it has a value greater than `minsize`.

Accesses on keys object (type `_dictkeysobject`). Allocates a  new table (`new_keys_object`) `new_keys_object` validates new size (is power of 2, is larger than min). Generates a usable fraction of ((n << 1) / 3), this keeps table spares, reducing collisions. Sets `es` var (??) to numver between 1-8. Optimization for size of min size: https://github.com/python/cpython/blob/master/Objects/dictobject.c#L551-L551. Allocates memory for object: sizeof(PyDictKeysObject), `+ es * size`, `+ sizeof(PyDictKeyEntry) * usable`. Increments total number of refs. Initializes new `_dictkeysobject` (size, usable, dk_refcnt). Sets es * size bytes to 0xff, and sets usable bytes to 0. Returns new object.

`dk_lookup` value set (this function used to perform lookups). One example is `lookdict`. TODO: Find out how it's set. Converts split table into combined table if old version is split table. If old dkentries is same as ma_used, copy entries from oldentries to newentries (`memset`), or loop over oldentries from 0 - newentries and add to newentries. If keys are minsize, dec reftotal and return to keys_free_list, else dec reftotal and free oldkeys.

`build_indices` called with ma_keys, newentries, and numentries;. Mask created from entries-size - 1. Loop over each key entry, mask hash (`size_t i =  hash & mask`). If there is a collision (`dictkeys_get_index(keys, i) != DKIX_EMPTY`), shift value until free space is found. Call `dictkeys_set_index` with keys, key index, and index in hash table.

Collision avoidance:

```c
perturb >>= PERTURB_SHIFT;
i = mask & (i*5 + perturb + 1);
```

`dictkeys_set_index`, set indices[i] to ix (which is position of key object in keys array, key entry has `value` member if combined table).
Size of indices depends on the number of items, e.g. for < 255, `int8_t`, for > 0xffffffff, `int64_t`.

Finally, set dk_usable and dk_nentries. -->


<!-- Initial creation is :
https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Python/ast.c#L2500-L2500 -->

<!-- `Dict()` is called to create dictionary expression node. `astfold_expr` called with Dict.keys and again with Dit.value. Compiler creates adds opcode `BUILD_MAP`. `BUILD_MAP` called in ceval.

`_PyDict_NewPresized` called, `PyDict_New` called. `new_dict` called. If there are some free min size dicts (numfree), use that. Else, create new object, set keys, ma_values, ma_used, ma_version_tag, to initial vals. keys and values is empty obj?.

Object is assigned. Now add. -->


<!--
`mp_ass_subscript` called (`dict_ass_sub`). https://github.com/python/cpython/blob/eb2b0c694aef6122fdf95015abb24e0d095b6401/Objects/abstract.c#L191-L191

`PyDict_SetItem` called:

`PyDict_SetItem`: validates input, creates a hash if one doesn't already exist (stored on `PyObject`)







`_PyDict_NewPresized` called, `PyDict_New` called. `new_dict` called. If there are some free min size dicts (numfree), use that. Else, create new object, set keys, ma_values, ma_used, ma_version_tag, to initial vals. keys and values is empty obj?.

Object is assigned. Now add. -->

