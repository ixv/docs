# `%ford` user manual

## ford runes

One of the most common ways to use ford is through ford's runes, all of which begin with `/` ("fas"). A ford rune runs a step of a ford build and places the result in the subject. There are various kinds of build steps, some of which take other build steps as parameters. The most common pattern is to have a sequence of ford runes at the top of a hoon source file that import the results of evaluating other hoon files. This is how we "import a library" in urbit: we add the result of compiling another hoon file into the current subject, possibly renaming it by wrapping a face around it.

Another common use case is assembling and rendering markdown templates into a single HTML file, to be sent as an HTTP response in urbit's web server `%eyre`. It's important to keep in mind that %ford is quite flexible and generic, and is used in other parts of urbit -- such as the dojo -- that would not traditionally fall under the auspices of a build system.

### `/+` import from lib/

The `/+` rune accepts a filename as an argument. It interprets that filename
as a hoon source file within the `lib` directory. This is how we import a shared
library in urbit.

Example:
```
/+  time-to-id
::
(time-to-id now)
```
produces: `"c.314d"` (or something similar depending on when you run it)

### `/-` import from sur/

The `/-` rune accepts a filename as an argument. It interprets that filename as
a hoon source file within the `sur` directory. The `sur` directory contains
shared structures that can be used by other parts of urbit. This is somewhat
similar to including a header file in C.

Example:
```
/-  talk
::
*serial:talk
```
produces: `0v0`

### `/~` twig by hand

`/~  <twig>` produces a horn that evaluates a twig and places the product
in the subject. Arbitrary hoon can be in the twig.

Example:
```
/~  [a=0 b=1]
[a b]
```
produces:
`[0 1]`

In wide-form, `/~` always takes a tuple (which may be a degenerate tuple of one element), and produces it.

Example:
```
/~[%something]
```
produces: `%something`

```
/~[%a %b]
```
produces: `[%a %b]`

### `//` include a file by relative path

`// <rel-path>` parses `rel-path` as a hoon twig, and then adds the resulting
twig to the subject. Note that the result type of this rune is not a horn, but
just a hoon twig, so its result can't be used as an argument to other runes
that expect a horn. There is no wide-form for this rune.

Example:
```
//  %/file-to-include
::
(frobnicate:file-to-include %.n)

::  contents of file-to-include.hoon
|%
++  frobnicate
  |=  a/?  ^+  a
  !a
--
```
produces: `%.y`

_Note: `%` means current directory, which in a hoon file will resolve to the
"directory" containing that file. Also, %clay doesn't have first-class
directories the way unix filesystems do. Whereas in unix, a directory is a
special kind of file, in %clay it's just a path prefix, and there is no file
stored at that path._

### `/=` wrap a face around an included horn

`/=` runs a horn (usually produced by another ford rune), takes the result of
that horn, and wraps a face around it.

Example:
```
/=  foo  /~  [a=0 b=1]
::
[a.foo b.foo]
```
produces: `[0 1]`

In wide-form, `/=` uses `=` as a delimiter: `/=foo=/~[a=0 b=1]`

### `/:` evaluate at path, and `/<mark>/` render mark at current path

`/:` takes a path and a horn, and evaluates the horn with the current path set to the supplied path.
`/mar/` renders the mark 'mar' at the current path.

Example:
```
/=  hoo-source  /:  /path/to/hoon-file  /hoon/
::
`@t`hoo-source
```

produces the text of the hoon file at "/path/to/hoon-file/hoon".

`/hoon/` renders the current path using the `%hoon` mark, which in this case
passes the contents through unchanged.  In general, rendering a file with a
mark will potentially run the contents through a series of conversion
operations. For details on marks, see the
[marks docs](https://urbit.org/docs/arvo/marks).

Here's an example that includes a mark conversion:
```
/=  page  /:  /path/to/html/file  /mime/
::
page

::  contents of /path/to/html/file:
<html><head><title>My Fascinating Blog</title></head></html>
```
produces:
`[[%text %html ~] 60 '<html><head><title>My Fascinating Blog</title></head></html>']`

This result includes the MIME type ('text/html'), the content length in bytes,
and the HTML itself.

In wide-form, `/:` uses `:` as a delimiter: `/=page=/:/path/to/html/file:/mime/`

### `/&` pass through a series of marks

`/&` passes a horn through multiple marks, right-to-left. It has both a
wide-form and a tall-form syntax. In wide-form, it takes a series of mark
arguments followed by a horn. In tall-form, it takes a single mark followed by
a horn.

```
/=  some-text  /:  /%/text-file  /&mime&/txt/
::
some-text
```
produces: `[[%text %plain ~] p=17 q='Hi I\'m some text\0a']`

This example shows two of the ways marks are used. The first way is what
happens with the `/txt/` mark: we use it to find a file in clay with that
extension, without performing any conversions.  Since this file is stored with
the `%txt` mark in `%clay`, its type is a `wain`: a list of cords, where each
cord is a single line. Once we've read the file, the `%mime` mark converts the
`wain` to a triple that includes the MIME type ("text/plain"), the content
length in bytes, and the content itself as a cord.

```
/=  page  /&html&elem&/~[;div.foo;]
::
page
```
produces: `'<html><head></head><body><div class="foo"></div></body></html>'`

This runs the sail expression `;div.foo;` through the `%elem` mark, then
through the `%html` mark.  The `/~` rune produces an item of mark `noun`.  The
`%elem` mark converts the mark of the expression from `noun` to `elem` by
checking that the type fits in `manx` (a hoon/sail type indicating an XML
element). The `%html` mark recognizes the `%elem` and converts it to an HTML
string with enclosing `<html>`, `<head>`, and `<body>` tags.

It's possible to use wide-form `/&` with more than two marks, by using `&` as a
delimiter between marks and adding a `/` before the last mark, like:
`/&d&c&b&/a/`. The last argument here can actually be any arbitrary horn, not
just a mark: `/&c&b&/:path:/mark/`

Tall-form `/&` takes only two arguments: a mark and a horn. The mark does not
need to be enclosed in `/`'s: `/&  html  /elem/`

### `/_` run a horn on each file in the current directory

`/_` takes a horn as an argument. It produces a new horn representing the
result of mapping the supplied horn over the list of files in the current
directory. The keys in the resulting map are the basenames of the files in the
directory, and each value is the result of running that horn on the contents of
the file.

Example:
```
/=  kids  /_  /hoon/
::
`(map term cord)`kids
```

produces a value of type `(map term cord)`, where each key is the basename (the
filename without the prefix), and each value is the result of running the
contents of the file through the `%hoon` mark, which validates that it's valid
hoon code and returns it unmodified. So, the resulting map associates basenames
with file contents.

### `/;` operate on

`/;` takes a twig and a horn. The twig should evaluate to a gate, which is then slammed
with the result of the horn as its sample.

Example:
```
/=  goo
    /;  |=({a/@ b/@} +(b))
    /~  [a=0 b=1]
::
goo
```

produces: `2`

Here's a slightly more complex example with runes that use the filesystem:
```
/=  file-length
    /;  |=(a/@t (lent (crip a)))
    /:  /%/path/to/hoon/file  /hoon/
::
file-length
```
produces the number of bytes in the file "/%/path/to/hoon/file."

### `/,` switch by path

`/,` is a switch statement, which picks a branch to evaluate based on
whether the current path matches the path in the switch statement. 
Takes a sequence of pairs of (path, horn) terminated by a `==`.

Example:
```
/=  just-right
    /:  /===/right-path                                 ::  set path to /%/right-path
    /,
      /wrong-path  /~  ~
      /another-wrong-path  /~  ~
      /right-path  /~  %evaluate-me                     ::  only evaluate this horn
    ==
::
`@t`just-right
```

produces: `'evaluate-me'`

### `/.` list

Produce a null-terminated list from a sequence of horns, terminated by a `==`.

Example:
```
/=  vanes
    /.
      /~  %ames
      /~  %behn
      /~  %clay
      /~  %dill
      /~  %eyre
      /~  %ford
      /~  %gall
    ==
::
vanes
```
produces: `[%ames %behn %clay %dill %eyre %ford %gall ~]`.

### `/*` heterogeneous map

_DEPRECATED_
TODO: remove

### `/^` cast

`/^` takes a mold and a horn, and casts the result of the horn to the mold.

Example:
```
/=  liz  /^  (list @)                                   ::  cast to real list
         /~  ~[1 2 3]                                   ::  no 'i' or 't' faces
::
?<  ?=($~ liz)                                          ::  prevent find-fork
i.liz
```

produces: `1`

Without the cast, we wouldn't be able to access the 'i' face of the list.

### `/#` insert dephash

`/#` takes a horn and produces a cell of the dependency
hash of the result of the horn, and the result itself.

Example:
```
/=  inline  /^  {dep/@uvH txt/@t}  /#  /:  /%/my-script  /js/
::
;=  ;script: (trip txt.inline)                          ::  set script contents
    ;script@"/~/on/{<dep.inline>}.js";                  ::  set script src
==
```
produces two `<script>` tags. The first has its contents set to the contents
of the JavaScript file we loaded from clay. The second is an auto-update
script that polls the server to check whether the dependency hash has changed.
This pattern is used in urbit's `tree` web publishing system.


### `/$` process extra arguments

`/$` will slam a gate on whatever extra arguments have been supplied to this build.
At the moment, only HTTP requests forwarded by `%eyre` contain any extra arguments,
and using this rune outside of that context will cause an error. Requests from `%eyre`
contain an argument representing the query string, which can be parsed using the
standard library gate `++fuel`.

Example:
```
/=  gas  /$  fuel
::
:*
  %who    {<(~(get ju aut.ced.gas) 0)>}
  %where  {(spud s.bem.gas)}
  %case   {(scow %ud p.r.bem.gas)}
==
```
produces: [%who '~zod' %where /ford/pages/web %case 60]
