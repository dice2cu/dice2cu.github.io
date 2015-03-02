A board game called [Alchemists](http://czechgames.com/en/alchemists/) came out in the US in early 2015.  One of the most interesting things about this game is that it depends on a [companion application][1] to store some secret information as four letter codes.  The game doesn't give us an algorithm for interpreting these codes, so that leaves it up to us to reverse engineer one.

## Summary

Before I go into way too much detail on this, let me just summarize the key findings.

The first letter of the code tells you the alchemical for the Fern ingredient:

| 1st Letter | Alchemical |
| ---------- | ---------- |
| A I Q Y    | ![R+][2]   |
| B J R Z    | ![G+][3]   |
| C K S      | ![B+][4]   |
| D L T      | ![R-][5]   |
| E M U      | ![G-][6]   |
| F N V      | ![B-][7]   |
| G O W      | ![++][8]   |
| H P X      | ![--][9]   |

The Feather ingredient has about a 35% chance of being either the all plus or all minus alchemical, a 40% chance of being a normal negative alchemical, and less than a 25% of being a normal positive alchemical.

There, done.  That's the important stuff.  You can stop reading unless you want to know why.


## Background
The board game has eight ingredients and eight alchemicals.  Your goal in the deduction part of the game is to figure out which alchemical corresponds to each ingredient.  You can get information to help you do that by combining two ingredients into potions and testing them.  See the [official explanation](http://alchemists.cge.as/) for more details.

The mapping between ingredients and alchemicals is different for every game.  So at the start of the game, you use the companion app to generate a random four letter code.  The companion app can compute a mapping from this code, and you can run your tests through the app to get information that helps you deduce the mapping.

The explanation of the code algorithm in the manual is not particularly helpful:  

> If the battery on your smartphone dies and you want to use the gamemaster triangle to finish your game, just convert the four-letter game code to an ordered list of 8 ingredients using this simple algorithm: ... Actually, on second thought, maybe we should keep that information to ourselves. 

But it does tell us that the algorithm is simple, which is a challenge if I've ever heard one.  So let's see if we can figure it out!


## Naming Conventions

The game instructions mostly use pictures to refer to the ingredients and alchemicals.  I'm way too lazy to do that, so we'll need to agree on some names for them.  And for the purposes of our algorithm, we'll also need to specify a default order for them.

### Ingredients

| ID | Name      |
| -- | --------- |
|  0 | Fern      |
|  1 | Claw      |
|  2 | Mushroom  |
|  3 | Flower    |
|  4 | Root      |
|  5 | Scorpion  |
|  6 | Toad      |
|  7 | Feather   |

The manual gives a name for every ingredient at some point except the flower.  For our purposes, I'm using the shortest name for each one ("Claw" instead of "Bird Claw", "Root" instead of "Mandrake Root", "Feather" instead of "Raven Feather").

The order of the ingredients is based on the order they appear in the [official companion app][1].  The filenames of the images in the app are "i0.png" through "i7.png", so that's the order we have them in.  Easy enough, right?



### Alchemicals

| ID | Image               | Code |
| -- | ------------------- | ---- |
|  0 | ![s_red_plus][2]    | `R+` |
|  1 | ![s_green_plus][3]  | `G+` |
|  2 | ![s_blue_plus][4]   | `B+` |
|  3 | ![s_red_minus][5]   | `R-` |
|  4 | ![s_green_minus][6] | `G-` |
|  5 | ![s_blue_minus][7]  | `B-` |
|  6 | ![u_plus][8]        | `++` |
|  7 | ![u_minus][9]       | `--` |

Unlike the ingredients, I haven't found any officials names for the alchemicals.  But the image filenames in the [companion app][1] come to our rescue again, giving us some hints about how we can name them:

* Six alchemicals have exactly one large aspect, and are named after it: s_red_plus, s_green_minus, etc.
* Two alchemicals have three large aspects of the same sign: u_plus and u_minus

The order of the alchemicals is from the companion app's result for code `AAAA`. That's the first possible code, so it's the best ordering to use for the purposes of our algorithm.


### Solutions

We'll also need a concise way to describe the solution for a code.  For that, I'm going to use an ordered list of the alchemicals.

Here's an example representation for the code `DEMO`:

`DEMO` = `R- R+ G+ -- B- B+ G- ++`

The first listed alchemical goes to our first ingredient (Fern), the second alchemical goes to our second ingredient (Claw), etc.  

This means that for the code `DEMO`, the mapping is:

* Fern = `R-`
* Claw = `R+`
* Mushroom = `G+` 
* Flower = `--`
* Root = `B-`
* Scorpion = `B+` 
* Toad = `G-`
* Feather = `++`


## Algorithm

With all of that garbage out of the way, we can finally get to the good part.  The basic steps in our version of the algorithm are:

1. Convert the four letters of the code into numbers: A = 0, B = 1, C = 2, etc.
2. Use a custom algorithm to translate those numbers into an eight digit [factoradic](http://en.wikipedia.org/wiki/Factorial_number_system) number
3. Convert the factoradic number to a permutation using the [Lehmer Code](http://en.wikipedia.org/wiki/Lehmer_code) encoding

That permutation will be of the numbers 0-7, which we can map back to our alchemicals by ID (from our [alchemical naming conventions](#alchemicals)).


Here's some pseudocode for the algorithm:

```javascript
function codeToPermutation(code) {
    var c1 = charToInt(code[0])
    var c2 = charToInt(code[1])
    var c3 = charToInt(code[2])
    var c4 = charToInt(code[3])

    var f1 = c1 % 8
    var f2 = c2 % 7
    var f3 = c3 % 6
    var f4 = c4 % 5
    var f5 = floor(c3 / 6) % 4
    var f6 = floor(c2 / 7) % 3
    var f7 = floor(c1 / 8) % 2
    var f8 = 0

    var factoradic = [f1, f2, f3, f4, f5, f6, f7, f8]
    return lehmerDecode(factoradic)
}
```

Note: `%` is the [modulo operator](http://en.wikipedia.org/wiki/Modulo_operation) and `floor` is the [floor function](http://en.wikipedia.org/wiki/Floor_and_ceiling_functions)

And I guess we can make this pseudocode into bad javascript if we add these functions:

```javascript
function lehmerDecode(factoradic) {
    var remaining = [0, 1, 2, 3, 4, 5, 6, 7]
    var permutation = []
    for (var i in [0, 1, 2, 3, 4, 5, 6, 7]) {
        var factoradicElement = factoradic[i]
        permutation[i] = remaining[factoradicElement]
        remaining.splice(factoradicElement, 1)
    }
    return permutation
}

function charToInt(c) {
    var i = c.toUpperCase().charCodeAt(0) - 'A'.charCodeAt(0)
    if (i < 0 || i >= 26) {
        i = 0
    }
    return i
}

function floor(i) {
    return Math.floor(i)
}
```

Here are the results from a few example codes:

* `DEMO` 
  * => `(3 4 0 4 2 0 0 0)`<sub>!</sub> 
  * => `(3 5 0 7 4 1 2 6)` 
  * => `R- B- R+ -- G- G+ B+ ++`
* `AAAA` 
  * => `(0 0 0 0 0 0 0 0)`<sub>!</sub> 
  * => `(0 1 2 3 4 5 6 7)` 
  * => `R+ G+ B+ R- G- B- ++ --`
* `ZZZZ` 
  * => `(1 4 1 0 0 0 1 0)`<sub>!</sub> 
  * => `(1 5 2 0 3 4 7 6)` 
  * => `G+ B- B+ R+ R- G- -- ++`
* `PUXE` 
  * => `(7 6 5 4 3 2 1 0)`<sub>!</sub> 
  * => `(7 6 5 4 3 2 1 0)` 
  * => `-- ++ B- G- R- B+ G+ R+`


## Ramifications

So at first glace, the algorithm seems reasonable.  It obscures the solution, and it's complicated enough that I can't run the whole thing in my head.  But on closer inspection, there are a couple flaws.

### The Fern

The algorithm works by building a [factoradic](http://en.wikipedia.org/wiki/Factorial_number_system) number.  That's great because when we convert that to a permutation, each digit in the permutation depends on the digits that came before it.  So even though there is only one code letter that determines each digit in the factoradic number, when we convert that to the permutation we add in some dependencies on other letters.  With one exception...

The first ingredient is the Fern, so the only code letter that affects its alchemical mapping is the first one.  That means if you know the first letter of the code, you can calculate the alchemical for the Fern:

| A I Q Y | `R+` |
| B J R Z | `G+` |
| C K S   | `B+` |
| D L T   | `R-` |
| E M U   | `G-` |
| F N V   | `B-` |
| G O W   | `++` |
| H P X   | `--` |

This is probably the most serious issue with the algorithm.  That's a small enough table to memorize, or a simple enough calculation to perform in your head.  So anyone could figure out what the alchemical mapping for the Fern is just by seeing the code (which is prominently displayed in the companion app).


### Permutations

There are only 456,976 possible codes (26<sup>4</sup>), so we can pretty easily generate the permutations for all of them: [alchemists_codes.7z](/public/files/alchemists_codes.7z).

Here's a summary of the results:

![Probability Chart](/public/images/alchemists_probchart.png)

|      | Fern  | Claw  | Mushroom | Flower | Root  | Scorpion | Toad  | Feather |
| `R+` | 15.4% | 13.0% | 13.8%    | 13.3%  | 12.7% | 14.4%    | 11.0% |  6.3%   |
| `G+` | 15.4% | 13.0% | 13.8%    | 12.2%  | 13.0% | 13.4%    | 11.5% |  7.7%   |
| `B+` | 11.5% | 13.6% | 13.2%    | 12.3%  | 13.4% | 13.5%    | 12.4% | 10.1%   |
| `R-` | 11.5% | 13.6% | 12.0%    | 12.2%  | 12.8% | 13.1%    | 13.1% | 11.8%   |
| `G-` | 11.5% | 13.6% | 11.5%    | 12.2%  | 12.1% | 12.5%    | 13.3% | 13.3%   |
| `B-` | 11.5% | 12.7% | 11.7%    | 12.3%  | 11.8% | 11.6%    | 13.4% | 15.0%   |
| `++` | 11.5% | 10.2% | 12.0%    | 12.7%  | 12.2% | 10.8%    | 13.4% | 17.2%   |
| `--` | 11.5% | 10.2% | 12.0%    | 12.7%  | 12.2% | 10.8%    | 12.0% | 18.6%   |


What does this tell us?  If you take a random code, then there is some bias in which alchemical will be mapped to an ingredient.  For most ingredients, this is only a small bias.  For example, the Flower and Root are actually pretty close to the ideal of 12.5% for each alchemical.

But the Feather has a weird distribution.  It has low odds of being either `R+` or `G+`, and it has high odds of being `++` or `--`.  This probably has something to do with it being the last ingredient?  It just gets whatever is left after all of the other ingredients have gotten their assignments.

This is only really a problem if you select a code at random.  I don't know for sure how the companion app chooses the code for a new game, but I have to assume that it does randomly generate a code since that would be the easiest method with this algorithm.

The app could avoid any bias by randomly selecting the permutation first, and then computing a code from that.  Each permutation will have more than one possible code, so it would probably be best if it then randomly selected from all possible codes for the permutation.

I doubt it does all of that now, though.  So it's likely are the above odds are what you can expect to see if you use the companion app to start a new game. 



[1]: http://gserver.czechgames.com/alchemy/
[2]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZlcnNpb249IjEuMSIgdmlld0JveD0iMCAwIDUwIDQ1IiBoZWlnaHQ9IjQ1IiB3aWR0aD0iNTAiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKC0xLjkwNzM0ODZlLTYsLTEwMDcuMzYyMikiPjxjaXJjbGUgcj0iMTUiIGN5PSIxMDIyLjQiIGN4PSIyNSIgZmlsbD0iI2YwMCIvPjxjaXJjbGUgcj0iMTAiIGN5PSIxMDQyLjQiIGN4PSI0MCIgZmlsbD0iIzAwZiIvPjxjaXJjbGUgcj0iMTAiIGN5PSIxMDQyLjQiIGN4PSIxMCIgZmlsbD0iIzAwODAwMCIvPjxyZWN0IHk9IjEwMTIuNCIgeD0iMjMiIGhlaWdodD0iMjAiIHdpZHRoPSI0IiBmaWxsPSIjZmZmIi8+PHJlY3QgeT0iMTAyMC40IiB4PSIxNSIgaGVpZ2h0PSI0IiB3aWR0aD0iMjAiIGZpbGw9IiNmZmYiLz48cmVjdCB5PSIxMDM1LjQiIHg9IjguNSIgaGVpZ2h0PSIxNCIgd2lkdGg9IjMiIGZpbGw9IiNmZmYiLz48cmVjdCB5PSIxMDQwLjkiIHg9IjMiIGhlaWdodD0iMyIgd2lkdGg9IjE0IiBmaWxsPSIjZmZmIi8+PHJlY3QgeT0iMTA0MC45IiB4PSIzMyIgaGVpZ2h0PSIzIiB3aWR0aD0iMTQiIGZpbGw9IiNmZmYiLz48L2c+PC9zdmc+ "s_red_plus"
[3]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI1MCIgaGVpZ2h0PSI0NyIgdmlld0JveD0iMCAwIDUwIDQ3IiB2ZXJzaW9uPSIxLjEiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKDAsLTEwMDUuMzYyMikiPjxjaXJjbGUgY3g9IjE1IiBjeT0iMTAzNy40IiByPSIxNSIgZmlsbD0iIzAwODAwMCIvPjxyZWN0IHdpZHRoPSI0IiBoZWlnaHQ9IjIwIiB4PSIxMyIgeT0iMTAyNy40IiBmaWxsPSIjZmZmIi8+PHJlY3Qgd2lkdGg9IjIwIiBoZWlnaHQ9IjQiIHg9IjUiIHk9IjEwMzUuNCIgZmlsbD0iI2ZmZiIvPjxjaXJjbGUgY3g9IjI3IiBjeT0iMTAxNS40IiByPSIxMCIgZmlsbD0iI2YwMCIvPjxyZWN0IHdpZHRoPSIxNCIgaGVpZ2h0PSIzIiB4PSIyMCIgeT0iMTAxMy45IiBmaWxsPSIjZmZmIi8+PGcgdHJhbnNmb3JtPSJ0cmFuc2xhdGUoMCwtNC45OTk5Nzc5KSI+PGNpcmNsZSBjeD0iNDAiIGN5PSIxMDQyLjQiIHI9IjEwIiBmaWxsPSIjMDBmIi8+PHJlY3Qgd2lkdGg9IjMiIGhlaWdodD0iMTQiIHg9IjM4LjUiIHk9IjEwMzUuNCIgZmlsbD0iI2ZmZiIvPjxyZWN0IHdpZHRoPSIxNCIgaGVpZ2h0PSIzIiB4PSIzMyIgeT0iMTA0MC45IiBmaWxsPSIjZmZmIi8+PC9nPjwvZz48L3N2Zz4= "s_green_plus"
[4]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI1MCIgaGVpZ2h0PSI0OCIgdmlld0JveD0iMCAwIDUwIDQ4IiB2ZXJzaW9uPSIxLjEiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKC0xLjkwNzM0ODZlLTYsLTEwMDQuMzYyMikiPjxjaXJjbGUgY3g9IjM1IiBjeT0iMTAzNy40IiByPSIxNSIgZmlsbD0iIzAwZiIvPjxyZWN0IHdpZHRoPSI0IiBoZWlnaHQ9IjIwIiB4PSIzMyIgeT0iMTAyNy40IiBmaWxsPSIjZmZmIi8+PHJlY3Qgd2lkdGg9IjIwIiBoZWlnaHQ9IjQiIHg9IjI1IiB5PSIxMDM1LjQiIGZpbGw9IiNmZmYiLz48Y2lyY2xlIGN4PSIxMCIgY3k9IjEwMzYuNCIgcj0iMTAiIGZpbGw9IiMwMDgwMDAiLz48cmVjdCB3aWR0aD0iMTQiIGhlaWdodD0iMyIgeD0iMyIgeT0iMTAzNC45IiBmaWxsPSIjZmZmIi8+PGNpcmNsZSBjeD0iMjUiIGN5PSIxMDE0LjQiIHI9IjEwIiBmaWxsPSIjZjAwIi8+PHJlY3Qgd2lkdGg9IjMiIGhlaWdodD0iMTQiIHg9IjIzLjUiIHk9IjEwMDcuNSIgZmlsbD0iI2ZmZiIvPjxyZWN0IHdpZHRoPSIxNCIgaGVpZ2h0PSIzIiB4PSIxOCIgeT0iMTAxMyIgZmlsbD0iI2ZmZiIvPjwvZz48L3N2Zz4= "s_blue_plus"
[5]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI1MCIgaGVpZ2h0PSI0NSIgdmlld0JveD0iMCAwIDUwIDQ1IiB2ZXJzaW9uPSIxLjEiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKC0xLjkwNzM0ODZlLTYsLTEwMDcuMzYyMikiPjxjaXJjbGUgY3g9IjI1IiBjeT0iMTAyMi40IiByPSIxNSIgZmlsbD0iI2YwMCIvPjxjaXJjbGUgY3g9IjQwIiBjeT0iMTA0Mi40IiByPSIxMCIgZmlsbD0iIzAwZiIvPjxjaXJjbGUgY3g9IjEwIiBjeT0iMTA0Mi40IiByPSIxMCIgZmlsbD0iIzAwODAwMCIvPjxyZWN0IHdpZHRoPSIyMCIgaGVpZ2h0PSI0IiB4PSIxNSIgeT0iMTAyMC40IiBmaWxsPSIjZmZmIi8+PHJlY3Qgd2lkdGg9IjE0IiBoZWlnaHQ9IjMiIHg9IjMiIHk9IjEwNDAuOSIgZmlsbD0iI2ZmZiIvPjxyZWN0IHdpZHRoPSIzIiBoZWlnaHQ9IjE0IiB4PSIzOC41IiB5PSIxMDM1LjQiIGZpbGw9IiNmZmYiLz48cmVjdCB3aWR0aD0iMTQiIGhlaWdodD0iMyIgeD0iMzMiIHk9IjEwNDAuOSIgZmlsbD0iI2ZmZiIvPjwvZz48L3N2Zz4= "s_red_minus"
[6]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZlcnNpb249IjEuMSIgdmlld0JveD0iMCAwIDUwIDQ3IiBoZWlnaHQ9IjQ3IiB3aWR0aD0iNTAiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKC0xLjkwNzM0ODZlLTYsLTEwMDUuMzYyMikiPjxjaXJjbGUgcj0iMTUiIGN5PSIxMDM3LjQiIGN4PSIxNSIgZmlsbD0iIzAwODAwMCIvPjxyZWN0IHk9IjEwMzUuNCIgeD0iNSIgaGVpZ2h0PSI0IiB3aWR0aD0iMjAiIGZpbGw9IiNmZmYiLz48Y2lyY2xlIHI9IjEwIiBjeT0iMTAxNS40IiBjeD0iMjciIGZpbGw9IiNmMDAiLz48cmVjdCB5PSIxMDA4LjQiIHg9IjI1LjUiIGhlaWdodD0iMTQiIHdpZHRoPSIzIiBmaWxsPSIjZmZmIi8+PHJlY3QgeT0iMTAxMy45IiB4PSIyMCIgaGVpZ2h0PSIzIiB3aWR0aD0iMTQiIGZpbGw9IiNmZmYiLz48Y2lyY2xlIHI9IjEwIiBjeT0iMTAzNy40IiBjeD0iNDAiIGZpbGw9IiMwMGYiLz48cmVjdCB5PSIxMDM1LjkiIHg9IjMzIiBoZWlnaHQ9IjMiIHdpZHRoPSIxNCIgZmlsbD0iI2ZmZiIvPjwvZz48L3N2Zz4= "s_green_minus"
[7]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI1MCIgaGVpZ2h0PSI0OCIgdmlld0JveD0iMCAwIDUwIDQ4IiB2ZXJzaW9uPSIxLjEiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKC0xLjkwNzM0ODZlLTYsLTEwMDQuMzYyMikiPjxjaXJjbGUgY3g9IjM1IiBjeT0iMTAzNy40IiByPSIxNSIgZmlsbD0iIzAwZiIvPjxyZWN0IHdpZHRoPSIyMCIgaGVpZ2h0PSI0IiB4PSIyNSIgeT0iMTAzNS40IiBmaWxsPSIjZmZmIi8+PGNpcmNsZSBjeD0iMTAiIGN5PSIxMDM2LjQiIHI9IjEwIiBmaWxsPSIjMDA4MDAwIi8+PHJlY3Qgd2lkdGg9IjMiIGhlaWdodD0iMTQiIHg9IjguNSIgeT0iMTAyOS40IiBmaWxsPSIjZmZmIi8+PHJlY3Qgd2lkdGg9IjE0IiBoZWlnaHQ9IjMiIHg9IjMiIHk9IjEwMzQuOSIgZmlsbD0iI2ZmZiIvPjxjaXJjbGUgY3g9IjI1IiBjeT0iMTAxNC40IiByPSIxMCIgZmlsbD0iI2YwMCIvPjxyZWN0IHdpZHRoPSIxNCIgaGVpZ2h0PSIzIiB4PSIxOCIgeT0iMTAxMyIgZmlsbD0iI2ZmZiIvPjwvZz48L3N2Zz4= "s_blue_minus"
[8]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZlcnNpb249IjEuMSIgdmlld0JveD0iMCAwIDYwIDU1IiBoZWlnaHQ9IjU1IiB3aWR0aD0iNjAiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKDAsLTk5Ny4zNjIyKSI+PHJlY3QgeT0iMTAzNS40IiB4PSI4LjUiIGhlaWdodD0iMTQiIHdpZHRoPSIzIiBmaWxsPSIjZmZmIi8+PHJlY3QgeT0iMTA0MC45IiB4PSIzIiBoZWlnaHQ9IjMiIHdpZHRoPSIxNCIgZmlsbD0iI2ZmZiIvPjxyZWN0IHk9IjEwMzUuNCIgeD0iMzguNSIgaGVpZ2h0PSIxNCIgd2lkdGg9IjMiIGZpbGw9IiNmZmYiLz48cmVjdCB5PSIxMDQwLjkiIHg9IjMzIiBoZWlnaHQ9IjMiIHdpZHRoPSIxNCIgZmlsbD0iI2ZmZiIvPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKC0xMCwxNS4wMDAwMjIpIj48Y2lyY2xlIGN4PSIyNSIgY3k9IjEwMjIuNCIgcj0iMTUiIGZpbGw9IiMwMDgwMDAiLz48cmVjdCB3aWR0aD0iNCIgaGVpZ2h0PSIyMCIgeD0iMjMiIHk9IjEwMTIuNCIgZmlsbD0iI2ZmZiIvPjxyZWN0IHdpZHRoPSIyMCIgaGVpZ2h0PSI0IiB4PSIxNSIgeT0iMTAyMC40IiBmaWxsPSIjZmZmIi8+PC9nPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKDIwLDE1LjAwMDAyMikiPjxjaXJjbGUgY3g9IjI1IiBjeT0iMTAyMi40IiByPSIxNSIgZmlsbD0iIzAwZiIvPjxyZWN0IHdpZHRoPSI0IiBoZWlnaHQ9IjIwIiB4PSIyMyIgeT0iMTAxMi40IiBmaWxsPSIjZmZmIi8+PHJlY3Qgd2lkdGg9IjIwIiBoZWlnaHQ9IjQiIHg9IjE1IiB5PSIxMDIwLjQiIGZpbGw9IiNmZmYiLz48L2c+PGcgdHJhbnNmb3JtPSJ0cmFuc2xhdGUoNSwtOS45OTk5Nzc5KSI+PGNpcmNsZSBjeD0iMjUiIGN5PSIxMDIyLjQiIHI9IjE1IiBmaWxsPSIjZjAwIi8+PHJlY3Qgd2lkdGg9IjQiIGhlaWdodD0iMjAiIHg9IjIzIiB5PSIxMDEyLjQiIGZpbGw9IiNmZmYiLz48cmVjdCB3aWR0aD0iMjAiIGhlaWdodD0iNCIgeD0iMTUiIHk9IjEwMjAuNCIgZmlsbD0iI2ZmZiIvPjwvZz48L2c+PC9zdmc+ "u_plus"
[9]: data:image/svg+xml;base64,PHN2ZyB4bWxuczpzdmc9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI2MCIgaGVpZ2h0PSI1NSIgdmlld0JveD0iMCAwIDYwIDU1IiB2ZXJzaW9uPSIxLjEiPjxzdHlsZT4uczB7ZmlsbDojZmZmO308L3N0eWxlPjxnIHRyYW5zZm9ybT0idHJhbnNsYXRlKDAsLTk5Ny4zNjIxOCkiPjxyZWN0IHdpZHRoPSIzIiBoZWlnaHQ9IjE0IiB4PSI4LjUiIHk9IjEwMzUuNCIgZmlsbD0iI2ZmZiIvPjxyZWN0IHdpZHRoPSIxNCIgaGVpZ2h0PSIzIiB4PSIzIiB5PSIxMDQwLjkiIGZpbGw9IiNmZmYiLz48cmVjdCB3aWR0aD0iMyIgaGVpZ2h0PSIxNCIgeD0iMzguNSIgeT0iMTAzNS40IiBmaWxsPSIjZmZmIi8+PHJlY3Qgd2lkdGg9IjE0IiBoZWlnaHQ9IjMiIHg9IjMzIiB5PSIxMDQwLjkiIGZpbGw9IiNmZmYiLz48Y2lyY2xlIGN4PSIxNSIgY3k9IjEwMzcuNCIgcj0iMTUiIGZpbGw9IiMwMDgwMDAiLz48cmVjdCB3aWR0aD0iMjAiIGhlaWdodD0iNCIgeD0iNSIgeT0iMTAzNS40IiBmaWxsPSIjZmZmIi8+PGNpcmNsZSBjeD0iNDUiIGN5PSIxMDM3LjQiIHI9IjE1IiBmaWxsPSIjMDBmIi8+PHJlY3Qgd2lkdGg9IjIwIiBoZWlnaHQ9IjQiIHg9IjM1IiB5PSIxMDM1LjQiIGZpbGw9IiNmZmYiLz48Y2lyY2xlIGN4PSIzMCIgY3k9IjEwMTIuNCIgcj0iMTUiIGZpbGw9IiNmMDAiLz48cmVjdCB3aWR0aD0iMjAiIGhlaWdodD0iNCIgeD0iMjAiIHk9IjEwMTAuNCIgZmlsbD0iI2ZmZiIvPjwvZz48L3N2Zz4= "u_minus"