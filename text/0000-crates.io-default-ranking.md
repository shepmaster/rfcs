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

Please see the [Appendix: Comparative Research][comparative-research] section
for ways that other package manager websites have solved this problem, and the
[Appendix: User Research][user-research] section for results of a user research
survey we did on how people evaluate crates by hand today.

When navigating to a category, crates will be ranked by default by...

We will know if this is working by...

## Example

## Out of scope

This proposal is not advocating to change the order of **search results**; those
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

[Django Packages][django] has the concept of [grids][], which are large tables
of packages in a particular category. Each package is a column, and each row is
some attribute of packages. The default ordering from left to right appears to
be GitHub stars.

[django]: https://djangopackages.org/
[grids]: https://djangopackages.org/grids/

<img src="http://i.imgur.com/YAp9WYf.png" alt="Example of a Django Packages grid" width="800" />

## Libhunt

[Libhunt][libhunt] pulls libraries and categories from [Awesome Rust][], then
adds some metadata and navigation.

The default ranking is relative popularity, measured by GitHub stars and scaled
to be a number out of 10 as compared to the most popular crate. The other
ordering offered is dev activity, which again is a score out of 10, relative to
all other crates, and calculated by giving a higher weight to more recent
commits.

[libhunt]: https://rust.libhunt.com/

<img src="http://i.imgur.com/Yv6diFU.png" alt="Example of a Libhunt category" width="800" />

You can also choose to compare two libraries on a number of attributes:

<img src="http://i.imgur.com/HBtCH2E.png" alt="Example of comparing two crates on Libhunt" width="800" />

## Maven Repository

[Maven Repository][mvn] appears to order by the number of reverse dependencies
("# usages"):

[mvn]: http://mvnrepository.com

<img src="http://i.imgur.com/nZEQdAr.png" alt="Example of a maven repository category" width="800" />

## Pypi

[Pypi][pypi] lets you choose multiple categories, which are not only based on
topic but also other attributes like library stability and operating system:

[pypi]: https://pypi.python.org/pypi?%3Aaction=browse

<img src="http://i.imgur.com/Y3llc5m.png" alt="Example of filtering by Pypi categories" width="800" />

Once you've selected categories and click the "show all" packages in these
categories link, the packages are in alphabetical order... but the alphabet
starts over multiple times... it's unclear from the interface why this is the
case.

<img src="http://i.imgur.com/xEKGTsQ.jpg" alt="Example of Pypi ordering" width="800" />

## GitHub Showcases

To get incredibly meta, GitHub has the concept of [showcases][] for a variety
of topics, and they have [a showcase of package managers][show-pkg]. The
default ranking is by GitHub stars (cargo is 17/27 currently).

[showcases]: https://github.com/showcases
[show-pkg]: https://github.com/showcases/package-managers

<img src="http://i.imgur.com/SCvKQi2.png" alt="Example of a GitHub showcase" width="800" />

## Ruby toolbox

[Ruby toolbox][rb] sorts by a relative popularity score, which is calculated
from a combination of GitHub stars/watchers and number of downloads:

[rb]: https://www.ruby-toolbox.com

<img src="http://i.imgur.com/5Qt03n3.png" alt="How Ruby Toolbox's popularity ranking is calculated" width="800" />

Category pages have a bar graph showing the top gems in that category, which
looks like a really useful way to quickly see the differences in relative
popularity. For example, this shows nokogiri is far and away the most popular
HTML parser:

<img src="http://i.imgur.com/tj8emlu.png" alt="Example of Ruby Toolbox ordering" width="800" />

Also of note is the amount of information shown by default, but with a
magnifying glass icon that, on hover or tap, reveals more information without a
page load/reload:

<img src="http://i.imgur.com/0NPi6ct.png" alt="Expanded Ruby Toolbox info" width="800" />

## npms

While [npms][] doesn't have categories, its search appears to do some exact
matching of the query and then rank the rest of the results [weighted][] by
three different scores:

* score-effect:14: Set the effect that package scores have for the final search score, defaults to 15.3
* quality-weight:1: Set the weight that quality has for the each package score, defaults to 1.95
* popularity-weight:1: Set the weight that popularity has for the each package score, defaults to 3.3
* maintenance-weight:1: Set the weight that the quality has for the each package score, defaults to 2.05

[npms]: https://npms.io
[weighted]: https://api-docs.npms.io/

<img src="http://i.imgur.com/aWMeNv5.png" alt="Example npms search results" width="800" />

There are [many factors][] that go into the three scores, and more are planned
to be added in the future. Implementation details are available in the
[architecture documentation][].

[many factors]: https://npms.io/about
[architecture documentation]: https://github.com/npms-io/npms-analyzer/blob/master/docs/architecture.md

<img src="http://i.imgur.com/0i897ts.png" alt="Explanation of the data analyzed by npms" width="800" />

## Package Control (Sublime)

[Package Control][] is for Sublime Text packages. It has Labels that are
roughly equivalent to categories:

[Package Control]: https://packagecontrol.io/

<img src="http://i.imgur.com/81PGbFM.png" alt="Package Control homepage showing Labels like language syntax, snippets" width="800" />

The only available ordering within a label is alphabetical, but each result has
the number of downloads plus badges for Sublime Text version compatibility, OS
compatibility, Top 25/100, and new/trending:

<img src="http://i.imgur.com/KtWcOXV.png" alt="Sample Package Control list of packages within a label, sorted alphabetically" width="800" />

# Appendix: User Research
[user-research]: #appendix-user-research
