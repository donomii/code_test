(This text is best viewed in a monospace font.)

This exercise is about an operation on a tree data structure.

Concepts
--------

Consider a set of distinct objects, called nodes. Each node has a
unique string-valued label.

Any two distinct nodes are either adjacent or not. No node is adjacent
to itself.

Define a "path" from n1 to nN to be a sequence of (at least two) nodes
n1, n2, n3, .... nN-1, nN where n1 is adjacent to n2, n2 to n3,
... and nN-1 is adjacent to nN, and where n1, n2, ... nN-1 are all
distinct.  N is the length of that path.

For every two distinct nodes n and m, there is a unique path from n to
m.

A set with the above properties is called a *tree*.

Note that in a tree, any two nodes n and m have a well defined
distance: distance is 0 if n=m, otherwise distance is the length of
the path from n to m.

We define a *rooted tree* to be a tree with a designated node, which
we call the *root*.

Define the *children* of a node n, where n has distance d to the root,
to be the nodes adjacent to n, that have a distance d+1 to the root.
n is the called the "parent" of those nodes.

We will draw a rooted tree with the root at the top, and children
below it connected by lines, and children's children below those, etc.

For example,

A
|
B

is a rooted tree with root A. and so is

  A
 / \
B   C
|
D

An operation
------------

Now consider changing the root of an existing tree. For example, for
this tree T1:

  A
 / \
B   C
|
D

we may make B the root and get this tree T2:

B
| \
D  A
   |
   C

As you can see, the adjacencies are the same, it is just the root that
is chosen differently.

The Perl data representation
-----------------------

In the Perl code, we represent a rooted tree as an array, where
the first element is the label of the root, and the rest of the
elements are the representations of the children.

In this representation, our tree T1 looks like

$T1 = ['A', ['B', ['D']], ['C']]

and T2 looks like $T2 = ['B', ['D'], ['A', ['C']]].

Exercise
========

Write a Perl function reroot($T, $X) that, given the above
representation of a tree $T, and a label $X of a node in $T, returns
the representation of $T with the root changed to $X.

In our example:

reroot($T1, 'B') could return ['B', ['D'], ['A', ['C']]]. (Note that
it could also have returned ['B', ['A', ['C']], ['D']] - we did not
specify the order of the children in the representation.)

reroot($T1, 'D') would return ['D', ['B', ['A', ['C']]]].

reroot($T1, 'C') would return ['C', ['A', ['B', ['D']]]].

Ideally, your code should consist of two parts: the part that knows
about our perl-array representation, and a part that implements the
algorithm.

Please package your solution as a Perl module that comes with a couple
of relevant unit tests.

Also, it would be great if you could say something about the
space-time complexity of your solution.

And if you need to make compromises because you are short on time,
delivering correct and readable code is more important than writing
efficient or working code.
