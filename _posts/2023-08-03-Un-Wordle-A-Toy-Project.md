---
title: Un-Wordle - A Toy Project
description:
image:
  path: /assets/posts/Un-Wordle-A-Toy-Project/wordle.jpg
  show_in_post: true
date: 2023-08-03 00:55:00 +0300
categories: []
tags: [toy-project,golang,coding]
---

## Wordle
Back in 2021 [Wordle](https://en.wikipedia.org/wiki/Wordle) was a pretty popular online trivia game. Players had six attempts to guess a five-letter word, with feedback given for each guess in the form of colored tiles indicating when letters match or occupy the correct position. 
Each day a new puzzle was available and having such a simple rule set, it rapidly gain lots of peoples attention.

Its instructions were presented like this:
![wordle instructions](/assets/posts/Un-Wordle-A-Toy-Project/instructions.png)

It was pretty common back then for people to share their results using the color code (or any other emoji styled output) that was used as the game's results:
⬜⬜⬜🟨⬜
🟨⬜🟨🟨⬜
🟩🟩🟨🟨🟩
🟩🟩🟩🟩🟩

## The solver: Unwordle

I found my self having a hard time solving this kind of puzzle so I though: Why not creating an program that solves this kind of puzzle? 😉

This is where I developed -- UNWORDLE --.

Wordle is a [Constraint satisfaction](https://en.wikipedia.org/wiki/Constraint_satisfaction_problem) problem and the way it is solved is using [Constraint propagation techniques](https://en.wikipedia.org/wiki/Local_consistency).
Other similar puzzle games are [Sudoku](https://en.wikipedia.org/wiki/Sudoku) and [Mastermind board game](https://en.wikipedia.org/wiki/Mastermind_(board_game)).

## Internals

Unrordle is a toy app that tries to solve the Wordle puzzle and its variations. 

Every solver uses a predefined dictionary that contains words of the language that wordle uses.

In order to solve a wordle puzzle as soon as unworlde starts a proposal will be given for the first word to use. The
first word (opener) is estimated based on the given dictionary. More on this process can be found at the  
[Estimating a good opener](#estimating-a-good-opener) chapter. The first proposal is like this:

```
Try #1: 		arose
```

As first try, the word `arose` is proposed to be used. The `unworlde` awaits for user input based on the results of the
puzzle. A string with the same length as the wordle's length indicating the presence of each letter should be given by
user. Letters accepted are `b`, `y` and `g`. These letters mean:

* b (black): the letter does not exist in the wordle
* y (yellow): the letter exists in the wordle but not at the given place
* g (green): the letter exists in the wordle at this specific place

```
Try #1: 	    	arose
Response (b|y|g): 	gbybb
```

The process continues until all tries are exhausted or a reply all of `g`s is given or only one possible word matches
our criteria.

```
Found Solution: 	thick
Hooray! :-)
```

or

```
No solution found.
Sorry. :-(
```

If we choose to display info for each try we get this output:

```
Try #3: 	dodge | Possibility: 1/125 | Score: 20
```

`1/125` is the possibility to success as unwordle has excluded all other words from the given dictionary. The way that
score is calculated is analyzed at the `Eliminating candidates` chapter.

To make it clear here is a demo:
![unwordle](/assets/posts/Un-Wordle-A-Toy-Project/unwordle.gif)

### Eliminating candidates

After each proposed word, unwordle requests for the user response based on the wordle result. This input contains valuable information for each one of the letters of the proposed word. So
 - for each letter that does not exist in the wordle (black), unwordle eliminates all dictionary's words that contain this letter
 - for each letter that exists on the wordle but is placed on wrong place (yellow), unwordle eliminates all dictionary's words that contain this letter at ths specific place

So, the maximum score a wordle can achieve is 5 * wordsLength. Doing this process for each input result given, unworlde manages to exclude as many words as possible from the given dictionary.


### Estimating a good opener

> Well begun is half done!

Unworlde is able to estimate a good opener word (the first proposed word) instead os using a random one from the given dictionary. The process of estimating a good opener goes as follows:
 - the occurrence of each letter is calculated based on the given dictionary
 - using the most frequent letters we search for a word that contains each one of them
  
This way the opener word will give more value even from the first try!

### Proposing a good solution

For each try, unwordle tries to propose the most promising word chosen from the words that have not been eliminated from the input dictionary. For each word within the dictionary, a score is calculated before proposing a new word. The score is calculated using only the given responses.

 - score is incremented by 1 for each letter that exists both in the word and the wordle
 - score is incremented by 5 for each letter that exists both in the word and the wordle in the exact place

## Future Improvements

Unwordle has various ways to get improved. Here are just a couple of them that will produce even more accurate results:

Provided dictionaries have some limitations. Not all words are available to some wordle variations. This can cause unwordle not to be able to find the solution. Apart from this, unwordle might propose to use some words that is not acceptable by the puzzle game leaving the user unable to move forward.

Rules used to eliminate words from dictionary and also calculate the score can be both enriched. Sophisticated rules can be applied such as when we have information about all the letters of a word (green and yellows) to exclude all other words.

## Contributing

Feel free to contribute by opening an issue or even a PR through the [Unwordle's repo](https://github.com/mzampetakis/unwordle).
