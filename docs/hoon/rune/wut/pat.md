---
navhome: /docs/
next: true
sort: 10
title: ?@ "wutpat"
---

# `?@ "wutpat"`

`[%wtpt p=wing q=hoon r=hoon]`: branch on whether a wing 
of the subject is an atom.

## Expands to

```
?:(?=(@ p) q r)
```

## Syntax

Regular: *3-fixed*.

## Examples

```
~zod:dojo> ?@(0 1 2)
! mint-vain
! exit
~zod:dojo> ?@(`*`0 1 2)
1
~zod:dojo> ?@(`*`[1 2] 3 4)
4
```
