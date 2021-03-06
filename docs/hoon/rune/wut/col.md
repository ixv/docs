---
navhome: /docs/
next: true
sort: 1
title: ?: "wutcol"
---

# `?: "wutcol"`

`[%wtcl p=hoon q=hoon r=hoon]`: branch on a boolean test.

## Produces

If `p` produces yes, then `q`. If `p` produces no, `r`.
If `p` is not a boolean, compiler yells at you.

The subjects of `q` and `r` are constrained to match any
pattern-matching algebra in `p`.  The analysis, which is
conservative, understands any combination of [`?=` ("wuttis")](../tis/), 
[`?&` ("wutpam")](../pam/), [`?|` ("wutbar")](../bar/), and 
[`?!` ("wutzap")](../zap/), and reduces the subject appropriately when 
compiling.

If test analysis reveals that either branch is not taken,
or if `p` is not a boolean, compilation fails.  However,
an untaken branch can be indicated with the `%lost` hoon.

## Syntax

Regular: *3-fixed*.

## Discussion

Short-circuiting in boolean tests works as you'd expect
and includes the expected inference.  For instance,
if you write `?&(a b)`, `b` is only executed if `a` is
positive, and compiled with that assumption.

Note also that all other branching hoons reduce to `?:`.

## Examples

```
~zod:dojo> ?:((gth 1 0) 3 4)
3

~zod:dojo> ?:  (gth 1 0) 
             3 
           4
3

~zod:dojo> ?:((gth 1 2) 3 4)
4

~zod:dojo> ?:  (gth 1 2) 
             3 
           4
4
```
