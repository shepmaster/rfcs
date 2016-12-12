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

A few assumptions we made:

- Measures that can be made automatically are preferred over measures that
  would need administrators, curators, or the community to spend time making
  manually.
- Measures that can be made for any crate regardless of that crate's choice of
  version control, repository host, or CI service are preferred over measures
  that would only be available or would be more easily available with git,
  GitHub, Travis, and Appveyor. Our thinking is that when this additional
  information is available, it would be better to display a badge indicating it
  since this is valuable information, but it should not influence the ranking
  of the crates.
- There are some measures, like "suitability for the current task" or "whether
  I like the way the crate is implemented" that crates.io shouldn't even
  attempt to assess, since those could potentially differ across situations for
  the same person looking for a crate.
- We assume we will be able to calculate these in a reasonable amount of time
  either on-demand or by a background job initiated on crate publish and saved
  in the database as appropriate. We think the measures we have proposed can be
  done without impacting the performance of either publishing or browsing
  crates noticeably. If this does not turn out to be the case, we will have to
  adjust the formula.

## Factors

Through [the survey we conducted][user-research], we found that when people
evaluate crates, they are looking primarily for approximate signals of:

- Ease of use
- Maintenance
- Quality

Feeding those signals are related measures of:

- Popularity
- Credibility

We propose to make it easier for people to evaluate crates along these axes by
making available measures that address each of these areas.

### Ease of use


- Percentage of public types + methods that have documentation
  - Would need an extension, something like the `missing_docs` lint, to count
    the number of public things and the number that have/don't have
    documentation.
  - Would need to unpack and run this on each package version in a background
    job started by a publish; then save the percentage in crates.io's database.

- In the crate root documentation, presence of a section headed with the word
  "Example" and containing a codeblock
  - Existing issue, seen in the survey results is that people look in both the
    README of the repo and the front page of the docs for examples
  - Increases the doc percentage score by 5%

- Presence of files in /examples
  - Future improvement: [render and link to examples in
    documentation](https://github.com/rust-lang/cargo/issues/2760)
  - Increases the doc percentage score by 5%

The percentage score plus the bonuses for examples would then be binned into
letter grades for display:

- 90-100% = A
- 80-89 = B
- 70-79 = C
- 60-69 = D
- 50-59 = E
- 0-49 = F

This is not exactly the American grading system, but we think it makes more
sense.

### Maintenance

- Last released version date: newer is better. This information is already
  available in crates.io's database; could be stored in the database and
  updated per-publish. Combined as follows, then reported as a percentage
  relative to the most released crate.
  - Number of releases in the last year - 10%
  - Number of releases in the last 6 mo - 30%
  - Number of releases in the last month - 60%

- Stable version number
  - >= 1.0.0 ranks higher than < 1.0.0
  - >=1.0.0 increases the maintenance score by 5%.

- Number of owners: more is better.
  - A GitHub group owner would count as 1.
  - Future improvement: count # of people in the github group at version
    publish time
  - >= 3 owners increases the maintenance score by 5%.

### Quality

The best we can come up with to do here without taking a stance on preferred
3rd party CI providers is:

- Are there `#[test]` annotations?
- Are there `test/` files?

### Popularity/credibility

- Number of downloads weighted by time across all versions. Combined as
  follows, then reported as a percentage relative to the most downloaded crate.
  Can be calculated as part of the [update-downloads][] background job.
  - Number of downloads in the last year - 10%
  - Number of downloads in the last 6 mo - 30%
  - Number of downloads in the last month - 60%

[update-downloads]: https://github.com/rust-lang/crates.io/blob/master/src/bin/update-downloads.rs

### Overall


When navigating to a category, we propose that crates will be ranked by default by...

- Scores are relative to all available crates



## Example

## Display

## Evaluation

We will know if this is working by...

## Out of scope

This proposal is not advocating to change the order of **search results**; those
should still be ordered by relevancy to the query based on the indexed content.

# How do we teach this?

A criticism we anticipate and that would be totally fair is that this formula
is too complex. If we go with this formula, we think it's important to make
available a clear explanation of why a crate has the score it does, for
transparency to both crate users and crate authors. [Ruby toolbox][ruby] has a
great example of what we'd like to provide.

[ruby]: #ruby-toolbox

A possible benefit of having multiple measures influence the ranking is making
it less likely that crate owners will go to the effort of gaming the formula in
order to have a higher ranking.

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

* score-effect:14: Set the effect that package scores have for the final search
  score, defaults to 15.3
* quality-weight:1: Set the weight that quality has for the each package score,
  defaults to 1.95
* popularity-weight:1: Set the weight that popularity has for the each package
  score, defaults to 3.3
* maintenance-weight:1: Set the weight that the quality has for the each
  package score, defaults to 2.05

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

## Demographics

We ran a survey for 1 week and got 134 responses. The responses we got seem to
be representative of the current Rust community: skewing heavily towards more
experienced programmers and just about evenly distributed between Rust
experience starting before 1.0, since 1.0, in the last year, and in the last 6
months, with a slight bias towards longer amounts of experience. 0 Graydons
responded to the survey.

<img src="http://i.imgur.com/huSYPyd.png" width="800" alt="Distribution of programming experience of survey repsondents, over half have been programming for over 10 years" />

<img src="http://i.imgur.com/t3kVXy9.png" width="800" alt="Distribution of Rust experience of survey respondents, slightly biased towards those who have been using Rust before 1.0 and since 1.0 over those with less than a year and less than 6 months" />

Since this matches about what we'd expect of the Rust community, we believe
this survey is representative. Given the bias towards more experience
programming, we think the answers are worthy of using to inform recommendations
crates.io will be making to programmers of all experience levels.

## Crate ranking agreement

The community ranking of the 5 crates presented in the survey for which order
people would try them out for parsing comes out to be:

1. nom
2. combine
3. and 4. peg and lalrpop, in some order
5. peresil

This chart shows how many people ranked the crates in each slot:

<img src="http://i.imgur.com/x5SOTps.png" width="800" alt="Raw votes for each crate in each slot, showing that nom and combine are pretty clearly 1 and 2, peresil is clearly 5, and peg and lalrpop both got slotted in 4th most often" />

This chart shows the cumulative number of votes: each slot contains the number
of votes each crate got for that ranking or above.

<img src="http://i.imgur.com/QsfwVNj.png" width="800" alt="" />

Whatever default ranking formula we come up with in this RFC, when applied to
these 5 crates, it should generate an order for the crates that aligns with the
community ordering. Also, not everyone will agree with the crates.io ranking,
so we should display other information and provide alternate filtering and
sorting mechanisms so that people who prioritize different attributes than the
majority of the community will be able to find what they are looking for.

## Factors considered when ranking crates

The following table shows the top 25 mentioned factors for the two free answer
sections. We asked both "Please explain what information you used to evaluate
the crates and how that information influenced your ranking." and "Was there
any information you wish was available, or that would have taken more than 15
minutes for you to get?", but some of the same factors were deemed to take too
long to find out or not be easily available, while others did consider those,
so we've ranked by the combination of mentions of these factors in both
questions.

Far and away, good documentation was the most mentioned factor people used to
evaluate which crates to try.

<table border="1" cellspacing="0" cellpadding="0">
    <tr>
        <th>
            &nbsp;
        </th>
        <th>
            Feature
        </th>
        <th>
            Used in evaluation
        </th>
        <th>
            Not available/too much time needed
        </th>
        <th>
            Total
        </th>
        <th>
            Notes
        </th>
    </tr>
    <tr>
        <td>
            1
        </td>
        <td>
            Good documentation
        </td>
        <td>
            94
        </td>
        <td>
            10
        </td>
        <td>
            104
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            2
        </td>
        <td>
            README
        </td>
        <td>
            42
        </td>
        <td>
            19
        </td>
        <td>
            61
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            3
        </td>
        <td>
            Number of downloads
        </td>
        <td>
            58
        </td>
        <td>
            0
        </td>
        <td>
            58
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            4
        </td>
        <td>
            Most recent version date
        </td>
        <td>
            54
        </td>
        <td>
            0
        </td>
        <td>
            54
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            5
        </td>
        <td>
            Obvious / easy to find usage examples
        </td>
        <td>
            37
        </td>
        <td>
            14
        </td>
        <td>
            51
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            6
        </td>
        <td>
            Examples in the repo
        </td>
        <td>
            38
        </td>
        <td>
            6
        </td>
        <td>
            44
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            7
        </td>
        <td>
            Reputation of the author
        </td>
        <td>
            36
        </td>
        <td>
            3
        </td>
        <td>
            39
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            8
        </td>
        <td>
            Description or README containing Introduction / goals / value prop / use cases
        </td>
        <td>
            29
        </td>
        <td>
            5
        </td>
        <td>
            34
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            9
        </td>
        <td>
            Number of reverse dependencies (Dependent Crates)
        </td>
        <td>
            23
        </td>
        <td>
            7
        </td>
        <td>
            30
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            10
        </td>
        <td>
            Version &gt;= 1.0.0
        </td>
        <td>
            30
        </td>
        <td>
            0
        </td>
        <td>
            30
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            11
        </td>
        <td>
            Commit activity
        </td>
        <td>
            23
        </td>
        <td>
            6
        </td>
        <td>
            29
        </td>
        <td>
            Depends on VCS
        </td>
    </tr>
    <tr>
        <td>
            12
        </td>
        <td>
            Fits use case
        </td>
        <td>
            26
        </td>
        <td>
            3
        </td>
        <td>
            29
        </td>
        <td>
            Situational
        </td>
    </tr>
    <tr>
        <td>
            13
        </td>
        <td>
            Number of dependencies (more = worse)
        </td>
        <td>
            28
        </td>
        <td>
            0
        </td>
        <td>
            28
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            14
        </td>
        <td>
            Number of open issues, activity on issues
        </td>
        <td>
            22
        </td>
        <td>
            6
        </td>
        <td>
            28
        </td>
        <td>
            Depends on GitHub
        </td>
    </tr>
    <tr>
        <td>
            15
        </td>
        <td>
            Easy to use or understand
        </td>
        <td>
            27
        </td>
        <td>
            0
        </td>
        <td>
            27
        </td>
        <td>
            Situational
        </td>
    </tr>
    <tr>
        <td>
            16
        </td>
        <td>
            Publicity (blog posts, reddit, urlo, "have I heard of it")
        </td>
        <td>
            25
        </td>
        <td>
            0
        </td>
        <td>
            25
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            17
        </td>
        <td>
            Most recent commit date
        </td>
        <td>
            17
        </td>
        <td>
            5
        </td>
        <td>
            22
        </td>
        <td>
            Dependent on VCS
        </td>
    </tr>
    <tr>
        <td>
            18
        </td>
        <td>
            Implementation details
        </td>
        <td>
            22
        </td>
        <td>
            0
        </td>
        <td>
            22
        </td>
        <td>
            Situational
        </td>
    </tr>
    <tr>
        <td>
            19
        </td>
        <td>
            Nice API
        </td>
        <td>
            22
        </td>
        <td>
            0
        </td>
        <td>
            22
        </td>
        <td>
            Situational
        </td>
    </tr>
    <tr>
        <td>
            20
        </td>
        <td>
            Mentioned using/wanting to use docs.rs
        </td>
        <td>
            8
        </td>
        <td>
            13
        </td>
        <td>
            21
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            21
        </td>
        <td>
            Tutorials
        </td>
        <td>
            18
        </td>
        <td>
            3
        </td>
        <td>
            21
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            22
        </td>
        <td>
            Number or frequency of released versions
        </td>
        <td>
            19
        </td>
        <td>
            1
        </td>
        <td>
            20
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
    <tr>
        <td>
            23
        </td>
        <td>
            Number of maintainers/contributors
        </td>
        <td>
            12
        </td>
        <td>
            6
        </td>
        <td>
            18
        </td>
        <td>
            Depends on VCS
        </td>
    </tr>
    <tr>
        <td>
            24
        </td>
        <td>
            CI results
        </td>
        <td>
            15
        </td>
        <td>
            2
        </td>
        <td>
            17
        </td>
        <td>
            Depends on CI service
        </td>
    </tr>
    <tr>
        <td>
            25
        </td>
        <td>
            Whether the crate works on nightly, stable, particular stable versions
        </td>
        <td>
            8
        </td>
        <td>
            8
        </td>
        <td>
            16
        </td>
        <td>
            &nbsp;
        </td>
    </tr>
</table>

## Relevant quotes motivating our choice of factors

### Easy to use

> 1) Documentation linked from crates.io  2) Documentation contains decent
> example on front page

> 3. "Docs Coverage" info - I'm not sure if there's a way to get that right
> now, but this is almost more important that test coverage.

> rust docs:  Is there an intro and example on the top-level page?  are the
> rustdoc examples detailed enough to cover a range of usecases?  can i avoid
> reading through the files in the examples folder?

> Documentation:
> - Is there a README? Does it give me example usage of the library? Point me
>   to more details?
> - Are functions themselves documented?
> - Does the documentation appear to be up to date?

> The GitHub repository pages, because there are no examples or detailed
> descriptions on crates.io. From the GitHub readme I first checked the readme
> itself for a code example, to get a feeling for the library. Then I looked
> for links to documentation or tutorials and examples. The crates that did not
> have this I discarded immediately.

> When evaluating any library from crates.io, I first follow the repository
> link -- often the readme is enough to know whether or not I like the actual
> library structure. For me personally a library's usability is much more
> important than performance concerns, so I look for code samples that show me
> how the library is used.    In the examples given, only peresil forces me to
> look at the actual documentation to find an example of use. I want something
> more than "check the docs" in a readme in regards to getting started.

> I would like the entire README.md of each package to be visible on crates.io
> I would like a culture where each README.md contains a runnable example

Ok, this one isn't from the survey, it's from [a Sept 2015 internals thread][]:

[a Sept 2015 internals thread]: https://users.rust-lang.org/t/lets-talk-about-ecosystem-documentation/2791/24?u=carols10cents

>> there should be indicator in Crates.io that show how much code is
>> documented, this would help with choosing well done package.
> I really love this idea! Showing a percentage or a little progress bar next
> to each crate with the proportion of public items with at least some docs
> would be a great starting point.

### Maintenance

> On nom's crates.io page I checked the version (2.0.0) and when the latest
> version came out (less than a month ago). I know that versioning is
> inconsistent across crates, but I'm reassured when a crate has V >= 1.0
> because it typically indicates that the authors are confident the crate is
> production-ready. I also like to see multiple, relatively-recent releases
> because it signals the authors are serious about maintenance.

> Answering yes scores points:  crates.io page:  Does the crate have a major
> version >= 1?  Has there been a release recently, and maybe even a steady
> stream of minor or patch-level releases?

> From github:
> * Number of commits and of contributors (A small number of commits (< 100)
> and of contributors (< 3) is often the sign of a personal project, probably
> not very much used except by its author. All other things equal, I tend to
> prefer active projects.);

### Quality

> Tests:
> - Is critical functionality well tested?
> - Is the entire package well tested?
> - Are the tests clear and descriptive?
> - Could I reimplement the library based on these tests?
> - Does the project have CI?
> - Is master green?

### Popularity/credibility

> 2) I look  at the number of download. If it is too small (~ <1000), I assume
> the crate has not yet reached a good quality. nom catches my attention
> because it has 200K download: I assume it is a high quality crate.

> 1. Compare the number of downloads: More downloads = more popular = should be
> the best

> Popularity:  - Although not being a huge factor, it can help tip the scale
> when one is more popular or well supported than another when all other
> factors are close.

### Overall

> I can't pick a most important trait because certain ones outweigh others when
> combined, etc. I.e. number of downloads is OK, but may only suggest that it's
> been around the longest. Same with number of dependent crates (which probably
> spikes number of downloads). I like a crate that is well documented, has a
> large user base (# dependent crates + downloads + stars), is post 1.0, is
> active (i.e. a release within the past 6 months?), and it helps when it's a
> prominent author (but that I feel is an unfair metric).

## Relevant bugs capturing other feedback

There was a wealth of good ideas and feedback in the survey answers, but not
all of it pertained to crate ranking directly. Commonly mentioned improvements
that could greatly help the usability and usefulness of crates.io included:

* [Rendering the README on crates.io](https://github.com/rust-lang/crates.io/issues/81)
* [Linking to docs.rs if the crate hasn't specified a Documentation link](https://github.com/rust-lang/crates.io/pull/459)
* [cargo doc should render crate examples and link to them on main documentation page](https://github.com/rust-lang/cargo/issues/2760)
* [`cargo doc` could support building/testing standalone markdown files](https://github.com/rust-lang/cargo/issues/739)
* [Allow documentation to be read from an external file](https://github.com/rust-lang/rust/issues/15470)
