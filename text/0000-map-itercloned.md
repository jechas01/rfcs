- Feature Name: map_itercloned
- Start Date: 15/11/2017
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Addition of an intrinsic `cloned()` method on `HashMap` and `BTreeMap` `Iter`s.

# Motivation
[motivation]: #motivation

`Iterator::cloned` is intended to take a by-reference iterator and clone each
item, thereby turning it into a by-value iterator. Semantically,
`std::collections::(hash|btree)_map::Iter` are by-reference iterators over
the key/value pairs in the map. Unfortunately, the constraints on
`Iterator::cloned` make it impossible to use on the map `Iter`s. This is due
to the constraints on `Iterator::cloned` requiring the original iterator to
return references, while the map `Iter`s return owned tuples containing
references to the key/value pairs.

It is not uncommon for new users to attempt to use `cloned` to get a copy of
a map iterator's contents, and then discover that it fails to compile. For
example, [#33941](https://github.com/rust-lang/rust/issues/33941). This can
lead to some initial confusion, since it appears semantically correct at
first glance. Current solutions generally involve manual calls to `.clone()`
or attempts to emulate `.cloned()`'s behavior with the ugly `.map(|(k, v)| (k.clone(), v.clone()))`

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Example rustdocs:

```
fn cloned(&self) -> IterCloned
where K: Clone,
      V: Clone,
```

Creates an iterator which clone()s all the key/value pairs.

This is useful when you need an iterator over owned key/value pairs. This
method takes precedence over `Iterator::cloned` since the tuple returned by
this iterator does not meet the requirements of the trait method.

### Example
```
let mut m: HashMap<String, String> = HashMap::new();

a.insert("foo".into(), "bar".into());
a.insert("spam".into(), "eggs".into());

// using .map()
let m_map: HashMap<_, _> = m.iter().map(|(k, v)| (k.clone(), v.clone())).collect();

// using .cloned()
let m_cloned: HashMap<_, _> = m.iter().cloned().collect();

assert_eq!(m_map, m);
assert_eq!(m_cloned, m);
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Currently, any attempts to call `.cloned()` on one of the map `Iter` structs
will fail to compile, regardless of the map's contents. By instead providing
an intrinsic method on the struct, the desired semantic behavior can be
provided mostly transparently.

As far as implementaton goes, this will require a couple of new iterator
structs, one for each map type, which will wrap the original by-reference
iterator, and call clone on the pairs returned by the underlying iterator.

# Drawbacks
[drawbacks]: #drawbacks

Since no code using only `std` can ever successfully call `Iterator::cloned`
on a map's `Iter`, there will never be a case where the proposed new methods
will override previously used trait methods. However, it is possible that a
third-party crate could provide a trait with its own `cloned` method and
implement it for the map `Iter` structs. In that case, the new intrinsic
methods have the potential to override the previously-used trait methods and
result in either compilation failures or altered behavior.

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?

    Users already expect to be able to call `.cloned()` on their by-reference
    iterators, so at first, it appears to be a hole in the standard library
    that they can't do so for the map iterators. This design would fill that
    hole in the spirit of doing what's most expected.

- What other designs have been considered and what is the rationale for not choosing them?

    - A new method on the map structs themselves rather than their iterators, such as `iter_cloned`

        This would also work, but would require additional digging for users
        not already aware of the limitations of `Iterator::cloned` who only
        discover their need for a different method after their attempts to
        call `.cloned()` fail to compile.

    - A new trait that provides a `cloned` method for iterators that return
    borrowed key/value pairs

        This would be a bit more intrusive as it would require a new trait
        and implementation of that trait for the structs in question. It
        would also suffer from similar discoverability and expectation vs
        reality related problems as the `iter_cloned` alternative - users
        would likely only discover its existence after a failure to compile.

    - A broadening of the existing `Iterator::cloned`'s current constraints

        This is the most intrusive of the alternatives, potentially requiring
        a new trait that would allow it to accept both references and tuples
        of references as the `Iterator::Item`. I'm pretty sure this is
        possible, in theory, but it would require a much more significant
        change since the signature of `Iterator::cloned` would be modified.

- What is the impact of not doing this?

    Inexperienced users are left trying to figure out why `Iterator::cloned`
    doesn't work for their map iterators, and usually end up resorting to
    either having to call `.clone()` manually, either via a `.map(|(k, v)|
    (k.clone(), v.clone()))` to emulate the missing behavior, or later when
    consuming the iterator.

# Unresolved questions
[unresolved]: #unresolved-questions

- Are there any existing crates that provide traits that implement `cloned` for the map `Iter`s that could be problematic?
- Would a change to the existing `Iterator::cloned`'s signature be as intrusive as I imagine? or could it be done in such a way that all existing uses remain valid while opening the door to new ones, i.e. for the map `Iter`s?
- Are there any better alternatives that would be more suited to inclusion in `std`?