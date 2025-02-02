---
template: page.html
---

There is no point building software if you can't install it.
Inspired by the <span i="Comprehensive TeX Archive Network">Comprehensive TeX Archive Network</span> [CTAN][ctan],
most languages now have an online archive from which developers can download packages.
Each package typically has a name and one or more version(s);
each version may have a list of dependencies,
and the package may specify a version or range of versions for each dependency.

Downloading files requires some web programming that is out of scope for this book,
while installing those files in the right places
uses the systems programming skills of <a section="systems-programming"/>.
The piece we are missing is a way to figure out exactly what versions of different packages to install
in order to create a consistent setup.
If packages A and B require different versions of C,
it might not be possible to use A and B together.
On the other hand,
if each one requires a range of versions of C and those ranges overlap,
we might be able to find a combination that works---at least,
until we try to install packages D and E.

We *could* install every package's dependencies separately with it;
the disk space wouldn't be much of an obstacle,
but loading dozens of copies of the same package into the browser
would slow applications down.
This chapter therefore explores how to find a workable installation or prove that there isn't one.
It is based in part on [this tutorial][package-manager-tutorial] by <span i="Nison, Maël">[Maël Nison][nison-mael]</span>.

> ### Satisfiability
>
> What we are trying to do is find a version for each package
> that makes the assertion "P is compatible with all its dependencies" true
> for every package P.
> The general-purpose tools for doing this are called <span g="sat_solver" i="satisfiability; SAT solver">SAT solvers</span>
> because they determine whether there is some assignment of values
> that satisfies the claim (i.e., makes it true).
> Finding a solution can be extremely hard in the general case,
> so most SAT solvers use heuristics to try to reduce the work.

## What is semantic versioning? {#package-manager-semver}

Most software projects use <span g="semantic_versioning" i="semantic versioning">semantic versioning</span> for software releases.
Each version number consists of three integers X.Y.Z,
where X is the major version,
Y is the minor version,
and Z is the <span g="patch" i="patch number; semantic versioning!patch number">patch</span> number.
(The [full specification][semver-spec] allows for more fields,
but we will ignore them in this tutorial.)

A package's authors increment its major version number
every time something changes in a way that makes the package incompatible with previous versions
For example,
if they add a required parameter to a function,
then code built for the old version will fail or behave unpredictably with the new one.
The minor version number is incremented when new functionality
is <span g="backward_compatible" i="backward compatibility">backward-compatible</span>---i.e.,
it won't break any existing code---and the patch number is changed
for backward-compatible bug fixes that don't add any new features.

The notation for specifying a project's dependencies looks a lot like arithmetic:
`>= 1.2.3` means "any version from 1.2.3 onward",
`< 4` means "any version before 4.anything",
and `1.0 - 3.1` means "any version in the specified range (including patches)".
Note that version 2.1 is greater than version 1.99:
no matter how large a minor version number becomes,
it never spills over into the major version number
in the way that minutes add up to hours or months add up to years.

It isn't hard to write a few simple comparisons for semantic version identifiers,
but getting all the different cases right is almost as tricky as handling dates and times correctly,
so we will rely on the [`semver`][node-semver] module.
`semver.valid('1.2.3')` checks that `1.2.3` is a valid version identifier,
while `semver.satisfies('2.2', '1.0 - 3.1')` checks that its first argument
is compatible with the range specified in its second.

## How can we find a consistent set of packages? {#package-manager-consistent}

Imagine that each package we need is represented as an axis on a multi-dimensional grid,
with its versions as the tick marks
(<a figure="package-manager-allowable"/>).
Each point on the grid is a possible combination of package versions.
We can block out regions of this grid using the constraints on the package versions;
whatever points are left when we're done represent legal combinations.

<figure id="package-manager-allowable">
  <img src="figures/allowable.svg" alt="Allowable versions" />
  <figcaption>Finding allowable combinations of package versions.</figcaption>
</figure>

For example,
suppose we have the set of requirements shown in <a table="package-manager-example-dependencies"/>.
There are 18 possible configurations
(2 for X × 3 for Y × 3 for Z)
but 16 are excluded by various incompatibilities.
Of the two remaining possibilities,
X/2 + Y/3 + Z/3 is strictly greater than X/2 + Y/2 + Z/2,
so we would probably choose the former
(<a table="package-manager-example-result"/>).
if we wound up with A/1 + B/2 versus A/2 + B/1,
we would need to add rules for resolving ties.

> ### Reproducibility
>
> No matter what kind of software you build,
> a given set of inputs should always produce the same output;
> if they don't,
> testing is much more difficult (or impossible) <cite>Taschuk2017</cite>.
> There may not be a strong reason to prefer one mutually-compatible set of packages over another,
> but a package manager should still resolve the ambiguity the same way every time.
> It may not be what everyone wants,
> but at least they will be unhappy for the same reasons everywhere.
> This is why [NPM][npm] has both `package.json` and a `package-lock.json` files:
> the former is written by the user and specifies what they *want*,
> while the latter is created by the package manager and specifies exactly what they *got*.
> If you want to reproduce someone else's setup for debugging purposes,
> you should install what is described in the latter file.

<div class="table" id="package-manager-example-dependencies" cap="Example package dependencies.">
| Package | Requires |
| ------- | -------- |
| X/1     | Y/1-2    |
| X/1     | Z/1      |
| X/2     | Y/2-3    |
| X/2     | Z/1-2    |
| Y/1     | Z/2      |
| Y/2     | Z/2-3    |
| Y/3     | Z/3      |
| Z/1     |          |
| Z/2     |          |
| Z/3     |          |
</div>

<div class="table" id="package-manager-example-result" cap="Result for example package dependencies.">
|   X |   Y |   Z | Excluded  |
| --- | --- | --- | --------- |
|   1 |   1 |   1 | Y/1 - Z/1 |
|   1 |   1 |   2 | X/1 - Z/2 |
|   1 |   1 |   3 | X/1 - Z/3 |
|   1 |   2 |   1 | Y/2 - Z/1 |
|   1 |   2 |   2 | X/1 - Z/2 |
|   1 |   2 |   3 | X/1 - Z/3 |
|   1 |   3 |   1 | X/1 - Y/3 |
|   1 |   3 |   2 | X/1 - Y/3 |
|   1 |   3 |   3 | X/1 - Y/3 |
|   2 |   1 |   1 | X/2 - Y/1 |
|   2 |   1 |   2 | X/2 - Y/1 |
|   2 |   1 |   3 | X/2 - Y/1 |
|   2 |   2 |   1 | Y/2 - Z/1 |
|   2 |   2 |   2 |           |
|   2 |   2 |   3 | X/2 - Z/3 |
|   2 |   3 |   1 | Y/3 - Z/1 |
|   2 |   3 |   2 | Y/3 - Z/2 |
|   2 |   3 |   3 | X/2 - Z/3 |
</div>

To construct <a table="package-manager-example-dependencies"/>
we find the <span i="transitive closure">transitive closure</span> of all packages plus all of their dependencies.
We then pick two packages and create a list of their valid pairs.
Choosing a third package,
we cross off pairs that can't be satisfied
to leave triples of legal combinations.
We repeat this until all packages are included in our table.

In the worst case this procedure will create
a <span g="combinatorial_explosion" i="combinatorial explosion">combinatorial explosion</span> of possibilities.
Smart algorithms will try to add packages to the mix
in an order that minimize the number of new possibilities at each stage,
or create pairs and then combine them to create pairs of pairs and so on.
Our algorithm will be simpler (and therefore slower),
but illustrates the key idea.

## How can we satisfy constraints? {#package-manager-constraints}

To avoid messing around with parsers,
our programs reads a JSON data structure describing the problem;
a real package manager would read the <span g="manifest" i="manifest (of package); package manifest">manifests</span> of the packages in question
and construct a similar data structure.
We will stick to single-digit version numbers for readability,
and will use this as our first test case:

<div class="include" file="double-chained.json" />

> ### Comments
>
> If you ever design a data format,
> please include a standard way for people to add comments,
> because they will always want to.
> YAML has this,
> but JSON and CSV don't.

To check if a combination of specific versions of packages is compatible with a manifest,
we add each package to our active list in turn and look for violations.
If there aren't any more packages to add and we haven't found a violation,
then what we have must be a legal configuration.

<div class="include" file="sweep.js" omit="allows" />

The simplest way to find configurations is to sweep over all possibilities.
For debugging purposes,
our function prints possibilities as it goes:

<div class="include" file="sweep.js" keep="allows" />

If we run this program on the two-package example shown earlier we get this output:

<div class="include" pat="sweep-double-chained.*" fill="sh out" />

When we run it on our triple-package example we get this:

<div class="include" pat="sweep-triple.*" fill="sh out" />

This works,
but it is doing a lot of unnecessary work.
If we sort the output by the case that caught the exclusion
it turns out that 9 of the 17 exclusions are redundant rediscovery of a previously-known problem
<a table="package-manager-exclusions"/>.

<div class="table" id="package-manager-exclusions" cap="Package exclusions.">
| Excluded  |   X |   Y |   Z |
| --------  | --- | --- | --- |
| X/1 - Y/3 |   1 |   3 |   1 |
| …         |   1 |   3 |   2 |
| …         |   1 |   3 |   3 |
| X/1 - Z/2 |   1 |   1 |   2 |
| …         |   1 |   2 |   2 |
| X/1 - Z/3 |   1 |   1 |   3 |
| …         |   1 |   2 |   3 |
| X/2 - Y/1 |   2 |   1 |   1 |
| …         |   2 |   1 |   2 |
| …         |   2 |   1 |   3 |
| X/2 - Z/3 |   2 |   2 |   3 |
| …         |   2 |   3 |   3 |
| Y/1 - Z/1 |   1 |   1 |   1 |
| Y/2 - Z/1 |   1 |   2 |   1 |
| …         |   2 |   2 |   1 |
| Y/3 - Z/1 |   2 |   3 |   1 |
| …         |   2 |   3 |   2 |
|           |   2 |   2 |   2 |
</div>

## How can we do less work? {#package-manager-optimize}

In order to make this more efficient we need to <span g="prune" i="prune (a search tree)">prune</span> the search tree
as we go along
(<a figure="package-manager-pruning"/>).
After all,
if we know that X and Y are incompatible,
there is no need to check Z as well.

<figure id="package-manager-pruning">
  <img src="figures/pruning.svg" alt="Pruning the search tree" />
  <figcaption>Pruning options in the search tree to reduce work.</figcaption>
</figure>

This version of the program collects possible solutions and displays them at the end.
It only keeps checking a partial solution if what it has found so far looks good:

<div class="include" file="prune.js" omit="compatible" />

The `compatible` function checks to see if adding something will leave us with a consistent configuration:

<div class="include" file="prune.js" keep="compatible" />

Checking as we go gets us from 18 complete solutions to 11.
One is workable
and two are incomplete---they represent 6 possible complete solutions that we didn't need to finish:

<div class="include" file="prune-triple.out" />

Another way to look at the work is the number of steps in the search.
The full search had 18×3 = 54 steps.
Pruning leaves us with (12×3) + (2×2) = 40 steps
so we have eliminated roughly 1/4 of the work.

What if we searched in the reverse order?

<div class="include" file="reverse.js" />

<div class="include" file="reverse-triple.out" />

Now we have (8×3) + (5×2) = 34 steps,
i.e.,
we have eliminated roughly 1/3 of the work.
That may not seem like a big difference,
but if we go five levels deep at the same rate
it cuts the work in half.
There are lots of <span g="heuristic">heuristics</span> for searching trees;
none are guaranteed to give better performance in every case,
but most give better performance in most cases.

> ### What research is for
>
> <span i="SAT solver">SAT solvers</span> are like regular expression libraries and random number generators:
> it is the work of many lifetimes to create ones that are both fast and correct.
> A lot of computer science researchers devote their careers to highly-specialized topics like this.
> The debates often seem esoteric to outsiders,
> and most ideas turn out to be dead ends,
> but even small improvements in fundamental tools can have a profound impact.

## Exercises {#package-manager-exercises}

### Comparing semantic versions {.exercise}

Write a function that takes an array of semantic version specifiers
and sorts them in ascending order.
Remember that `2.1` is greater than `1.99`.

### Parsing semantic versions {.exercise}

Using the techniques of <a section="regex-parser"/>,
write a parser for a subset of the [semantic versioning specification][semver-spec].

### Using scoring functions {.exercise}

Many different combinations of package versions can be mutually compatible.
One way to decide which actual combination to install
is to create a <span g="scoring_function">scoring function</span>
that measures how good or bad a particular combination is.
For example,
a function could measure the "distance" between two versions as:

```js
const score (X, Y) => {
  if (X.major !== Y.major) {
    return 100 * abs(X.major - Y.major)
  } else if (X.minor !== Y.minor) {
    return 10 * abs(X.minor - Y.minor)
  } else {
    return abs(X.patch - Y.patch)
  }
}
```

1.  Implement a working version of this function
    and use it to measure the total distance between
    the set of packages found by the solver
    and the set containing the most recent version of each package.

2.  Explain why this doesn't actually solve the original problem.

### Using full semantic versions {.exercise}

Modify the constraint solver to use full semantic versions instead of single digits.

### Regular releases {.exercise}

Some packages release new versions on a regular cycle,
e.g.,
Version 2021.1 is released on March 1 of 2021,
Version 2021.2 is released on September 1 of that year,
version 2022.1 is released on March 1 of the following year,
and so on.

1.  How does this make package management easier?

2.  How does it make it more difficult?

### Writing unit tests {.exercise}

Write unit tests for the constraint solver using Mocha.

### Generating test fixtures {.exercise}

Write a function that creates fixtures for testing the constraint solver:

1.  Its first argument is an object whose keys are (fake) package names
    and whose values are integers indicating the number of versions of that package
    to include in the test set,
    such as `{'left': 3, 'middle': 2, 'right': 15}`.
    Its second argument is a <span g="seed">seed</span> for random number generation.

2.  It generates one valid configuration,
    such as `{'left': 2, 'middle': 2, 'right': 9}`.
    (This is to ensure that there is at least one installable set of packages.)

3.  It then generates random constraints between the packages.
    (These may or may not result in other installable combinations.)
    When this is done,
    it adds constraints so that the valid configuration from the previous step is included.

### Searching least first {.exercise}

Rewrite the constraint solver so that it searches packages
by looking at those with the fewest available versions first.
Does this reduce the amount of work done for the small examples in this chapter?
Does it reduce the amount of work done for larger examples?

### Using generators {.exercise}

Rewrite the constraint solver to use generators.

### Using exclusions {.exercise}

1.  Modify the constraint solver so that
    it uses a list of package exclusions instead of a list of package requirements,
    i.e.,
    its input tells it that version 1.2 of package Red
    can *not* work with versions 3.1 and 3.2 of package Green
    (which implies that Red 1.2 can work with any other versions of Green).

2.  Explain why package managers aren't built this way.
