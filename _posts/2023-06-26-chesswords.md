---
layout: post
title: Chesswords - a chess wordsearch generator
date: 2023-06-26 10:12:00 +0100
categories: blog
---

A couple of months back, I was doing a wordsearch with my 6-year-old when she came up with a brilliant idea. She was getting disheartened with one of the clues so we started talking about if there was a technique to guarantee you would find the clue, no matter what.

I showed her what I do:

1. Go line by line and find the letter at the start of the clue
2. Look at all the the surrounding squares to see if the second letter in the clue is there
3. If not, go back to 1. Otherwise keep following that direction to see if the rest of the letters match

This is probably what most people do and it's not all that interesting (but is effective!).

To explain it to her, I drew her a picture (something like the image below) and immediately she came out with __"Just like the king in chess!"__.

![A picture of a wordsearch showing step 2 - pick a letter and check its 8 surround letters](/images/wordsearch.png)

### Chess Wordsearches

This got us thinking - could you make a wordsearch where the clues are hidden in the shape of chess moves? Soon we had forgotten the original wordsearch and had made a couple of _chess wordsearches_ on paper.

These turned out to be really fun - especially the _Knight_ clues. All the other pieces' patterns are already possible in a standard wordsearch but the knight introduces jumps which makes them far more interesting!

Here is an example of a Knight clue for the word _Forby_[^1]:

![An example of a Knight Clue or the word ](/images/knight-clue.png)

### Chesswords - a chess wordsearch generator

Over a couple of evenings I made [Chesswords](https://chesswords.fly.dev/) - a simple website that generates chess wordsearches on the fly.

It has a devout user base of one and is built very simply:

* Python + Flask to generate the chess word searches and render the html
* Some vanilla javascript to add interaction
* Unicode chess pieces [https://en.wikipedia.org/wiki/Chess_symbols_in_Unicode]()
* [fly.io](https://fly.io/) for hosting
* Pen and paper for _Chester_: ![Chester](/images/chester.jpeg)

# Footnotes

[^1]: I very gratefully used an open source word list to generate the clues for Chesswords: [https://github.com/dwyl/english-words](https://github.com/dwyl/english-words). There are some very strange words in this list!
