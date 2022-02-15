# 2022-02-15

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
