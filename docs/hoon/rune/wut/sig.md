---
navhome: /docs/
next: true
sort: 9
title: ?~ "wutsig"
---

# `?~ "wutsig"` 

`[%wtsg p=wing q=hoon r=hoon]`: branch on whether a wing 
of the subject is null.
 
## Expands to

```
?:(?=($~ p) q r)
```

## Syntax

Regular: *3-fixed*.

## Discussion

It's bad style to use `?~` to test for any zero atom.  Use it
only for a true null, `~`.

## Examples

```
~zod:dojo> =foo ""
~zod:dojo> ?~(foo 1 2)
1
```
