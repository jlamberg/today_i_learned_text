# Entries

## 2022-02-15

Let's assume we have an ActiveRecord class Track with the attributes `id`, `title`, `length`, and some others, and that `title` is aliased by `name` using `alias_attribute :name, :title` (with parameters "newname, oldname").

Let's also assume that we want to serialize only the three mentioned attributes:

```ruby
# The list of attributes we want to serialize.
attributes = [:id, :name, :length]

# Fetch the track... (btw, alias fields have their finder methods, too!)
track = Track.find_by_name('Jeremy')

# ...and serialize it.
track.as_json(only: attributes)     # => {"id"=>6, "length"=>318}
```

That doesn't look like what we wanted; the attribute `name` was not serialized.

What we need to do is to serialize using the original attribute name and not the aliased one.

```ruby
# Find which attributes have been aliased.
aliases = Track.attribute_aliases   # => {"name"=>"title"}

# Find whether any of our attributes are aliases.
# Need to use `.map(&:to_s)` to compare strings with strings.
intersection = aliases.keys & attributes.map(&:to_s)    # => ["name"]

# For each of our alias attributes, find the original attribute name and
# replace our alias one with it.
intersection.each do |attr_alias|
    index = attributes.index(attr_alias.to_sym)
    attributes[index] = aliases[attr_alias].to_sym if !index.nil? && index >= 0
end

# Now serialize the object again.
track.as_json(only: attributes)     # => {"id"=>6, "title"=>"Jeremy", "length"=>318}
```

## 2022-10-19

I was looking at a codebase where I noticed the use of numbered parameters, a new feature in Ruby 2.7 some 3 years ago.

There seemed to be a lot of discussion about this feature on the Ruby mailing list and bug tracker back when it was planned and its first version was implemented, which was with a bit different syntax than what it is now.

Even Matz himself has [stated](https://bugs.ruby-lang.org/issues/15723#note-2):

> FYI, I am not fully satisfied with the beauty of the code with numbered parameters, so I call it a compromise.

to which, among others, Pascal Betz [commented](https://bugs.ruby-lang.org/issues/15723#note-8):

> A compromise does not sound like something that should be added to the language.

Many others have commented that having a new language feature only to save a few characters was not a good idea, especially when it doesn't work the same way in all cases.

In the end, Matz changed the syntax from using the at sign, e.g. `@1`, to using the underscore, i.e. `_1`, and dropped the `_0`. This didn't seem to change most people's opinions one way or another.

This feature is very likely to cause confusion because it works differently for different data structures, and can also hide the arity of a block, in all but the simplest cases.

My advice would be to not use numbered parameters at all, because of the potential confusion and errors: it is not obvious what the parameter refers to because it is also not always obvious what the existing data structure is, either, and especially in the cases when the data structure changes. (I've seen these kinds of data structure changes happen many times over the years, so been there, done that, and don't want to make code worse or more bug-prone on purpose.)

### Arrays

With one-dimensional arrays, you will get what you most likely expect:

```ruby
> [1, 2, 3].each { p _1 }
1
2
3
```

And that's where it starts going downhill.

With multi-dimensional (two or more dimensions) arrays, your expectation is wrong:

```ruby
> [ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].each { p _1 }
[1, 2, 3]
[4, 5, 6]
[7, 8, 9]
```

`_1` doesn't refer to the first subelement but to the entire sub-list.

For the first subelements, you need to use indexing:

```ruby
> [ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].each { p _1[0] }
1
4
7
```

Compare it with the following more descriptive version, if really only the first subelements are interesting:

```ruby
> [ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].each {|first,| p first }
```

This works for one-dimensional arrays as well, which on the other hand might also be confusing, so don't use this for them. In most cases, you would want to give the parameter a more descriptive name than `first`, anyway.

For the second and nth subelements, you need to use:

```ruby
> [ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].each { p _2 }
2
5
8
```

There is also this even more confusing way:

```ruby
> [ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].each { p _1[1] }
2
5
8
```

It is the same as using `_2` to access the second subelements.

The maximum numbered parameter is `_9` that accesses the 9th subelement. (Using `_p1[20]` works, though, to access the 21st subelement, but it doesn't make much sense from code readability perspective.)

Yes, there are better ways to conway the meaning of the code, for example:

```ruby
> [ [1, 2, 3], [4, 5, 6], [7, 8, 9] ].each {|_, important, _| p important }
```

Here, the underscores are subelements we are not interested in, but this is not particularly pretty, either.

### Hashes

Unfortunately, hashes also don't work as expected:

```ruby
> { a: 1, b: 2, c: 3 }.each { p _1 }
[:a, 1]
[:b, 2]
[:c, 3]
```

There already exists a more meaningful and readable alternative:

```ruby
> { a: 1, b: 2, c: 3 }.each_pair {|pair| p pair}
```

To get the keys, you need use:

```ruby
> { a: 1, b: 2, c: 3 }.each { p _1[0] }
:a
:b
:c
```

Again, there exists a more meaningful and readable alternative that yields the same result:

```ruby
> { a: 1, b: 2, c: 3 }.each_key {|k| p k}
```

To get the values, you need to use:

```ruby
> { a: 1, b: 2, c: 3 }.each { p _2 }
1
2
3
```

Once more, there is a better alternative that yields the same result:

```ruby
> { a: 1, b: 2, c: 3 }.each_value {|v| p v}
```

As mentioned before, you would want to give the parameters better names than `k` or `v`, in practice.
