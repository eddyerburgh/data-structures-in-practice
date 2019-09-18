---
title: "Linked lists"
date: 2019-09-13T12:28:04+01:00
---

This post will teach you what linked lists are, why they're useful, and how they're used in cURL to set HTTP headers.<!--more-->

## What are linked lists?

A linked list is a collection of data connected via references.

{{< figure src="/images/linked-lists/singly-linked-list.svg" title="Figure 1: A linked list" >}}

Each element (or **node**) in a list contains a value and a reference to the next node.

The simplest linked list is a **singly linked list**, where each node points to the next node in the list. In a singly linked list, the final node points to a null value.

You can represent a singly linked list as a `list` structure containing `data` and a `next` pointer to another `list`:

```c
typedef struct list {
  DATA data;
  struct list *next;
} list;
```

To find an element in a list you need to visit each node one after the other, starting at the first node in the list (the **head**) and finishing at the last node (the **tail**).

{{< figure src="/images/linked-lists/singly-linked-list-with-head.svg" title="Figure 2: A linked list with a head and tail" >}}

The following `find` function demonstrates how you would search a linked list. Starting from the first node in the list (`head`), `find` loops through each node and checks to see if the node contains the data it's searching for:

```c
list* find(list* head, DATA data) {
  list* node = head;
  while(node) {
    if(node->data == data) {
      return node;
    }
    node = node->next;
  }
  return node;
}
```

An algorithm to find a list node may need to traverse an entire list in the worst case. This makes linked list search an O(n) operation.

In contrast, linked list insertion takes constant time (provided you have a reference to the node you want to insert at). An `insert` method can insert a new node by creating a node dynamically and changing the pointers of existing nodes.

The following code creates a new node and adds it to the end of a list:

```c
void insert(DATA data, list* tail) {
  list* node = create_node(data)
  tail->next = node;
  tail = node;
}
```

_Note: A linked list node is often dynamically allocated using a function like `malloc`. This adds overhead for each node that's created._

Now you know what a linked list is, the next question is _why use them_?

## Why use linked lists?

There are three main benefits of linked lists:

1. They grow dynamically.
2. They have constant time insertion and deletion.
3. They're easy to implement.

A big plus for linked lists is that they grow and shrink dynamically. You can add and remove nodes on demand without performing costly resizing operations.

Another benefit is the speed of insertion and deletion, which are both O(1) operations.

You can see the cost of the common operations in the following table:

| Operation | Worst case |
| --------- | ---------- |
| Access    | O(n)       |
| Search    | O(n)       |
| Insertion | O(1)       |
| Deletion  | O(1)       |

The final benefit is that they're easy to implement (which means less surface area for bugs).

Despite these benefits, dynamic arrays are often more performant than linked lists. Think carefully before you decide to use linked lists in your project.

The rest of this post will look at a project that's used linked lists successfully—cURL.

## Linked lists in cURL

cURL is a command line tool (curl) and a library (libcurl) used for transferring data with URLs.

Commonly, the curl tool is used to make HTTP requests to a server:

```bash
curl http://www.google.com
```

HTTP requests can include a variable number of **headers** in the form `field-name ":" [ field-value ]`.

You can add headers with the curl `--header` option:

```bash
curl --header "X-MyHeader: 123" http://www.google.com
```

Both curl and libcurl represent headers as linked lists.

### Implementing a linked list

cURL has two linked list data structures: `curl_llist` and `curl_slist`. `curl_slist` is the data structure used to represent HTTP headers.

`curl_slist` is a singly linked list that holds strings (hence the name _slist_, short for _string list_). Each node contains a pointer to a string (`data`) and a pointer to the next node in the list (`next`):

```c
struct curl_slist {
  char *data;
  struct curl_slist *next;
};
```

_Note: The code examples in this post are from curl v7.66.1._

libcurl provides methods to interact with the list. For example, `curl_slist_append`, which adds a new string to a list:

```c
struct curl_slist *headers = NULL;

slist = curl_slist_append(headers, "X-MyHeader: 123");
```

`curl_slist_append` copies the string (using `strdup`) and then calls `Curl_slist_append_nodup` to add the duplicated string to the list:

```c
struct curl_slist *curl_slist_append(struct curl_slist *list,
                                     const char *data)
{
  char *dupdata = strdup(data);
  // ..
  list = Curl_slist_append_nodup(list, dupdata);
  // ..
  return list;
}
```

_Note: Copying the string means the original string can be overwritten after `curl_slist_append` is called._

`Curl_slist_append_nodup` creates a new list node (calling `malloc` to allocate memory). It then adds the data and sets the `next` pointer of the new node to `NULL`.

If the current list is `NULL` (and therefore uninitialized) `Curl_slist_append_nodup` returns the new list node. If the list exists, the last node is retrieved with `slist_get_last`, and its `next` pointer is set to the new node:

```c
struct curl_slist *Curl_slist_append_nodup(struct curl_slist *list, char *data)
{
  struct curl_slist     *last;
  struct curl_slist     *new_item;
  // ..
  new_item = malloc(sizeof(struct curl_slist));
  // ..
  new_item->next = NULL;
  new_item->data = data;

  /* if this is the first item, then new_item *is* the list */
  if(!list)
    return new_item;

  last = slist_get_last(list);
  last->next = new_item;
  return list;
}
```

`slist_get_last` loops through until it reaches an item where `next` points to `NULL`. This is the last item in the list, and so it's returned:

```c
static struct curl_slist *slist_get_last(struct curl_slist *list)
{
  struct curl_slist     *item;
  // ..
  item = list;
  while(item->next) {
    item = item->next;
  }
  return item;
}
```

That's an overview of the `curl_slist` API. The interesting part is how it's used to add headers to an HTTP request.

Before that, a quick primer on HTTP/1.1.

### HTTP/1.1

HTTP/1.1 is an application-layer protocol for sending data between a client and a server. It was created with ease-of-use in mind.

HTTP/1.1 is ASCII-encoded, so it's human readable. A request contains a **request-line** (in the form `Method Request-URI HTTP-Version CRLF`), and a variable-length list of **headers**.

An HTTP/1.1 GET request might look like this:

```plain
GET / HTTP/1.1
Host: www.google.com
User-Agent: curl/7.54.0
Accept: */*
```

HTTP is typically sent over TCP. On POSIX-compliant systems, TCP connections are created using the sockets API. **Sockets** are an abstraction that let you establish a TCP connection with another process (often running on a remote host) and send/ receive data over the connection. You can send data to an open socket using the POSIX-defined `send` function.

The main takeaway here is that cURL can send an HTTP request by calling `send` with the HTTP request as a string.

The next section will look at how headers gets converted from command-line options, into a linked list, and then into the HTTP request string that `send` is called with.

### Adding custom headers in curl

In libcurl, custom headers are set using a `curl_slist` structure. So `curl_slist` is a user-facing data structure.

curl (the tool) uses libcurl internally. In fact, most of the work curl does it to convert command-line options into configuration operations for libcurl.

libcurl exposes an "easy API" to perform transfers, you create a `CURL` handle (called an "easy handle") to configure the request. You can set options using helper methods, like `curl_easy_setopt`.

The following is an example of making a request with a custom header:

```c
CURLcode res;
CURL *curl = curl_easy_init();

struct curl_slist *headers = NULL;

headers = curl_slist_append(headers, "X-MyHeader: 123");

// .. add other options

curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);

res = curl_easy_perform(curl); // make the request

curl_slist_free_all(headers);
```

curl tool uses the libcurl easy API internally. curl converts the command line options into an easy handle object.

When curl is called from the command line, it first parses the options passed to it in `argv`. It normalizes option names, and enters a switch statement, which runs against all possible options. For any `header` options, the option parameter value is added to a `curl_slist`.

Once curl has parsed the options, it creates an easy handle to pass to libcurl, before executing the operation.

For HTTP requests, libcurl creates a buffer to hold the request string. The request string is built by converting options into the relevant strings. The headers are added by `Curl_add_custom_headers`.

`Curl_add_custom_headers` loops over the headers list. For each header, the header is validated (by checking for a colon), and then added to the request buffer along with with a CRLF (`\r\n`).

```c
CURLcode Curl_add_custom_headers(
  struct connectdata *conn,
  bool is_connect,
  Curl_send_buffer *req_buffer)
{
  // ..
  char *ptr;
  struct curl_slist *headers;

  // ..

  // .. assign headers list to headers variable

    while(headers) {
      // ..
      ptr = strchr(headers->data, ':');
      // ..
      if(ptr) {
        // ..
        if(*ptr /* .. */ ) {
          // ..
          char *compare = /* ... */ headers->data;
          // ..
            result = Curl_add_bufferf(
              &req_buffer,
              "%s\r\n",
              compare
            );
          // ..
        }
      }
      headers = headers->next;
    }
  // ..

  return CURLE_OK;
}
```

Once all the text has been added to an HTTP request, cURL calls `send` to send the data over the TCP connection.

As well as being used for request headers, `curl_slist` is also used for other variable-length data, like cookies and trailer headers. The linked list data structure is well-suited to representing objects of varying lengths.

## Problems with linked lists

Although linked lists work well for representing variable data, they can cause performance problems in time-critical apps due to poor [cache locality](https://en.wikipedia.org/wiki/Locality_of_reference).

For example, when DOOM 3 was rewritten to run at 60FPS, the developers identified the use of linked lists as a major performance issue (see [the DOOM 3 technical note](http://fabiensanglard.net/doom3_documentation/DOOM-3-BFG-Technical-Note.pdf) for details).

Despite potential performance performance issues, linked lists are a common data structure in C programs. They are good at representing variable-length data and they are simple to implement (which goes a long way in C!).

The next blog post will look at a variation of linked lists—intrusive linked lists—and how they are used in Linux.
