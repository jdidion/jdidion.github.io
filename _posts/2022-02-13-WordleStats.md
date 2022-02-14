--- 
published: true
title:  What should my first Wordle guess be?
layout: post
author: John
category: articles
tags: 
- games
- stats

---

Like just about everyone else, I've gotten sucked into playing Wordle. I typically don't go in for games with such straight-forward mechanics, but for a couple reasons it hits a sweet-spot for me. First, it is time-boxed: you can only play one round a day and each round is relatively short, so there's no risk of it sucking up big chunks of time. Second, on its surface it is a word game (and I'm a bit of a word nerd), but underneath it is a game about probability.

Why do I say it's a game of probability? Because there's really only two strategies for choosing the first word, and they are both based on the frequency of letters in the word list:

* Select the word that (on average) maximizes the overlap with the set of letters in the answer word (i.e. the number of yellow tiles)
* Select the word that (on avearge) maximizes the number of correctly placed letters (i.e. the number of green tiles)

I tend to prefer the first strategy, but I do not have any concrete analysis to show that it is better than the second. Whichever strategy you choose, you need to decide what your first word will be, and you can narrow it down a relatively small number of choices using some simple analysis of letter frequencies. For fun, I will do this using just some unix command line tools and a bit of R.

## The word list

First we need a list of words. While Wordle does not publish their word list, we can probably get a good approximation of the letter frequencies in that list using any large list of 5-letter words. Fortunately, linux-based operating systems provide a list of words read for use: `/usr/share/dict/words`.

Since this file contains words of varying lenghts, we need to filter it to keep only the 5-letter words. Also, since it appears the list contains words with a mix of upper- and lower-case, and since Wordle is not case-sensitive, we need to normalize the letter cases. This can be accomplished with `awk` and the `tr` command:

Finally, even though this list is probably already sorted and does not contain duplicates, it's still good practice to make sure. We can do that with the `sort` and `uniq` commands.

Here is the full pipeline. I've piped the result to head so we can just see an example of what we're working with:

```
$ cat /usr/share/dict/words | awk 'length($0) == 5'| tr '[:upper:]' '[:lower:]' | sort | uniq | head
aalii
aaron
abaca
aback
abaff
abaft
abama
abase
abash
abask
```

## Strategy 1: Maximize Yellow Tiles

Maximizing yellow tiles simply means choosing the word with the letters with the highest frequency in the dictionary, ideally without repeating any letters (since a repeated letter doesn't give us any additional information). To get a frequncy table, we can use `uniq -c`, which collapses repeated lines into a single line and a count. But first we need to transform our list of words to a list of characters. For that, we can use `fold -w1` which wraps each line to a length of 1. After folding, we need to sort, then count, then sort again to get our final sorted frequency table.

```bash
cat /usr/share/dict/words | awk 'length($0) == 5'| tr '[:upper:]' '[:lower:]' | sort | uniq | fold -w1 | sort | uniq -c | sort -r
5627 a
4801 e
3460 r
3238 i
3237 o
2814 s
2719 l
2691 n
2662 t
2264 u
1792 c
1789 y
1640 d
1580 m
1550 h
1450 p
1336 b
1261 g
1074 k
 740 w
 724 f
 541 v
 306 z
 233 x
 233 j
  98 q
```

It turns out that `a,e,r,i,o` are the most frequent characters in 5-letter words in the dictionary, so our ideal first word would contain these characters. We can use grep to check if there are any such words.

```bash
$ grep -P '^[aerio]{5}$' /usr/share/dict/words
aerie
airer
arara
arear
aroar
arrie
eerie
error
reree
```

There are a few, but unfortunately they all have at least one repeated character. Now, if we look closely at the dictionary we can see that it does not contain plurals. Since `s` is the sixth most frequent character, I replaced `o` with `s` and, after removing any words with dupliated letters, end up with:

```
arise
raise
serai
```

Now, having played Wordle for a few weeks, I've have not seen any uncommon words, so my assumption is that the word list has been filtered to contain only words that most people will know. I think `serai` would be much less likely than the other two, so I would choose `arise` or `raise` as my first word to maximize my chance of getting a hit on my first guess.

## Strategy 2: Maximize Green Tiles

To maximize green tiles, we are not just interested in letter frequency, but specifically *positional* letter frequency, i.e. which letter is most likely in each of the 5 positions. Analyzing positinal letter frequency is (I think) more than we can do with linux command line tools, so instead I'll load the data into R and do my analysis there. We can still stick with the command line thanks to the `Rscript` command. I'll be using the pipeline operator (`%>%`), which requires installing the `matrittr` library.

```R
> install.packages("magrittr")
```

The first thing we have to do is read the word list from `stdin`. There are many ways to do this, but I'll use base R whenever possible to minimize the number of libraries I need to install.

```bash
cat /usr/share/dict/words | awk 'length($0) == 5'| tr '[:upper:]' '[:lower:]' | sort | uniq | Rscript -e 'scan(file("stdin"), "raw")'
[1] "aalii" "aaron" "abaca" "aback" "abaff" "abaft" ...
```

From here, we want to get to a matrix of words (rows) by letters (columns), i.e. an `N*5` matrix. For that, we can use `strsplit` followed by `unlist` to get to an array of characters, then fold them into a matrix using `matrix(ncol=5)`.


```bash
$ cat /usr/share/dict/words | awk 'length($0) == 5'| tr '[:upper:]' '[:lower:]' | sort | uniq | Rscript -e 'scan(file("stdin"), "raw") %>% strsplit("") %>% unlist %>% matrix(ncol=5, byrow=T)'
     [,1] [,2] [,3] [,4] [,5]
[1,] "a"  "a"  "l"  "i"  "i"
[2,] "a"  "a"  "r"  "o"  "n"
[3,] "a"  "b"  "a"  "c"  "a"
[4,] "a"  "b"  "a"  "c"  "k"
[5,] "a"  "b"  "a"  "f"  "f"
[6,] "a"  "b"  "a"  "f"  "t"
...
```

As a first approach to analyzing positional frequency, we can simply count the frequency of each letter in each column, using `table` (for brevity I'll just focus on the R command from now on):
```R
> scan(file("stdin"), "raw") %>% strsplit("") %>% unlist %>% matrix(ncol=5, byrow=T) %>% apply(2, table)
  [,1] [,2] [,3] [,4] [,5]
a 1773  885 1063  963  943
b  882  117  123  105  109
c  703  408  231  274  176
d  272  700  239  205  224
e  846 1203  833 1028  891
f   72  461   50   76   65
g  196  597  178  140  150
h  295  364  303  308  280
i  548  571  863  610  646
j   25   17  170   10   11
k  160  114  414  206  180
l  585  519  753  503  359
m  223  219  749  201  188
n  470  425  792  446  558
o  632  624  788  703  490
p  131  148  125  745  301
q    2    6    7   65   18
r  621  768  512  918  641
s  323  321  370 1105  695
t  371  376  367  378 1170
u  324  450  450  393  647
v   68   86   75   87  225
w   80   72   61   81  446
x   44   47   35   50   57
y  264  412  371  346  396
z   62   62   50   26  106
```

Here we see that `a` is most common in positions 1 and 3, `e` is most common at position 2, `s` is most common at position 4, and `t` is most common at position 5, but unfortunately `aeast` is not a word.

Another approach is to look at how over-represented each letter is in each position relative to the other letters, e.g. by normalizing by the median frequency:

```R
> scan(file("stdin"), "raw") %>% strsplit("") %>% unlist %>% matrix(ncol=5) %>% apply(2, table) %>% {t(t(.) / apply(., 2, median))} %>% round(2)
  [,1] [,2] [,3] [,4] [,5]
a 6.25 2.26 3.17 3.31 3.25
b 3.11 0.30 0.37 0.36 0.38
c 2.48 1.04 0.69 0.94 0.61
d 0.96 1.79 0.71 0.70 0.77
e 2.98 3.07 2.49 3.53 3.07
f 0.25 1.18 0.15 0.26 0.22
g 0.69 1.52 0.53 0.48 0.52
h 1.04 0.93 0.90 1.06 0.96
i 1.93 1.46 2.58 2.10 2.22
j 0.09 0.04 0.51 0.03 0.04
k 0.56 0.29 1.24 0.71 0.62
l 2.06 1.32 2.25 1.73 1.24
m 0.79 0.56 2.24 0.69 0.65
n 1.66 1.08 2.36 1.53 1.92
o 2.23 1.59 2.35 2.42 1.69
p 0.46 0.38 0.37 2.56 1.04
q 0.01 0.02 0.02 0.22 0.06
r 2.19 1.96 1.53 3.15 2.21
s 1.14 0.82 1.10 3.80 2.39
t 1.31 0.96 1.10 1.30 4.03
u 1.14 1.15 1.34 1.35 2.23
v 0.24 0.22 0.22 0.30 0.77
w 0.28 0.18 0.18 0.28 1.54
x 0.16 0.12 0.10 0.17 0.20
y 0.93 1.05 1.11 1.19 1.36
z 0.22 0.16 0.15 0.09 0.36
```

This tells us, for example, that `a` in position 1 is more important than it is in position 3. With this information, we can create a regular expression that we can use to `grep` the word list to give us some candidates where positional letter frequency is maximized. For example, here I'm including every letter with a score > 2.25.

```bash
$ grep -P '^[abce][ae][aeilno][aeoprs][aest]$' /usr/share/dict/words
balas
balsa
banat
beast
beest
beira
beisa
belee
benet
canoe
cease
cense
```

Of these, `beast, canoe, and cease` are fairly common, and of these `beast` has the highest score (the sum of the value of each letter in each column).

## What to do next?

Assuming this wasn't the one day in every ~30 years where your first guess was correct (assuming a word list of ~10,000), you need to decide between two strategies for choosing your second guess. Unlike the first guess, which can be the same every time, what to do for the second guess depends on how many yellow and green tiles your first guess turned up. If you think you have enough tiles, you can start trying to guess the correct word. If not, you are better off guessing a second word designed to maximize the number of yellow or green tiles, meaning you can follow the same general approach you did for choosing your first word.

For example, I use `arise` as my first word. Excluding those letters, the next most frequent letters are `o,l,n,t,u,c`

```bash
$ grep -P '^[olntuc]{5}$' /usr/share/dict/words
cloot
clout
cocco
colon
conto
count
lotto
loulu
nullo
ululu
uncut
```

Excluding words with repeated letters and less common words, `clout, count, and uncut` seem like the best bets. I use `count` for my second word when `arise` does not turn up enough tiles.

After two guesses, you will at least have one vowel. If you still don't have enough tiles to make a guess at the word, then you can use your knowledge of letter frequencies to come up with a third guess that will maximize the number of yellow or green tiles. For example, if I just have an `o` then `moldy` is a good third guess.

The question of how many tiles you need to know before trying to guess the word is an interesting one, but I do not have a good intuition for how to determine this objectively. My personal rule of thumb is that if I have at least 3 tiles I will start tailoring my guesses to uncover the positions of the yellow tiles. For example, if I guessed `route` and `r, t, and e` are yellow, then my next guess might be `tires` - new positions for each of the yellow tiles, and two new letters whith high frequency. 
