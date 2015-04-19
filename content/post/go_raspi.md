+++
categories = []
date = "2015-04-19T20:15:26+02:00"
description = ""
keywords = []
title = "Crosscompile to rasberry pi"

+++


Crosscompile to rasberry pi. 
From mac:
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew reinstall go --cross-compile-all
```

```
GOOS=linux GOARCH=arm go build
```

Now copy the binary to rasberry pi and rund it.