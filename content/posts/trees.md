---
title: "Trees"
date: 2019-10-11
---

This post is an introduction to the tree data structure. You'll learn what a tree is, why you would use trees, and how the DOM tree is represented in chromium. <!--more-->

Just like arboreal trees, there are many different types of tree data structures. Some types are useful for searching for an item quickly, some are useful for cheaply maintaining a hierarchical order, and some are just useful for representing hierarchical structures.

This post is an introduction to general trees.

## What are trees

Trees are a directed acyclic graph.

<!-- TODO: add image of tree -->

Think family tree.

* Image of tree
* Tree can have one or many children
* Directed acyclic graph?

Trees are full of colorful terminology, there's a root, leaves. Define Leaf nodes. Define root.

<!-- TODO: add image of tree with root, leaves -->

Tree data structures inherited their terminology from the kinship model. A node references its children directly, and it is in turn a parent to those children. The child nodes of a node are siblings.The parent of a parent is its grandparent. Step family and in-laws haven't been included, but I'm sure they could be.

<!-- TODO: add image of tree with parent, children -->

The term tree was first coined in 1857 by a British mathematician.

Define tree as a connected, acyclic graph (in other words it's a graph that doesn't contain cycles and each node in the graph is connected by a path). Rooted tree (where a node is designated as root).

## Why trees?

Trees work well for a variety of reasons:

* Modelling hierarchical data, e.g. family tree.
* Can be built with special properties to improve cost, e.g. binary search tree

Trees have been used for centuries to represent hierarchies. Family trees, content trees (in the original Encyclopedia), .

## How represented in DOM

The DOM (Document Object Model) provides a way to interact with an HTML document in JavaScript running in a browser.

HTML documents are structured documents that can be represented as a tree. For example:

```plain
document
├── head
│   └── title
└── body
    ├── h1
    ├── paragraph
    ├──ol
    │   ├── li
    │   └── li
    └── footer
```

The DOM is defined in .

Trees work well for representing HTML documents because an HTML document is structured. It uses a style of markdown that builds tree data structures.

<!-- TODO: DOM HTML image -->

This is well-represented as a tree, which it is specifies in the DOM spec defines the object model as a tree.


Blink builds parses HTML documents into a DOM tree. The DOM is used both to internally represent the DOM, and to implement the JavaScript DOM API.

The rest of this post will look at how Blink represents the DOM, and how it traverses the DOM when `document.querySelectorAll()` is called.

### Representing DOM nodes in chromium

The DOM tree can contain several different types of nodes, like text nodes, and comment nodes. This section will focus on DOM elements (check correct name).

Element nodes contain pointers to their parents, their first child, and their last child. Siblings are represented as a linked list, where each sibling contains a pointer to its next sibling.

<!-- TODO: Add image of nodes connected in tree -->

A DOM node is represented with an `Element` object(CHECK). Each Element object contains a pointer to its first child and last child and its parent. Siblings are linked lists, with one sibling pointing to the next. This is because ??

Being C++, there's plenty of inheritance. An `Element` inherits from `ContainerNode`, which includes pointers to child nodes. `first_child_`: The meaning is obvious and - `last_child_`.

```cpp
class CORE_EXPORT ContainerNode : public Node {
 public:
  // ..
  HTMLCollection* Children();

  Element* QuerySelector(const AtomicString& selectors);
  // ..

 private:
  TraceWrapperMember<Node> first_child_;
  TraceWrapperMember<Node> last_child_;
};
```

ContainerNode inherits from the `Node` class. The `Node` contains a pointer: `parent_or_shadow_host_node_`, `previous_`: Points to the previous sibling, `next_`: Points to the next sibling. The class also contains methods for interacting with the node, for example `nextSibling()`:

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

### Traversing the tree

You can access DOM elements matching a selector using the `querySelectorAll()` method:

```js
document.querySelectorAll('.some-class')
```

Blink uses range-based for loops and iterators to implement the traversal. I'll explain how these work, so don't worry if you're not familiar with C++.

Will search the DOM tree for all matching elements. The traversal is in pre-order, known as **tree order** in the DOM spec. This searches first: root, left node, right node.

<!-- TODO: copy image of pre-order traversal -->

Chromium (Blink?) creates bindings between V8 (the JavaScript engine) and C++. Calling `document.querySelectorAll` calls a corresponding method on the corresponding C++ `Document` object.

In the QuerySelectorAll method, Blink first parses and validates the selector string. If the selector is valid, it executes the query. Blink contains a fast-path for a single class selector, which calls `CollectElementsByClassName()` to traverse each element in preorder order to find matching elements.

`CollectElementsByClassName()` performs the tree traversal using a range-based for loop, which access each element (`element`) in preorder search:

```cpp
template <typename SelectorQueryTrait>
static void CollectElementsByClassName(
    ContainerNode& root_node,
    const AtomicString& class_name,
    const CSSSelector* selector,
    typename SelectorQueryTrait::OutputType& output) {
  for (Element& element : ElementTraversal::DescendantsOf(root_node)) {
    // Check if element matches selector
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

If you aren't familiar with C++, a range-based for loop executes a for loop over a range. A range in this case is just an object (`__range`) with `begin()` and `end()` members, which return the first item and the item that the for loop stops at, respectively. You can think of a range-based for loop as `for ( ; __begin != __end; ++__begin)`, where `__begin` is `__range.begin()`, and __end is `__range.end()`.

The first item returned from the iterator is the first child. On each following iteration, `++` is used to access the next element (CHECK how it's done). Items returned by the `ElementTraversal::DescendantsOf()` range are iterator objects with an overloaded `++` operator member that returns the next element in the traversal.

Initially, the first element is the left-most child of the root element that querySelectorAll was called on.

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

If the node doesn't have any children, then it calls `nextSibling()` (`current.next_`):

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


## TODO

- Check terminology for for-loop, how is element created each loop, pointers?
