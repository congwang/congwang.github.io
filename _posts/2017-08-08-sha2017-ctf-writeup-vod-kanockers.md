---
layout: post
title: "SHA2017 CTF writeup — Vod Kanockers"
date: 2017-08-08 01:12:38 +0000
categories: blog
---

### SHA2017 CTF writeup — Vod Kanockers

> The name is Kanockers. Vod Kanockers.

Click the link and view the source of that page. Find out the hint:

```
<!-- *Knock Knock* 88 156 983 1287 8743 5622 9123 -->
```

This reminds me of https://wiki.archlinux.org/index.php/Port_knocking. This means we have to “knock” these ports in a predefined order and then some port, either in this list or not in this list, will be open to us.

Let’s find out which of them are not fully closed. Try curl to each of them, only the last one does not return a RST.

```
curl
```

Then try to knock them with the command knock:

```
knock
```

```
knock vod.stillhackinganyway.nl 88 156 983 1287 8743 5622 9123 -v
```

and try curl on 9123 again, still no response, our SYN packets are still dropped there. So either the “target” port is not 9123, or the above order is not correct.

```
curl
```

Use nmap to scan that host to see if there is any other port not in this list is open. Nope!

```
nmap
```

Now, we have to figure out the order! It is quite easy to brute force these 7! sequences, with the following Python code:

```
7!
```

```
import itertools
```

```
for l in itertools.permutations([88, 156, 983, 1287, 8743, 5622, 9123]):
```

```
print " ".join(map(str, l))
```

Now pipe each line of the output to knock :

```
knock
```

```
python permute_ports.py | while read i
```

```
do
```

```
echo $i
```

```
knock vod.stillhackinganyway.nl $i
```

```
curl --connect-timeout 1 vod.stillhackinganyway.nl:9123
```

```
done
```

The flag will be shown.

Happy hacking!