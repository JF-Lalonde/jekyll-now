---
layout: post
title: Using Zgrep
published: true
date: '2021-02-01 17:48'

---

I just discovered the other day that you can search zipped files without needing
to uncompress them by using the `zgrep` command.

I'm sure plenty of you know about it, or don't care that you don't know about
it, but I was really excited when I discovered its existence. If I'm grepping
through logs that are older than a certain limit, our provider archives these
and I have to download a gzip file.

My first time coming across one I naively tried to unzip it and open it with
Apple Numbers. This file was a couple of GBs and Numbers kept crashing trying to
open it.

```shell
> open recipe_for_a_bad_time.tsv
```

I finally realized I could grep the file and redirect the output to a separate
file. This however still took considerable time, but at least afterwards I had a
smaller file I could open to view the results.

Better:
```shell
> grep 'good time' recipe_for_a_bad_time.tsv > temp.tsv
```

I then remembered that I use the silver searcher for my fuzzy-finding in my text
editor, so I leveraged that instead of `grep`. Interestingly it didn't perform
any faster that grep for my use-case. I decided to do a little benchmarking to
see what the difference looked like.

```shell
# Benchmarking 'grep'
> time grep 'good time' recipe_for_a_bad_time.tsv > 'temp.tsv'
> 2.09s user 0.08s system 99% cpu 2.316 total

# Benchmarking 'ag'
> time ag 'good time' recipe_for_a_bad_time.tsv > 'temp.tsv'
> 3.07s user 0.20s system 98% cpu 3.308 total
```

So **grep** actually was _faster_ than **ag** by abount 1 second. I re-ran the
tests a dozen times each and tried different keywords and consistently got the
same results.
I was surprised since **ag** is touted as a faster solution to search.
Of course this is one very specific case and different files might produce
different results.


Then, the other day, I had to search through archived logs again and decided I'd
look to see if it was possible to search without even needing to uncompress the
gzipped file. Enter **zgrep**!

The syntax is the same as **grep** but you don't need to unzip your file and I
found the search to be as fast as grep.

```shell
> time zgrep 'good time' recipe_for_a_bad_time.tsv.gz > 'temp.tsv'
> 2.13s user 0.05s system 99% cpu 2.190 total
```

Incidentally, I tried using the silver searcher with the `-z` flag to see if
it'd be even faster, but it returned no matches almost immediately. I'll have to
dig into that to see what I was doing wrong.

So there it is, next time you need to pore over gzipped files, reach for
**zgrep**!

Feel free to leave a comment if you have any solutions that are more performant
or that you like better for any reason!
