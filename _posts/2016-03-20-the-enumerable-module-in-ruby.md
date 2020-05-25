---
layout: post
title: The Enumerable Module in Ruby
tags: [blog]
---

A brief look at Ruby’s Enumerable Module and methods

> # “A set is a Many that allows itself to be thought of as a One” — **Georg Cantor**

I’m hurtling towards the one year mark as a full time Rubyist. I still consider myself to be an utter novice but I hope to have at least picked up a few good habits along the way. One area where my understanding has evolved is in dealing with Ruby collections — traversing Hashes and Arrays and manipulating them in a way that is efficient and idiomatic.

I intend to sketch out the behaviour of some of the more common methods found within Ruby’s ‘Enumerable’ module, in particular when you want to interrogate or manipulate a collection. In general I’m using ‘enumerable’ loosely in the context of Arrays and Hashes i.e. collections which can be counted/ordered in some way. This post is heavily indebted to all of the further reading identified at the end.

### Everything was each

When I first started working with Ruby I more or less only knew the fundamental ‘each’ method, and I manipulated and twisted it to achieve the desired outcome For instance, if I wanted to identify only the even numbers in an array:

```ruby
even_numbers = []
[1, 2, 3, 4, 5].each do |num|
  even_numbers << num if num.even?
end

puts even_numbers
  => [2, 4]
```

As far as I was concerned this worked fine. But I was conscious of regularly seeing other, far terser ways of achieving the same work in other people’s code.

### The Enumerable Module

One of the core Ruby modules is ‘[Enumerable](http://ruby-doc.org/core-2.2.2/Enumerable.html)’. A mixin which provides a “_several traversal and searching methods, and with the ability to sort_”. This module is included within the core classes Array and Hash

```ruby
Array.included_modules?
  => [Enumerable, Kernel]
Hash.included_modules?
  => [Enumerable, Kernel]
```

I’m not going to look too closely at sorting in this post, but focus more on traversing enumerable classes and, perhaps, manipulating them in some way. I first saw the various methods I’d been stumbling across clearly laid out by [Robert Qualls](http://www.sitepoint.com/guide-ruby-collections-iii-enumerable-enumerator/):

    Enumerable.instance_methods.sort

      => [:all?, :any?, :chunk, :collect, :collect_concat, :count,  :cycle, :detect, :drop, :drop_while, :each_cons, :each_entry, :each_slice, :each_with_index, :each_with_object, :entries, :find, :find_all, :find_index, :first, :flat_map, :grep, :group_by, :include?, :inject, :lazy, :map, :max, :max_by, :member?, :min, :min_by, :minmax, :minmax_by, :none?, :one?, :partition, :reduce, :reject, :reverse_each, :select, :slice_after, :slice_before, :slice_when, :sort, :sort_by, :take, :take_while, :to_a, :to_h, :zip]

I’m going to briefly dwell on a small subset of these methods that I find myself using regularly: #any?, #collect and #select.

### #any?

The first thing to note about this method is the concluding question mark, which conventionally indicates that it will return a boolean value (i.e. True or False). Such methods are also called ‘predicate’ methods. Generally any is used to ask whether any of the elements of the collection conform to the block that is passed in, for example:

```ruby
capitals = ['London', 'Edinburgh', 'Belfast', 'Cardiff']
capitals.any? { |name| name.length >=5 }
  => true
```

The fundamental behaviour of any? is to “_[return] true if the block ever returns a value other than false or nil_”. If no block is passed to the method then it simply iterates through the collection implictly asking ‘is there anything here that isn’t false or nil?’

```ruby
foo = [nil, false, true, 5]
bar = [nil, false, nil, false]
foo.any? # true
bar.any? # false
```

I now find myself using any? regularly, not to manipulate a collection but rather to question its contents.

### #collect

The absence of a question mark suggests that the above method does more than simply return a boolean value. In contrast to #each, #collect actually returns a whole new array, its elements are the product of running the passed block on every item in the original collection. So, for instance,

```ruby
def double_it(enum)
  enum.collect { |x| x * 2 }
end

double_it([2, 4, 6, 8, 10])
  => [4, 8, 12, 16, 20] # returns the manipulated array
```

The collect method, in practical terms, [is almost interchangable with the ‘map’ method](http://stackoverflow.com/questions/5254732/difference-between-map-and-collect-in-ruby).

### #select

This is the method I find myself using most regularly. In manipulating a collection I often want to identify a subset of it that conforms to a certain condition e.g. ‘_All of those in Year 9 who scored more than 75 in the exam_’.

```ruby
year_nine = {Adams: 34, Butler: 55, Carr: 93, Davies: 44, Edwards: 88}

year_nine.select { |student, score| score > 75 }
  => {:Carr=>93, :Edwards=>88}
```

Select returns an entirely new object consisting of those elements from the original collection that met the conditions passed into the select’s block. In other words it’s used when you’re asking ‘_Can I have a subset of the original collection which fits a particular criteria’._

The inverse of #select is #reject, which returns an array of all of the elements for which the block returned false. For instance:

```ruby
homes = {'Jay Gatsby' => 'West Egg', 'José Buendía' => 'Macondo', 'Billy Pilgrim' => 'New York'}

homes.reject { |name, town| town.length < 8 }
  => {"Jay Gatsby"=>"West Egg", "Billy Pilgrim"=>"New York"}
```

In the above example we end up with a new Hash containing those elements for which the test ‘_Town name is less than 8 characters_’ returned false. i.e. all of those towns whose name is eight or more character in length.

### Easily Enumerable

It is also possible to use the core Enumerable Module in your own classes ([this was explained really excellently by Jaime Bellmyer](http://kconrails.com/2010/11/30/ruby-enumerable-primer-part-1-the-basics/)). Conceptually an object is enumerable if, broadly speaking, it can be ‘counted’ i.e. it has elements which are able to be iterated through. So if you’re defining a class which represents a collection then the logic of Enumerable could be useful.

```ruby
class Bookshelf
  include Enumerable
  attr_accessor :books

  def initialize
    @books = []
  end

  def each(&block)
    @books.each { |book| block.call(book) }
  end
end
```

There are a few deceptively complicated things going on above (e.g. ‘& _the_ _unary ampersand operator_’ and an attr_accessor) — but for the purpose of this post the key point is that you can include the Enumerator mixin, and the powerful logic it provides, in your own classes. The key requirement is that the class has its own #each method, because this core behaviour of being able to move through a set is key to the Enumerable module.

### Ampersand shortcut

Just as a final addendum — the _unary ampersand operater_ I mentioned above often appears when traversing collections

```ruby
users.map(&:name)
```

There are some good explanations of what’s going on ‘under the hood’ [here](http://stackoverflow.com/questions/1961030/ruby-ampersand-colon-shortcut) & [here](http://kconrails.com/2010/12/01/ruby-enumerable-primer-part-2-unary-ampersand-operator/) (some of which still eludes my understanding). But to quickly outline in purely practical terms: when you’re looking (for instance) to identify a single property from items in a collection, this allows you to quickly call for the attribute on every member of a set and return an array of those values. So the above code is equivalent to the slightly more verbose

```ruby
users.map do |user|
  user.name
end
```

## Conclusion

Clearly we have only scratched the surface of Ruby’s Enumerable module. The message I hoped to capture in writing this post, as someone who’s still very new to Ruby, is that it’s always worth taking a step back and considering ‘_What am I trying to find out here_?’ — because this will often change the tools you need for the job.

(Thanks very much to @chrislowis for his suggestions on improving this post).

## Further Reading

- [‘_Enumerables_’ — The Bastard Book of Ruby](http://ruby.bastardsbook.com/chapters/enumerables/)

- [‘_Ruby Explained: Map, Select, and Other Enumerable Methods_’ — Erik Trautman](http://www.eriktrautman.com/posts/ruby-explained-map-select-and-other-enumerable-methods)

- [‘_A Guide To Ruby Collections_’ — Robert Qualls](http://www.sitepoint.com/guide-ruby-collections-iii-enumerable-enumerator/)

- [‘_Understanding Ruby Enumerables by Reimplementing Them_’ — Maurício Linhares](http://mauricio.github.io/2015/01/12/implementing-enumerable-in-ruby.html)

- [‘_Ruby Enumerable Magic: The Basics_’ — Jaime Bellmyer](http://kconrails.com/2010/11/30/ruby-enumerable-primer-part-1-the-basics/)

## Background noise

- Jamie xx — In Colour

- Sun Kil Moon — Universal Themes
