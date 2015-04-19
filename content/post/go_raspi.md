+++
categories = []
date = "2015-04-19T20:15:26+02:00"
description = ""
keywords = []
title = "Crosscompile to rasberry pi:"

+++


Crosscompile to rasberry pi:

```
brew reinstall go --cross-compile-all
```

```
GOOS=linux GOARCH=arm go build
```