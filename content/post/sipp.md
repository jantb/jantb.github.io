+++
date = "2015-04-17T20:48:00+02:00"
title = "sipp.io registrered"
+++


Registrered sipp.io today. The name was found by:

```
for domain in `cat * | perl -ne's/io$/.io/ && print lc if length == 7'` ; do whois $domain | grep "is available for purchase" ; done
```

Where cat * is the path to a dictionary.
