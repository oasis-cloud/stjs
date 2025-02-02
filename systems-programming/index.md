---
template: page.html
---

The biggest difference between JavaScript and most other programming languages
is that many operations in JavaScript are <span g="asynchronous" i="asynchronous execution; execution!asynchronous">asynchronous</span>.
Its designers didn't want browsers to freeze while waiting for data to arrive or for users to click on things,
so operations that might be slow are implemented by describing now what to do later.
And since anything that touches the hard drive is slow from a processor's point of view,
[Node][nodejs] implements <span g="filesystem" i="filesystem operations">filesystem</span> operations the same way.

> ### How slow is slow?
>
> <cite>Gregg2020</cite> used the analogy in <a table="systems-programming-times"/>
> to show how long it takes a computer to do different things
> if we imagine that one CPU cycle is equivalent to one second.

<div class="table" id="systems-programming-times" cap="Computer operation times at human scale.">
| Operation | Actual Time | Would Be… |
| --------- | ----------- | --------- |
| 1 CPU cycle | 0.3 nsec | 1 sec |
| Main memory access | 120 nsec | 6 min |
| Solid-state disk I/O | 50-150 μsec | 2-6 days |
| Rotational disk I/O | 1-10 msec | 1-12 months |
| Internet: San Francisco to New York | 40 msec | 4 years |
| Internet: San Francisco to Australia | 183 msec | 19 years |
| Physical system reboot | 5 min | 32,000 years |
</div>

Early JavaScript programs used <span g="callback" i="callback function">callback functions</span> to describe asynchronous operations,
but as we're about to see,
callbacks can be hard to understand even in small programs.
In 2015,
the language's developers standardized a higher-level tool called promises
to make callbacks easier to manage,
and more recently they have added new keywords called `async` and `await` to make it easier still.
We need to understand all three layers in order to debug things when they go wrong,
so this chapter explores callbacks,
while <a section="async-programming"/> shows how promises and `async`/`await` work.
This chapter also shows how to read and write files and directories with Node's standard libraries,
because we're going to be doing that a lot.

## How can we list a directory? {#systems-programming-ls}

To start,
let's try listing the contents of a directory the way we would in <span i="Python">[Python][python]</span>
or <span i="Java">[Java][java]</span>:

<div class="include" file="list-dir-wrong.js" />

<!-- continue -->
We use <span i="import module"><code>import <em>module</em> from 'source'</code></span> to load the library <code><em>source</em></code>
and assign its contents to <code><em>module</em></code>.
After that,
we can refer to things in the library using <code><em>module.component</em></code>
just as we refer to things in any other object.
We can use whatever name we want for the module,
which allows us to give short nicknames to libraries with long names;
we will take advantage of this in future chapters.

> ### `require` versus `import`
>
> In 2015, a new version of JavaScript called ES6 introduced
> the keyword <span i="import vs. require; require vs. import">`import`</span> for importing modules.
> It improves on the older `require` function in several ways,
> but Node still uses `require` by default.
> To tell it to use `import`,
> we have added `"type": "module"` at the top level of our Node `package.json` file.

Our little program uses the [`fs`][node-fs] library
which contains functions to create directories, read or delete files, etc.
(Its name is short for "filesystem".)
We tell the program what to list using <span g="command_line_argument" i="command-line argument">command-line arguments</span>,
which Node automatically stores in an array called <span i="process.argv">`process.argv`</span>.
`process.argv[0]` is the name of the program used to run our code (in this case `node`),
while `process.argv[1]` is the name of our program (in this case `list-dir-wrong.js`);
the rest of `process.argv` holds whatever arguments we gave at the command line when we ran the program,
so `process.argv[2]` is the first argument after the name of our program (<a figure="systems-programming-process-argv"/>):

<figure id="systems-programming-process-argv">
  <img src="figures/process-argv.svg" alt="Command-line arguments in `process.argv`" />
  <figcaption>How Node stores command-line arguments in <code>process.argv</code>.</figcaption>
</figure>

If we run this program with the name of a directory as its argument,
`fs.readdir` returns the names of the things in that directory as an array of strings.
The program uses `for (const name of results)` to loop over the contents of that array.
We could use `let` instead of `const`,
but it's good practice to declare things as <span i="const declaration!advantages of">`const`</span> wherever possible
so that anyone reading the program knows the variable isn't actually going to vary---doing
this reduces the <span g="cognitive_load" i="cognitive load">cognitive load</span> on people reading the program.
Finally,
<span i="console.log">`console.log`</span> is JavaScript's equivalent of other languages' `print` command;
its strange name comes from the fact that
its original purpose was to create <span g="log_message">log messages</span> in the browser <span g="console">console</span>.

Unfortunately,
our program doesn't work:

<div class="include" pat="list-dir-wrong.*" fill="sh out" />

<!-- continue -->
The error message comes from something we didn't write whose source we would struggle to read.
If we look for the name of our file (`list-dir-wrong.js`)
we see the error occurred on line 4;
everything above that is inside `fs.readdir`,
while everything below it is Node loading and running our program.

The problem is that `fs.readdir` doesn't return anything.
Instead,
its documentation says that it needs a callback function
that tells it what to do when data is available,
so we need to explore those in order to make our program work.

> ### A theorem
>
> 1.  Every program contains at least one bug.
> 2.  Every program can be made one line shorter.
> 3.  Therefore, every program can be reduced to a single statement which is wrong.
>
> --- variously attributed

## What is a callback function? {#systems-programming-callback}

JavaScript uses a <span g="single_threaded" i="single-threaded execution; execution!single-threaded">single-threaded</span> programming model:
as the introduction to this lesson said,
it splits operations like file I/O into "please do this" and "do this when data is available".
`fs.readdir` is the first part,
but we need to write a function that specifies the second part.

JavaScript saves a reference to this function
and calls with a specific set of parameters when our data is ready
(<a figure="systems-programming-callbacks"/>).
Those parameters defined a standard <span g="protocol" i="protocol!API as; API!as protocol">protocol</span>
for connecting to libraries,
just like the USB standard allows us to plug hardware devices together.

<figure id="systems-programming-callbacks">
  <img src="figures/callbacks.svg" alt="Running callbacks" />
  <figcaption>How JavaScript runs callback functions.</figcaption>
</figure>

This corrected program gives `fs.readdir` a callback function called `listContents`:

<div class="include" file="list-dir-function-defined.js" />

<!-- continue -->
<span i="callback function!conventions for">Node callbacks</span>
always get an error (if there is any) as their first argument
and the result of a successful function call as their second.
The function can tell the difference by checking to see if the error argument is `null`.
If it is, the function lists the directory's contents with `console.log`,
otherwise, it uses `console.error` to display the error message.
Let's run the program with the <span g="current_working_directory">current working directory</span>
(written as '.')
as an argument:

<div class="include" pat="list-dir-function-defined.*" fill="sh slice.out" />

Nothing that follows will make sense if we don't understand
the order in which Node executes the statements in this program
(<a figure="systems-programming-execution-order"/>):

1.  Execute the first line to load the `fs` library.

1.  Define a function of two parameters and assign it to `listContents`.
    (Remember, a function is just another kind of data.)

1.  Get the name of the directory from the command-line arguments.

1.  Call `fs.readdir` to start a filesystem operation,
    telling it what directory we want to read and what function to call when data is available.

1.  Print a message to show we're at the end of the file.

1.  Wait until the filesystem operation finishes (this step is invisible).

1.  Run the callback function, which prints the directory listing.

<figure id="systems-programming-execution-order">
  <img src="figures/execution-order.svg" alt="Callback execution order" />
  <figcaption>When JavaScript runs callback functions.</figcaption>
</figure>

## What are anonymous functions? {#systems-programming-anonymous}

Most JavaScript programmers wouldn't define the function `listContents`
and then pass it as a callback.
Instead,
since the callback is only used in one place,
it is more <span g="idiomatic">idiomatic</span>
to define it where it is needed
as an <span g="anonymous_function" i="anonymous function; function!anonymous">anonymous function</span>.
This makes it easier to see what's going to happen when the operation completes,
though it means the order of execution is quite different from the order of reading
(<a figure="systems-programming-anonymous-functions"/>).
Using an anonymous function gives us the final version of our program:

<div class="include" file="list-dir-function-anonymous.js" />

<figure id="systems-programming-anonymous-functions">
  <img src="figures/anonymous-functions.svg" alt="Anonymous functions as callbacks" />
  <figcaption>How and when JavaScript creates and runs anonymous callback functions.</figcaption>
</figure>

> ### Functions are data
>
> As we noted above,
> a function is just <span i="code!as data">another kind of data</span>.
> Instead of being made up of numbers, characters, or pixels, it is made up of instructions,
> but these are stored in memory like anything else.
> Defining a function on the fly is no different from defining an array in-place using `[1, 3, 5]`,
> and passing a function as an argument to another function is no different from passing an array.
> We are going to rely on this insight over and over again in the coming lessons.

## How can we select a set of files? {#systems-programming-fileset}

Suppose we want to copy some files instead of listing a directory's contents.
Depending on the situation
we might want to copy only those files given on the command line
or all files except some explicitly excluded.
What we *don't* want to have to do is list the files one by one;
instead,
we want to be able to write patterns like `*.js`.

To find files that match patterns like that,
we can use the [`glob`][node-glob] module.
(To <span g="globbing" i="globbing">glob</span> (short for "global") is an old Unix term for matching a set of files by name.)
The `glob` module provides a function that takes a pattern and a callback
and does something with every filename that matched the pattern:

<div class="include" pat="glob-all-files.*" fill="js slice.out" />

The leading `**` means "recurse into subdirectories",
while `*.*` means "any characters followed by '.' followed by any characters"
(<a figure="systems-programming-globbing"/>).
Names that don't match `*.*` won't be included,
and by default,
neither are names that start with a '.' character.
This is another old Unix convention:
files and directories whose names have a leading '.'
usually contain configuration information for various programs,
so most commands will leave them alone unless told to do otherwise.

<figure id="systems-programming-globbing">
  <img src="figures/globbing.svg" alt="Matching filenames with `glob`" />
  <figcaption>Using <code>glob</code> patterns to match filenames.</figcaption>
</figure>

This program works,
but we probably don't want to copy Emacs backup files whose names end with `~`.
We can get rid of them by <span g="filter" i="globbing!filtering results">filtering</span> the list that `glob` returns:

<div class="include" pat="glob-get-then-filter-pedantic.*" fill="js slice.out" />

<span i="Array.filter">`Array.filter`</span> creates a new array
containing all the items of the original array that pass a test
(<a figure="systems-programming-array-filter"/>).
The test is specified as a callback function
that `Array.filter` calls once once for each item.
This function must return a <span g="boolean">Boolean</span>
that tells `Array.filter` whether to keep the item in the new array or not.
`Array.filter` does not modify the original array,
so we can filter our original list of filenames several times if we want to.

<figure id="systems-programming-array-filter">
  <img src="figures/array-filter.svg" alt="Using `Array.filter`" />
  <figcaption>Selecting array elements using <code>Array.filter</code>.</figcaption>
</figure>

We can make our globbing program more idiomatic by
removing the parentheses around the single parameter
and writing just the expression we want the function to return:

<div class="include" file="glob-get-then-filter-idiomatic.js" />

However,
it turns out that `glob` will filter for us.
According to its documentation,
the function takes an `options` object full of key-value settings
that control its behavior.
This is another common pattern in Node libraries:
rather than accepting a large number of rarely-used parameters,
a function can take a single object full of settings.

If we use this,
our program becomes:

<div class="include" file="glob-filter-with-options.js" />

<!-- continue -->
Notice that we don't quote the key in the `options` object.
The keys in objects are almost always strings,
and if a string is simple enough that it won't confuse the parser,
we don't need to put quotes around it.
Here,
"simple enough" means "looks like it could be a variable name",
or equivalently "contains only letters, digits, and the underscore".

> ### No one knows everything
>
> We combined `glob.glob` and `Array.filter` in our functions for more than a year
> before someone pointed out the `ignore` option for `glob.glob`.
> This shows:
>
> 1.  Life is short,
>     so most of us find a way to solve the problem in front of us
>     and re-use it rather than looking for something better.
>
> 2.  Code reviews aren't just about finding bugs:
>     they are also the most effective way to transfer knowledge between programmers.
>     Even if someone is much more experienced than you,
>     there's a good chance you might have stumbled over a better way to do something
>     than the one they're using (see point #1 above).

To finish off our globbing program,
let's specify a source directory on the command line and include that in the pattern:

<div class="include" file="glob-with-source-directory.js" />

<!-- continue -->
This program uses <span g="string_interpolation" i="string interpolation">string interpolation</span>
to insert the value of `srcDir` into a string.
The template string is written in back quotes,
and JavaScript converts every expression written as `${expression}` to text.
We could create the pattern by concatenating strings using
`srcDir + '/**/*.*'`,
but most programmers find interpolation easier to read.

## How can we copy a set of files? {#systems-programming-copy}

If we want to copy a set of files instead of just listing them
we need a way to create the <span g="path">paths</span> of the files we are going to create.
If our program takes a second argument that specifies the desired output directory,
we can construct the full output path by replacing the name of the source directory with that path:

<div class="include" file="glob-with-dest-directory.js" />

<!-- continue -->
This program uses <span g="destructuring_assignment" i="destructuring assignment; assignment!destructuring">destructuring assignment</span>
to create two variables at once
by unpacking the elements of an array
(<a figure="systems-programming-destructuring-assignment"/>).
It only works if the array contains the enough elements,
i.e.,
if both a source and destination are given on the command line;
we'll add a check for that in the exercises.

<figure id="systems-programming-destructuring-assignment">
  <img src="figures/destructuring-assignment.svg" alt="Matching values with destructuring assignment" />
  <figcaption>Assigning many values at once by destructuring.</figcaption>
</figure>

A more serious problem is that
this program only works if the destination directory already exists:
`fs` and equivalent libraries in other languages usually won't create directories for us automatically.
The need to do this comes up so often that there is a function called `ensureDir` to do it:

<div class="include" file="glob-ensure-output-directory.js" />

Notice that we import from `fs-extra` instead of `fs`;
the [`fs-extra`][node-fs-extra] module provides some useful utilities on top of `fs`.
We also use [`path`][node-path] to manipulate pathnames
rather than concatenating or interpolating strings
because there are a lot of tricky <span g="edge_case">edge cases</span> in pathnames
that the authors of that module have figured out for us.

> ### Using distinct names
>
> We are now calling our command-line arguments `srcRoot` and `dstRoot`
> rather than `srcDir` and `dstDir`.
> We originally used `dstDir` as both
> the name of the top-level destination directory (from the command line)
> and the name of the particular output directory to create.
> This was legal,
> since every function creates
> a new <span g="scope" i="scope!of variable definitions; variable definition!scope">scope</span>,
> but hard for people to understand.

Our file copying program currently creates empty destination directories
but doesn't actually copy any files.
Let's use `fs.copy` to do that:

<div class="include" file="copy-file-unfiltered.js" />

The program now has three levels of callback
(<a figure="systems-programming-triple-callback"/>):

1.  When `glob` has data, do things and then call `ensureDir`.

1.  When `ensureDir` completes, copy a file.

1.  When `copy` finishes, check the error status.

<figure id="systems-programming-triple-callback">
  <img src="figures/triple-callback.svg" alt="Three levels of callback" />
  <figcaption>Three levels of callback in the running example.</figcaption>
</figure>

Our program looks like it should work,
but if we try to copy everything in the directory containing these lessons
we get an error message:

<div class="include" pat="copy-file-unfiltered.*" fill="sh out" />

The problem is that `node_modules/fs.stat` and `node_modules/fs.walk` match our globbing expression,
but are directories rather than files.
To prevent our program from trying to use `fs.copy` on directories,
we must use `fs.stat` to get the properties of the thing whose name `glob` has given us
and then check if it's a file.
The name "stat" is short for "status",
and since the status of something in the filesystem can be very complex,
<span i="fs.stat">`fs.stat`</span> returns [an object with methods that can answer common questions][node-fs-stats].

Here's the final version of our file copying program:

<div class="include" file="copy-file-filtered.js" />

<!-- continue -->
It works,
but four levels of asynchronous callbacks is hard for humans to understand.
<a section="async-programming"/> will introduce a pair of tools
that make code like this easier to read.

## Exercises {#systems-programming-exercises}

### Where is Node? {.exercise}

Write a program called `wherenode.js` that prints the full path to the version of Node is is run with.

### Tracing callbacks {.exercise}

In what order does the program below print messages?

<div class="include" file="x-trace-callback/trace.js"/>

### Tracing anonymous callbacks {.exercise}

In what order does the program below print messages?

<div class="include" file="x-trace-anonymous/trace.js"/>

### Checking arguments {.exercise}

Modify the file copying program to check that it has been given the right number of command-line arguments
and to print a sensible error message (including a usage statement) if it hasn't.

### Significant entries {.exercise}

`count-lines-histogram.js` displays many zeroes and gives no visual sense of how large entries are.
Modify it so that:

1.  When it is run with the `--nonzero` flag only non-zero values are shown.

2.  When it is run with the `--graphical` flag the numeric values are replaced with rows of asterisks.

3.  If both flags are given the program prints an error message instead of running.

### Glob patterns {.exercise}

What filenames does each of the following glob patterns match?

-   `results-[0123456789].csv`
-   `results.(tsv|csv)`
-   `results.dat?`
-   `./results.data`

### Filtering arrays {.exercise}

Fill in the blank in the code below so that it runs correctly.
Note: you can compare strings in JavaScript using `<`, `>=`, and other operators,
so that (for example) `person.personal > 'P'` is `true`
if someone's personal name starts with a letter that comes after 'P' in the alphabet.

<div class="include" pat="x-array-filter/filter.*" fill="js txt"/>

### String interpolation {.exercise}

Fill in the code below so that it prints the message shown.

<div class="include" pat="x-string-interpolation/interpolate.*" fill="js txt"/>

### Destructuring assignment {.exercise}

What is assigned to each named variable in each statement below?

1.  `const first = [10, 20, 30]`
1.  `const [first, second] = [10, 20, 30]`
1.  `const [first, second, third] = [10, 20, 30]`
1.  `const [first, second, third, fourth] = [10, 20, 30]`
1.  `const {left, right} = {left: 10, right: 30}`
1.  `const {left, middle, right} = {left: 10, middle: 20, right: 30}`

### Counting lines {.exercise}

Write a program called `lc` that counts and reports the number of lines in one or more files and the total number of lines,
so that `lc a.txt b.txt` displays something like:

```txt
a.txt 475
b.txt 31
total 506
```

### Renaming files {.exercise}

Write a program called `rename` that takes three or more command-line arguments:

1.  A <span g="filename_extension">filename extension</span> to match.
2.  An extension to replace it with.
3.  The names of one or more existing files.

When it runs,
`rename` renames any files with the first extension to create files with the second extension,
but will *not* overwrite an existing file.
For example,
suppose a directory contains `a.txt`, `b.txt`, and `b.bck`.
The command:

```sh
rename .txt .bck a.txt b.txt
```

<!-- continue -->
will rename `a.txt` to `a.bck`,
but will *not* rename `b.txt` because `b.bck` already exists.
