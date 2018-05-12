So there are two answers to this queston.  One is shorter, but slower because it runs in `O(n*n)` time, where n is the number of nodes.

The other can run in O(1) time, and is the answer I would favour in most situations.

Let's take a look at the short one first.

# The short solution

```perl

sub treewalk {
        my ($tree, $target, $path) = @_;

        die ($path) if ($tree->[0] eq $target);

        foreach my $e (@$tree[1..@$tree-1]) {
           treewalk($e, $target, [@$e, [ grep { !ref($_) ||   $_->[0] ne $e->[0] } @$path] ]);}
}

```

The parameters

* tree - The subtree we are searching
* $target - The new root node identifier.  In this case, it is alway the first element of the array.
* $path - The path through the old tree that we have followed to get to the current node.  Conveniently, this path is also the correct answer.

```perl
        die ($path) if ($tree->[0] eq $target);
```

Perl doesn't have continuations, so when we find the correct result, we will stop searching and return our result through the die handler, which we catch later with an `eval {}` block.

```perl
        foreach my $e (@$tree[1..@$tree-1]) {
           treewalk($e, $target, [@$e, [ grep { !ref($_) ||   $_->[0] ne $e->[0] } @$path] ]);}
```
Perl is a wonderfully compact language, which allows me to write gems like this.  It's even smaller than the scheme version, which surprised me, because I thought scheme would be a little better at this.  

Starting from the top, we can see that I visit every child node `foreach (@$tree[1..@$tree-1])` in depth first order.

```perl
    [@$e, [ grep { !ref($_) ||   $_->[0] ne $e->[0] } @$path] ]
```
Here we build the path that we have followed, by placing the current node `@$e` as the parent of the path tree, and the current path `@$path` as the children, thus inverting the tree as we work our way down.

I also make sure to remove the current node from the child list, to prevent recursive loops.

```perl
grep { !ref($_) ||   $_->[0] ne $e->[0] }
```

## The module
```perl
package Reroot;
use strict;
use Exporter;
our @ISA = qw(Exporter);
our @EXPORT_OK = qw(reroot);

sub treewalk {
        my ($t, $target, $path) = @_;

        die ($path) if ($t->[0] eq $target);

        foreach my $e (@$t[1..@$t-1]) {
           treewalk($e, $target, [@$e, [ grep { !ref($_) ||   $_->[0] ne $e->[0] } @$path] ]);}
}

sub reroot {
        eval { treewalk($_[0], $_[1], $_[0]); die undef};
        return $@;
}

1;
```

## The test harness

```perl
use strict;
use Test::More;
use lib '.';
use Reroot qw/reroot/;

my $nodes = ['A', ['B', ['D']], ['C']];

print "Original: ['A', ['B', ['D']], ['C']]\n";
is_deeply( reroot($nodes, 'D'), ['D', ['B', ['A', ['C']]]], "ReRoot to D");
is_deeply( reroot($nodes, 'C'), ['C', ['A', ['B', ['D']]]], "ReRoot to C");
is_deeply( reroot($nodes, 'B'), ['B', ['D'], ['A', ['C']]], "ReRoot to B");
done_testing();
```

## Bonus: The scheme version

```racket
#lang racket

[define nodes  '[A [B [D]] [C]]]

[define [treewalk tree target path return] 
  [if [equal? [car tree] target] [return path]
  [map [lambda [e]
         [treewalk e target
                   [append e [list [filter [lambda [t] [or [not [list? t]] [not [equal? [car t] [car e]]]]] path]]]
                   return]
         ] [cdr tree]]]]

[define [test expected got name]
  [displayln [if [equal? expected got]
                 [format  "~a... Passed!" name]
                 [format "~a... Failed! Expected ~a, got ~a" name expected got]]]]

[define [reroot tree target]
  [call/cc [lambda [return]
             [treewalk tree target tree return]]]]

[test '[D [B [A [C]]]] [reroot nodes 'D] "Reroot D"]
[test '[C [A [B [D]]]] [reroot nodes 'C] "Reroot C"]
[test '[B [D] [A [C]]]  [reroot nodes 'B] "Reroot B"]
```

# The long solution

The short solution works in cases where we don't get to prepare the data structure ourselves, perhaps because we are getitng it from a framework or another module.  I'd still prefer to modify that module, but that depends on the circumstances.

If we can prepare the data structure ourselves, I would choose a different graph structure that would allow me to return answers in O(1) time.

Even if I can't do that, this approach is worth using if I have to reroot the same data structure many times.

The challenge to this problem is in finding a reverse path through a directed graph, which forces me to follow the path and then build the reverse path.  The easiest way to do this is to build a new tree, which stores the forwards and reverse paths.  Effectively, double-linked lists. Then all I have to do to reroot the tree is find the desired root node, and that is the answer.

The calling function should be able to work with the new tree easily, but if that is unacceptable, I have included a function to build a new tree in the same format as the input tree.

If I need to build a new tree every time, the complexity is O(2n), if I can just return the start of the new tree, then it is O(1).

Let's take a look.

```perl
         $nodes{$_[0]}->{$_[1]} = 1;
         $nodes{$_[1]}->{$_[0]} = 1;
```

In `prepare`, I walk the entire input tree, noting the forwards, and backwards, connections between each node.  Later, we start at new node by looking it up in the $nodes hash

```perl
        $nodes->{'D'}
```

and then fetch all its neighbours

```perl
foreach my $k (reverse sort keys %{$nodes->{$root}}) { 
        print "Neighbour: $k\n";
}
```

I can then build a new "perl array representation" by recursively visiting all neightbours

```perl
 foreach my $k (reverse sort keys %{$nodes->{$root}}) {
     push @ret, toPerl($k, $nodes, $root) unless $k eq $prev;
 }
```

As an example of this working, end-to-end, I read in a fresh tree, rebuild it to find all neighbours, then return the "perl array representation".

```perl
sub reroot {
        my $tree = shift;
        my $target = shift;
        my $nodes = prepare($tree);
        my $res =  toPerl($target, $nodes);
        return $res;
}
```

However, as noted, this is the least efficient process, running at O(2n).  If we can cache and work with the intermediate structure, the efficiency goes to O(1).


## The module
```perl
package Reroot1;
use strict;
use Exporter;
our @ISA = qw(Exporter);
our @EXPORT_OK = qw(reroot);


sub prepare {
        my $tree = shift;
        my %nodes;


        my $adjacent = sub {
         $nodes{$_[0]}->{$_[1]} = 1;
         $nodes{$_[1]}->{$_[0]} = 1;
        };

        my $treewalk;
        $treewalk = sub {
                 my $t = shift;
                 return $t unless ref($t);
                 foreach my $e (@$t[1..@$t-1]) {
                     &$adjacent($t->[0], &$treewalk($e));
                 }
                 return $t->[0];
        };

        &$treewalk($tree);
        return \%nodes;
}

sub toPerl {
 my ($root, $nodes, $prev) = @_;
 return $root unless (keys %{$nodes->{$root}});
 my @ret = ($root);
 #Reverse sort so we can test easily.  Would write proper tests but I've already spent waaay to long on this.
 foreach my $k (reverse sort keys %{$nodes->{$root}}) {
     push @ret, toPerl($k, $nodes, $root) unless $k eq $prev;
 }

 return [@ret];
}

sub reroot {
        my $tree = shift;
        my $target = shift;
        my $nodes = prepare($tree);
        my $res =  toPerl($target, $nodes);
        return $res;
}

1;
```

## The test harness
```perl
use strict;
use Test::More;
use lib '.';
use Reroot1 qw/reroot/;

my $nodes = ['A', ['B', ['D']], ['C']];

print "Original: ['A', ['B', ['D']], ['C']]\n";
is_deeply( reroot($nodes, 'D'), ['D', ['B', ['A', ['C']]]], "ReRoot to D");
is_deeply( reroot($nodes, 'C'), ['C', ['A', ['B', ['D']]]], "ReRoot to C");
is_deeply( reroot($nodes, 'B'), ['B', ['D'], ['A', ['C']]], "ReRoot to B");
done_testing();
```
