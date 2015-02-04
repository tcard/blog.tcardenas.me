---
title: Walking trees through Go's concurrency
layout: post
---

Today at work I faced the task of extracting some stats about all files in a folder and its subfolders. Thinking about how to express the problem in Go, this is the best I came up with, leveraging some of the most characteristic features of the language. I hope you find it interesting!

**Hey! This post folows a long path to the actual interesting findings. You probably want to [just jump there](#communicating-sequential-processes).**

So, quick reminder. A tree is a recursive data structure. A _tree_ is composed of _nodes_. Each node is either a _branch_ or a _leaf_. _Branches_ have trees clinging from them.

Now, let's say we want to have a tree and do something with its nodes. For instance, finding one of them by some tag they have attached. We must then compare each node's label with our query with and return the matching one.

### Haskell perfection

Haskell's algebraic data types (ADTs) are supposed to be the perfect match to this:

{% highlight haskell %}
-- A Tree is either a Branch with subtrees or a Leaf.
-- Both carry a String as a tag.
data Tree = Branch String [Tree] | Leaf String
  deriving (Show)

findNode :: String -> Tree -> Maybe Tree
-- A function can happily walk a tree via pattern
-- matching.
findNode q t@(Leaf tag)
  | tag == q    = Just t
  | otherwise   = Nothing
findNode q t@(Branch tag subtrees)
  | tag == q    = Just t
  | otherwise   = foldl findInBranch Nothing subtrees
    where 
      findInBranch acc subtree = case acc of
        Just _  -> acc
        Nothing -> findNode q subtree


allLeavesTags :: Tree -> [String]
allLeavesTags (Leaf tag) = [tag]
allLeavesTags (Branch _ subtrees) = foldl allInBranch [] subtrees
    where 
      allInBranch acc subtree = acc ++ (allLeavesTags subtree)
{% endhighlight %}

(**[Runnable.](http://codepad.org/DCtgZunm)**)

(Note: I'm not really a Haskeller. This piece of code might or might not hurt the sensibility of a real one.)

So that's Haskell, and Haskell means perfection, right?

Let's see how our favourite language does.

### Go: our way is the old way.

We all know that [Go doesn't have <strike>generics</strike> algebraic types][go-sum-types], so no luck with that beautiful approach. We must resort the ugly old unsafe type-casting (or type-asserting, in Go terms); after all, Go is the new Blub language for sad 9-to-5ers, anti-intellectual pundits and illiterate script kiddies.

{% highlight go %}
type Tree interface {
	Tag() string
}

type Branch struct {
	tag      string
	Subtrees []Tree
}

func (b Branch) Tag() string { return b.tag }

type Leaf struct {
	tag string
}

func (l Leaf) Tag() string { return l.tag }

func findNode(q string, t Tree) Tree {
	if t.Tag() == q {
		return t
	}

	branch, ok := t.(Branch)
	if !ok {
		return nil
	}

	for _, subtree := range branch.Subtrees {
		v := findNode(q, subtree)
		if v != nil {
			return v
		}
	}

	return nil
}

func allLeavesTags(t Tree) []string {
	switch node := t.(type) {
	case Leaf:
		return []string{node.Tag()}
	case Branch:
		ret := []string{}
		for _, subtree := range node.Subtrees {
			ret = append(ret, allLeavesTags(subtree)...)
		}
		return ret
	}
	return nil // wat
}
{% endhighlight %}

(**[Runnable.](http://play.golang.org/p/obX88lryWp)**)

The main problems that I envision here compared to the Haskell version are:

1. `nil` is a possible value for `t` and there is nothing you can do about it. Paying our part of [the billion dolar][billion-dollar] here. In Haskell there is no null values, and maybe-or-maybe-not-exists values are explicitly handled and type-checked.

2. Since any type can implement `Tree`, your traversing function can get values with types you don't know about and it will compile no problemo; and it can blow up your function if you are not careful. (Even if you [do the trick][sum-types-trick], you still have to deal with 1 regarding this.) In Haskell ADTs, all different concrete types (`Branch`, `Leaf`) are listed out in the definition of `Tree`, and you can't extend that list.

It still feels like that we're forcing interfaces into a role they are not really designed to cover, don't you think?

### Communicating sequential processes.

There is a feature in Go that can help us express this better.

In both versions, when `findNode` and `allLeavesTags` are handed a `Branch` and they want to traverse its subtrees, they have to loop over them. I think those functions shouldn't know how to walk a tree. It is only interested in its nodes. What if the subtrees were lazily loaded from the filesystem, or from the network? So why don't we leave that responsibility to a generic function?

Goroutines to the rescue. We can model tree traversal as two communicating processes: one producing tree nodes, the other consuming them.

Here is what we're doing:

* `WalkTree(Tree)` starts a traversal of the given tree. It returns a `TreeWalker`, composed of three channels: `Branches` and `Leaves`, which produce `BranchWalker` and `Leaf` respectively, and `Done`, which signals that there are no more nodes in the tree.

* `BranchWalker` is like `Branch`, but its `Subtree` is a `TreeWalker` instead of a `[]Tree`. This way, the way the subtrees are generated is abstracted to the consumer.

* The consumer walks a tree by repeatedly receiving through the channels in its `TreeWalker` in a select.

Now the code at the producer part is more complex. You can [check it out in the runnable playground](http://play.golang.org/p/mekC5QDbjM). The consumer part looks like this:

{% highlight go %}
func findNode(q string, t TreeWalker) Tree {
	for {
		select {
		case branch := <-t.Branches:
			if branch.Tag() == q {
				return branch
			}
			if v := findNode(q, branch.Subtrees); v != nil {
				return v
			}
		case leaf := <-t.Leaves:
			if leaf.Tag() == q {
				return leaf
			}
		case <-t.Done:
			return nil
		}
	}
}

func allLeavesTags(t TreeWalker) []string {
	ret := []string{}

	for {
		select {
		case branch := <-t.Branches:
			ret = append(ret, allLeavesTags(branch.Subtrees)...)
		case leaf := <-t.Leaves:
			ret = append(ret, leaf.Tag())
		case <-t.Done:
			return ret // Hey, the awkward 'return nil' is gone!
		}
	}
}
{% endhighlight %}

(**[Runnable.](http://play.golang.org/p/mekC5QDbjM)**)

So now the consumers don't know how to walk a tree: they just listen through some channels.

Along the way, we got rid of type-asserting in the consumers by providing a type-safe interface from the producer. Isn't that nice? On the flip side, now we can induce deadlocks from the consumers if we're not careful.

Unfortunately, as Go lacks generics, we can't make this code reusable for other types of trees without resorting to `interface{}`, which perhaps defeats the whole purpose. But is this complication worth it anyway? That's on you to decide.

[go-sum-types]: http://golang.org/doc/faq#variant_types
[billion-dollar]: http://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare
[sum-types-trick]: http://www.jerf.org/iri/post/2917