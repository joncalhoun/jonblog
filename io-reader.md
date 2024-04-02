+++
title = "Go's io.Reader"
description = "In this article we explore the io.Reader interface provided by Go's io package."
date = 2024-03-06

[author]
name = "Jon Calhoun"
email = "jon@calhoun.io"
+++

The `io.Reader` is defined as:

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```
