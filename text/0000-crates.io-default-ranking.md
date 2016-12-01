- Feature Name: crates_io_default_ranking
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Crates.io has many useful libraries for a variety of purposes, but it's
difficult to find which crates are meant for a particular purpose and then to
decide among the available crates which one is most suitable in a particular
context. [Categorization][cat-pr] and [badges][badge-pr] are coming to
crates.io; categories help with finding a set of crates to consider and badges
help communicate attributes of crates. The question of how to order crates
within a category is still open. This RFC proposes a method of ranking crates
combining number of downloads, verison, and other attributes in order to help
decide what crate to use.

[cat-pr]: https://github.com/rust-lang/crates.io/pull/473
[badge-pr]: https://github.com/rust-lang/crates.io/pull/481

# Motivation
[motivation]: #motivation

Finding and evaluating crates can be time consuming. People already familiar
with the Rust ecosystem often know which crates are best for which puproses, but
we want to share that knowledge with everyone. For example, someone looking for
a crate to parse command-line arguments should be able to navigate to a category
for that purpose and get a list of crates to consider. This list would include
crates such as [clap-rs][] and [docopt.rs][], and the order in which they appear
should be significant and should help make the decision between the crates in
this category easier.

[clap-rs]: https://github.com/kbknapp/clap-rs
[docopt.rs]: https://github.com/docopt/docopt.rs

This helps address the goal of "Rust should provide easy access to high quality
crates" as stated in the [Rust 2017 Roadmap][roadmap].

[roadmap]: https://github.com/rust-lang/rfcs/pull/1774

# Detailed design
[design]: #detailed-design

Please see the [Appendix: Comparative Research][appendix] section for ways that
other package manager websites have solved this problem.

[appendix]: #appendix-comparative-research

This proposal is not attempting to change the order of search results; those
should still be ordered by relevancy to the query based on the indexed content.

# How do we teach this?

- How do we communicate the ranking formula (and possible changes) to crate
users and crate authors?
  - A tool for crate authors to see why their crate is ranked the way it is and
what they could do to change it.

# Drawbacks
[drawbacks]: #drawbacks

We might create a system that incentivizes attributes that are not useful, or
worse, actively harmful to the Rust ecosystem. For example, if we weight the
presence of a documentation link highly, crates might add trivial documentation
that isn't useful just to get to the top of the list.

# Alternatives
[alternatives]: #alternatives

## Manual curation

We could keep the default ranking as number of downloads, and leave further
curation to sites like [Awesome Rust][].

[Awesome Rust]: https://github.com/kud1ing/awesome-rust

We could build entirely manual ranking into crates.io, as [Ember Observer][]
does. This would be a lot of work that would need to be done by someone, but
would presumably result in higher quality evaluations and be less vulnerable to
gaming.

[Ember Observer]: https://emberobserver.com/about

## More options instead of a default

We could add filtering options for metadata, so that each user could choose, for
example, "show me only crates that work on stable" or "show me only crates that
have a version greater than 1.0".

We could add independent axes of sorting criteria in addition to the existing
alphabetical and number of downloads.

These sorting and filtering options would let each user choose exactly what's
important to them, which gives them more freedom but also more work. Crates.io
would avoid taking a position on what "best" means, which could prevent gaming
of the system since crate authors wouldn't know how users are ultimately sorting
and filtering.

# Unresolved questions
[unresolved]: #unresolved-questions

- There might be metadata about crates that we haven't thought of yet that would
be useful.
- How do we change the ranking if we try something for a while and decide it's
not what we want? Would we need another RFC?

# Appendix: Comparative Research
[comparative-research]: #appendix-comparative-research

This is how other package hosting websites handle default sorting within
categories.

## Django Packages

[Django Packages][django] has the concept of [grids][], which are large tables of packages in a particular category. Each package is a column, and each row is some attribute of packages. The default ordering from left to right appears to be Github Stars.

[django]: https://djangopackages.org/
[grids]: https://djangopackages.org/grids/

<img src="http://i.imgur.com/YAp9WYf.png" alt="Example of a Django Packages grid" width="800" style="border: 1px solid #ccc;"/>


