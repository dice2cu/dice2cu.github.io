---
---

In the [last post](/2015/03/01/alchemists-code.html), we analyzed the algorithm used by the [Alchemists](http://czechgames.com/en/alchemists/) board game to generate permutations from a four-letter code.  We found some issues with the algorithm but didn't present any solutions.

So I'm revisiting the topic again to propose a sample algorithm that might work better.  I don't know why I'm writing about this, because I doubt anyone really cares.  But sometimes you have to write the things that no one wants to read, you know?


## Problem Definition

Given a four-letter code, we want to generate a permutation of size 8.  Example:

`DEMO` = `3 5 0 7 4 1 2 6`

A permutation of size 8 can be represented as a number between 0 and `8!`.  So we could restate the problem as generating a pseudo-random number between 0 and 40,319.  Example:

`DEMO` = 18,108

There are a few properties we would like to have in our algorithm:

* Deterministic - The same code always generates the same number.  It doesn't depend on the current date, the hardware that runs the function, etc.

* Uniformity - For any given code code, every permutation should have about the same probability of being generated.

* Pseudorandom - The output from our function should look approximately random.  It should not be feasible for a person to learn anything about the output just from knowing the code (without tool assistance), and a small change in the code should result in a large change in the output.

Also, it's worth noting that we never have to reverse this algorithm: we need to be able to generate a permutation from a code, but we don't need to be able to generate a code from a permutation.


## Original Solution

The [algorithm](/2015/03/01/alchemists-code.html#algorithm) used by the [official companion app](http://gserver.czechgames.com/alchemy/) for the Alchemists board game is based on modular arithmetic and division.  

It is deterministic, which is very important for allowing more than one device to be used in the same game just by sharing codes.  But based on our analysis in the [last post](/2015/03/01/alchemists-code.html#ramifications), it fails to meet the pseudorandom and uniformity goals. 

Can we do better?

## Cryptography

Our ideal solution would be something that approximates a cryptographic [random oracle](http://en.wikipedia.org/wiki/Random_oracle) with an output that is between 0 and 40,319.

And that's the key to finding a good solution to our problem: just knowing that this is a basic cryptography problem that a lot of people have been researching for decades.  We don't have to invent our own custom algorithm if we can take advantage of a standard cryptography algorithm.

Specifically, I would consider using a [hash function](http://en.wikipedia.org/wiki/Cryptographic_hash_function), a [block cipher](http://en.wikipedia.org/wiki/Block_cipher), or a [PRNG](http://en.wikipedia.org/wiki/Pseudorandom_number_generator).  Then it would just be a matter or encoding the input to a form supported by an implementation of the chosen algorithm, and decoding the output into a non-negative integer less than 40,320.


## Sample Solution

My solution is based on the [SHA-1](http://en.wikipedia.org/wiki/SHA-10) hash algorithm.  It's one of the most popular hash functions and any of its theoretical weaknesses don't matter for our application since we don't really need collision or preimage resistance.

So with our algorithm selected, we just have to decide out to encode the input and decode the output.  Warning: it might get a little javascript-heavy in this section.

### Input

The input encoding doesn't really matter as long as long as we're consistent.  Assuming our implementation of SHA-1 accepts a byte array as input, we can just use the same encoding as the official app's algorithm.

Here's an example javascript implementation:

```javascript
function codeToByteArray(code) {
	code = code.toUpperCase();
	var result = new Uint8Array(code.length);
	for (var i = 0; i < code.length; i++) {
		var b = code.charCodeAt(i) - 'A'.charCodeAt(0);
		if (i < 0 || i >= 26) {
			b = 0;
		}
		result[i] = b;
	}
	return result;
}
```

We're keeping the case-insensitive, letters-only restriction from the original algorithm.  But again, just about any encoding would be fine.


### Output

The output is where this gets a little more interesting.

Let's assume our SHA-1 implementation give us output as a byte array.  For the SHA-1 algorithm, this output will always be exactly 20 bytes of output.

Continuing with our javascript example, here's how we could generate the SHA-1 hash using the [Web Cryptography API](http://www.w3.org/TR/WebCryptoAPI/):

```javascript
function generateHashAsync(code) {
	var input = codeToByteArray(code);
	return crypto.subtle.digest({ name: 'sha-1' }, input).then(function(result) {
		return new Uint8Array(result);
	});
}
```

Our allowed output range is a number from 0 to 40,319.  So we need to take our 20-byte hash and decode some part of it into a number in this range.

But there's a problem: our range doesn't exactly fit into a binary number.  15 bits will store a number up to 32,768 (not enough) and 16 bits will store a number up to 65,536 (too much):

```
40319 (base 10) = 1001 1101 0111 1111 (base 2)
```

But we know we need at least two bytes (16 bits), so let's start by taking just the first two bytes.  That will give us a number from 0 to 65,535.  

How do we deal with numbers that are too big? Our first instinct might be to use modulo arithmetic:

```javascript
function generatePermutationIndex(hashByteArray) {
	var permutationIndex = hashByteArray[0] * 256 + hashByteArray[1];
	return permutationIndex % 40320;
}
```

This will get us a valid index, but the problem is that we no longer have a random number.  Smaller results (less than 25,216) will have higher odds of being generated.  In other words, we've failed our uniformity goal.

So what can we do better?  Well, we've got 20 bytes of pseudo-random data to work with.  So if the first two bytes don't give us a valid number, why don't we just move on to the next two?  And if we continue that for all 20 bytes and still don't get a valid number, then this is probably unlikely enough that we'd be ok using the modulo solution as a last resort.

```javascript
function generatePermutationIndex(hashByteArray) {
	var permutationIndex;
	for (var i = 0; i < hashByteArray.length; i += 2)
	{
		permutationIndex = hashByteArray[i] * 256 + hashByteArray[i + 1];
		if (permutationIndex < 40320)
		{
			break;
		}
	}
	return permutationIndex % 40320;
}
```

I'm not sure if this is a perfect solution, but it's definitely better than just using the modulo method.  And there are probably more complicated alternatives we could apply here if we really needed to, like applying the hash function again as in some kind of cycle-walking cipher.


### All Together

We can put all of this together with the following sample javascript implementation:

```javascript
function codeToPermutationAsync(code) {
    return generateHashAsync(code).then(function (output) {
        var index = generatePermutationIndex(output);
        var factoradic = indexToFactoradic(index, 8);
        var permutation = factoradicToPermutation(factoradic);
        return permutation;
    });
};

function generateHashAsync(code) {
    var input = codeToByteArray(code);
    return crypto.subtle.digest({ name: 'sha-1' }, input).then(function (result) {
        return new Uint8Array(result);
    });
}

function codeToByteArray(code) {
    code = code.toUpperCase();
    var result = new Uint8Array(code.length);
    for (var i = 0; i < code.length; i++) {
        var b = code.charCodeAt(i) - 'A'.charCodeAt(0);
        if (i < 0 || i >= 26) {
            b = 0;
        }
        result[i] = b;
    }
    return result;
}

function generatePermutationIndex(hashByteArray) {
    var permutationIndex;
    for (var i = 0; i < hashByteArray.length; i += 2) {
        permutationIndex = hashByteArray[i] * 256 + hashByteArray[i + 1];
        if (permutationIndex < 40320) {
            break;
        }
    }
    return permutationIndex % 40320;
}

function indexToFactoradic(index, size) {
    var factoradicArray = [];
    var remaining = index;
    for (var i = 1; i <= size; i++) {
        factoradicArray[size - i] = remaining % i;
        remaining = (remaining / i) | 0;
    }
    return factoradicArray;
}

function factoradicToPermutation(factoradic) {
    var remaining = [];
    for (var i = 0; i < factoradic.length; i++) {
        remaining.push(i);
    }

    var permutation = []
    for (var i = 0; i < factoradic.length; i++) {
        var factoradicElement = factoradic[i];
        permutation[i] = remaining[factoradicElement];
        remaining.splice(factoradicElement, 1);
    }
    return permutation;
}

codeToPermutationAsync('DEMO').then(function(permutationString) {
	console.log(permutationString);
});
```

A very basic demo that uses this code is available at [https://jsfiddle.net/3mshy0yL/](https://jsfiddle.net/3mshy0yL/).

Note that this code does require some ES6 features (TypedArray, Web Cryptography API), so it will only work on modern browsers (at least without a polyfill).


## Sample Results

We've got our fancy new algorithm now, so let's check to see how well it works with four-letter codes.


### Alchemical Distribution

Using the set of all four-letter codes, we have the following distribution: 

![Chart](/public/images/alchemists_altprobchart.png)

|     | Fern   | Claw   | Mushroom | Flower | Root   | Scorpion | Toad   | Feather |
| --- | ------ | ------ | -------- | ------ | ------ | -------- | ------ | ------- |
| R+  | 12.53% | 12.48% | 12.54%   | 12.47% | 12.43% | 12.56%   | 12.52% | 12.47%  |
| G+  | 12.48% | 12.46% | 12.49%   | 12.54% | 12.51% | 12.47%   | 12.53% | 12.53%  |
| B+  | 12.52% | 12.56% | 12.58%   | 12.51% | 12.44% | 12.48%   | 12.48% | 12.44%  |
| R-  | 12.53% | 12.49% | 12.42%   | 12.48% | 12.57% | 12.47%   | 12.53% | 12.51%  |
| G-  | 12.53% | 12.45% | 12.55%   | 12.52% | 12.47% | 12.44%   | 12.52% | 12.52%  |
| B-  | 12.46% | 12.59% | 12.51%   | 12.49% | 12.49% | 12.51%   | 12.48% | 12.47%  |
| ++  | 12.53% | 12.47% | 12.44%   | 12.47% | 12.56% | 12.50%   | 12.51% | 12.52%  |
| --  | 12.43% | 12.52% | 12.46%   | 12.52% | 12.54% | 12.57%   | 12.42% | 12.54%  |

That's pretty close to uniform, and much closer than the [original algorithm](/2015/03/01/alchemists-code.html#permutations).

### Permutation Distribution

The alchemical distribution looks pretty good, but what about the distribution of permutations?

We have 456,976 possible 4-letter codes and 40,320 possible permutations.  That means with a perfectly uniform distribution, each permutation would be generated from 11 or 12 different codes (456,976 / 40,320 = 11.33).

To show these results, I generated a table of all possible codes and the permutation we get by applying our algorithm to the code.  Then from that table, I generated this chart:


![Chart](/public/images/alchemists_altpermutationdistchart.png)

Where:

* X = the number of codes that generate the permutation
* Y = the number of permutations that are mapped from X codes  
* "Alternate" is the sample algorithm I proposed here
* "Original" is the algorithm from the official companion app.

That is a terrible explanation of what this data is supposed to be, but that shouldn't be a problem because no one will ever read this far.

Here's the data used in the chart:


|     | Alternate | Original |
| --- | --------- | -------- |
|   1 | 6         |          |
|   2 | 24        |          |
|   3 | 113       |          |
|   4 | 307       |          |
|   5 | 735       | 8448     |
|   6 | 1509      | 2112     |
|   7 | 2228      |          |
|   8 | 3326      |          |
|   9 | 4165      |          |
|  10 | 4542      | 17488    |
|  11 | 4877      |          |
|  12 | 4450      | 4372     |
|  13 | 3915      |          |
|  14 | 3150      |          |
|  15 | 2436      |          |
|  16 | 1702      |          |
|  17 | 1194      |          |
|  18 | 717       |          |
|  19 | 435       |          |
|  20 | 241       | 5920     |
|  21 | 126       |          |
|  22 | 66        |          |
|  23 | 38        |          |
|  24 | 11        | 1480     |
|  25 | 5         |          |
|  26 | 1         |          |
|  27 | 1         |          |
|  40 |           | 400      |
|  48 |           | 100      |


So this tells us that we do not have a uniform distribution of permutations.  But we do have what appears to be a normal distribution.

And I think this is probably good enough?  Every permutation is possible with a four-letter code, and the actual ingredient to alchemical mappings are close to a uniform distribution.  

And either way, the distribution looks better than the one from the original algorithm.

## Backwards Compatibility

It's one thing to come up with a new algorithm, but if anyone wanted to actually use a new algorithm then it would be important to consider backwards compatibility.

The printed instructions for the Alchemists game specifically uses the `DEMO` code in some examples, so any changes to the app would probably want to maintain the mapping for at least that code.  And I think it is important to be able to regenerate the results from an old game with a recorded code at any time, even if a new algorithm is implemented.

So if I were to update the official companion app (or make an alternative app) with a new algorithm, I'd keep using the existing algorithm with all four-letter codes.  For the new algorithm, I would add support for five-letter codes (or any length other than four) and only apply it to these new codes.  Then the new five-letter codes could be used when you start a new game, but a player could still enter either four-letter or five-letter codes manually.


## Conclusion

I like the idea of having a section called "Conclusion" at the end of a post.  Doesn't it just feel right?

Anyway, I presented one alternate algorithm here that's based on using the SHA-1 hash algorithm.  But there are all kinds adjustments that could be made to this algorithm:

*  Switch the hash function out for another one (MD5, SHA-256, SHA-3)
*  Replace the hash function with a block cipher (AES, DES) or HMAC to add a key parameter
*  Use a different encoding for the input and/or ouput

And I'm sure there's all kinds of completely different algorithms that would work.  For example, a format-preserving encryption algorithm like [BPS](http://csrc.nist.gov/groups/ST/toolkit/BCM/documents/proposedmodes/bps/bps-spec.pdf) would probably be a good candidate for this type of work.

The key point of this whole thing is to recognize this type of problem as something that a lot of people have spent a lot of time solving in the cryptography domain.