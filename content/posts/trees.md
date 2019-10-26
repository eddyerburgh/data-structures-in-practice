---
title: "Trees"
date: 2019-10-11
---

This post is an introduction to the tree data structure. You'll learn what a tree is, why you might use trees, and how the DOM tree is implemented in Blink — the browser-engine used by Chrome. <!--more-->

## What are trees

A tree is an abstract data type that represents a hierarchical tree structure, like a family tree.

{{< figure src="/images/trees/bernoulli-family-tree.svg" title="Figure 1: The Bernoulli family tree" >}}

Trees are defined recursively by Donald Knuth as a finite set \\(T\\) of one or more nodes where:

1. There is one designated root node.
2. The remaining nodes (excluding the root) are partitioned into \\(m >= 0\\) disjoint sets \\(T_1, ... ,T_m,\\) and each of the sets are also trees.

Typically, trees are drawn with the root at the top, like in Figure 2.

{{< figure src="/images/trees/tree.svg" title="Figure 2: A tree" >}}

You can use terminology from family trees to describe the relationship between tree nodes. A node has \\(m\\) child nodes (or \\(m\\) children), and that node is parent to each of those children. Child nodes of a parent are sibling nodes, and the parent of a parent is its grandparent. I haven't heard people referring to nodes as third cousins before, but you certainly could if you wanted to.

{{< figure src="/images/trees/labeled-tree.svg" title="Figure 3: Relationship of nodes to node A" >}}

Trees are a common data structure that are used in many situations, but what's so great about them?

## Why trees?

Trees are fantastic for modelling hierarchical data. Humans have been using tree-like structures to represent data for hundreds of years, and so the tree abstract data type was a logical creation.

Trees can also have properties that make them efficient for searching. B-trees for example, are often used by databases to create indexes for fast retrieval. I'm going to write many future posts about the different types of tree and their uses, but this post is focussed on generic trees.

One use of trees is to represent an HTML document.

## The DOM tree

The DOM (Document Object Model) is an interface for interacting with documents, usually an HTML document.

HTML uses a style of markdown that builds a structured document. That document can be represented as a tree, with a `Document` as the root node.

For example, consider the following HTML:

```html
<!DOCTYPE html>
<head>
  <title>My page</title>
</head>
<body>
  Some text.
</body>
```

This HTML is represented by the following DOM tree:

```plain
Document
├── Doctype: html
├── Element: head
│   └── Element: title
│       └── Text
└── Element: body
    └── Text
```

The DOM is implemented in all major browsers.

The rest of the post will look at how the DOM is implemented in Blink, the browser-engine used by Chrome. Blink is responsible for parsing HTML documents and rendering the document in the browser. It creates a DOM tree to both represent the DOM and to implement the JavaScript DOM API.

The rest of this post will look at how Blink represents the DOM, and how it traverses the DOM when `document.querySelectorAll()` is called.

### Representing DOM nodes in chromium

The DOM tree can contain several different types of nodes, such as text nodes and comment nodes. To keep things simple, this posts focusses on element nodes.

Element nodes contain pointers to their parents, their first child, and their last child. Siblings are represented as a linked list, where each sibling contains a pointer to its next and previous sibling.

{{< figure src="/images/trees/blink-dom-tree.svg" title="Figure 4: Elements in a tree" >}}

Blink is written in C++. Being C++, there's plenty of inheritance used to representing a DOM tree. Each `Element` inherits from a `ContainerNode` class, which includes pointers to the child nodes: `first_child_` and `last_child_`.

```cpp
class CORE_EXPORT ContainerNode : public Node {
 // ..
 private:
  TraceWrapperMember<Node> first_child_;
  TraceWrapperMember<Node> last_child_;
};
```

`ContainerNode` inherits from the `Node` class. `Node` contains a pointer to the parent: `parent_or_shadow_host_node_` (I'm going to ignore the shadow trees in this post), as well as the `previous_` and `next_` pointers that point to the siblings. `ContainerNode` also contains methods for accessing the siblings, for example `nextSibling()`:

```cpp
class CORE_EXPORT Node : public EventTarget {
 public:
  // ..
  Node* nextSibling() const { return next_; }
  // ..
  private:
  // ..
  TraceWrapperMember<Node> parent_or_shadow_host_node_;
  TraceWrapperMember<Node> previous_;
  TraceWrapperMember<Node> next_;
  // ..
};
```

So that's how the tree is represented: using firstChild and lastChild pointers, and having each sibling point to the next.

### Traversing the tree

You can access DOM elements matching a selector using the `querySelectorAll()` method:

```js
document.querySelectorAll('.some-class');
```

`querySelectorAll()` searches the DOM tree for all elements matching the selector, starting at the first child of the node it was called on. The traversal is specified as a pre-order traversal (called **tree order** in the DOM spec).

A pre-order depth-first-search processes the root node first, then performs a tree order traversal on each of its children from left to right recursively.

{{< figure src="/images/trees/pre-order-tree.svg" title="Figure 5: Numbering of nodes visited in pre-order traversal" >}}

Calling `document.querySelectorAll()` in the browser causes V8 (the JavaScript engine) to call into Blink using bindings created at compile-time. V8 calls a `querySelectorAll()` C++ method with the corresponding C++ `Document` object. `querySelectorAll()` then calls `QuerySelectorAll()` on the `Document` object to get the result.

In `QuerySelectorAll()`, Blink first parses and validates the selector string. If the selector is valid, it executes the query. Blink contains a fast-path for a single class selector, which calls `CollectElementsByClassName()`.

`CollectElementsByClassName()` performs the tree traversal using a range-based for loop, which iterates over each element in pre-order search:

```cpp
template <typename SelectorQueryTrait>
static void CollectElementsByClassName(
    ContainerNode& root_node,
    const AtomicString& class_name,
    const CSSSelector* selector,
    typename SelectorQueryTrait::OutputType& output) {
  for (Element& element : ElementTraversal::DescendantsOf(root_node)) {
    // Process node
  }
}
```

For each element, Blink checks if it matches the selector, and appends the element to the output vector if it does:

```cpp
template <typename SelectorQueryTrait>
static void CollectElementsByClassName(/* */) {
  for (Element& element : ElementTraversal::DescendantsOf(root_node)) {
    // ..
    if (selector && !SelectorMatches(*selector, element, root_node))
      continue;
    SelectorQueryTrait::AppendElement(output, element);
    if (SelectorQueryTrait::kShouldOnlyMatchFirstElement)
      return;
  }
}
```

The real meat of the traversal is implemented in the range returned by `ElementTraversal::DescendantsOf()`.

If you aren't familiar with C++, a range-based for loop executes a for loop over a range. A range can just be an object (`__range`) with `begin()` and `end()` members that return the first item and the item the for loop should stop at, respectively. You can think of a range-based for loop as `for ( ; __begin != __end; ++__begin)`, where `__begin` is `__range.begin()`, and `__end` is `__range.end()`.

The first item returned by the iterator is the first child. On each following iteration, `++` is used to access the next element in the iterator returned by `ElementTraversal::DescendantsOf()`.

Initially, the first element is the left-most child of the root element that `querySelectorAll()` was called on.

A descendant iterator eventually calls `NodeTraversal::TraverseNextTemplate()` to generate the next element, which is where the traversal logic is implemented.

```cpp
template <class NodeType>
inline Node* NodeTraversal::TraverseNextTemplate(NodeType& current,
                                                 const Node* stay_within) {
  if (current.hasChildren())
    return current.firstChild();
  // ..
}
```

If the node doesn't have any children, it calls `nextSibling()` (`current.next_`):

```cpp
template <class NodeType>
inline Node* NodeTraversal::TraverseNextTemplate(NodeType& current,
                                                 const Node* stay_within) {
  // ..
  if (current.nextSibling())
    return current.nextSibling();
  // ..
}
```

If there are no children, and the child has no siblings to the right, `NextAncestorSibling()` is called to traverse back up the tree:

```cpp
template <class NodeType>
inline Node* NodeTraversal::TraverseNextTemplate(NodeType& current,
                                                 const Node* stay_within) {
  // ..
  return NextAncestorSibling(current, stay_within);
}
```

`NextAncestorSibling()` uses another range-based for loop to move to the next ancestor sibling. This loop uses an iterator to climb the tree. It does this by calling `node.parentNode()`, which calls `node.parent_or_shadow_host_node_.get()`. It then accesses the next sibling of the parent node (if there is one)

```cpp
Node* NodeTraversal::NextAncestorSibling(const Node& current,
                                         const Node* stay_within) {
  // ..
  for (Node& parent : AncestorsOf(current)) {
    // ..
    if (parent.nextSibling())
      return parent.nextSibling();
  }
  return nullptr;
}
```

This loop uses an iterator to climb up the tree (by calling `node.parentNode()`).

Once it reaches the root node (`stay_within`), Blink returns a `nullptr`. The query has finished:

```cpp
Node* NodeTraversal::NextAncestorSibling(const Node& current,
                                         const Node* stay_within) {
  // ..
  for (Node& parent : AncestorsOf(current)) {
    if (parent == stay_within)
      return nullptr;
    // ..
  }
  return nullptr;
}
```

Once the iterators have completed, the result is in a vector. The vector is then returned to V8 as a NodeList via V8 binding.

That's how traversals are done in Chromium.

## Conclusion

Trees are great for representing hierarchical data.

The DOM is represented as a tree, and Chrome uses a tree structure internally to represent the DOM.

Trees can have lots of interesting properties that make them great for a variety of use-cases. Future posts will look at how different trees are used to implement databases, fast-lookup in Linux, and many more.
