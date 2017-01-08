---
title: "More Recursive Data Structures"
layout: default
tags: [allonge, noindex]
---

In [Why recursive data structures?](http://raganwald.com/2016/12/27/recursive-data-structures.html), we met `multirec`, a *recursive combinator*:

```javascript
function mapWith (fn) {
  return (mappable) => mappable.map(fn);
}

function multirec({ indivisible, value, divide, combine }) {
  return function myself (input) {
    if (indivisible(input)) {
      return value(input);
    } else {
      const parts = divide(input);
      const solutions = mapWith(myself)(parts);

      return combine(solutions);
    }
  }
}
```

We used `multirec` to implement [quadtrees][quadtree] and coloured quadtrees (the full code for creating and rotating quadtrees and coloured quatrees is [below](#quadtrees)).

Although we talked about the relative performance between quadtrees and coloured quadtrees, our focus was on the notion of an isomorphism between the data structures and the algorithms. Today, we're going to talk about performance.

---

[![Index Cards](/assets/images/dog-faeces.png)](https://www.flickr.com/photos/robadob/229219543)

*Recursion is a pile of dog faeces, © 2006 Robin Corps, [some rights reserved][cc-by-sa-2.0]*

---

### optimization

Performance-wise, naïve array algorithms are O _n_, and naïve quadtree algorithms are O _n_ log _n_. Coloured quadtrees are worst-case O _n_ log _n_, but are faster than naïve quadtrees whenever there are regions that are entirely white or entirely black, because the entire region can be handled in one 'operation.'

They can even be faster than naïve array algorithms if the image contains enough blank regions. But can they be _even faster_?

The general idea behind coloured quadtrees is that if we know a way to compute the result of an operation (whether rotation or superimposition) on an entire region, we don't need to recursively drill down and do the operation on every cell in the region. We save O _n_ log _n_ operations where _n_ is the size of the region.

We happen to know that all-white or all-black regions are a special case for rotation an superimposition, so coloured quadtrees optimize that case. But if we could find an even more common case, we could go even faster.

One interesting special case is this: If we've done the operation on an identical quadrant before, we could remember the result instead of recomputing it.

For example, if we want to rotate:

```
⚪️⚪️⚪️⚫️⚫️⚪️⚪️⚪️
⚪️⚪️⚫️⚪️⚪️⚫️⚪️⚪️
⚪️⚫️⚫️⚪️⚪️⚫️⚫️⚪️
⚫️⚪️⚪️⚫️⚫️⚪️⚪️⚫️
⚫️⚪️⚪️⚫️⚫️⚪️⚪️⚫️
⚪️⚫️⚫️⚪️⚪️⚫️⚫️⚪️
⚪️⚪️⚫️⚪️⚪️⚫️⚪️⚪️
⚪️⚪️⚪️⚫️⚫️⚪️⚪️⚪️
```

We would divide it into:

```
⚪️⚪️⚪️⚫️ ⚫️⚪️⚪️⚪️
⚪️⚪️⚫️⚪️ ⚪️⚫️⚪️⚪️
⚪️⚫️⚫️⚪️ ⚪️⚫️⚫️⚪️
⚫️⚪️⚪️⚫️ ⚫️⚪️⚪️⚫️

⚫️⚪️⚪️⚫️ ⚫️⚪️⚪️⚫️
⚪️⚫️⚫️⚪️ ⚪️⚫️⚫️⚪️
⚪️⚪️⚫️⚪️ ⚪️⚫️⚪️⚪️
⚪️⚪️⚪️⚫️ ⚫️⚪️⚪️⚪️
```

We would complete the first, second, third, and fourth quadrants. They're all different. However, consider computing the first quadrant.

We divide it into:

```
⚪️⚪️ ⚪️⚫️
⚪️⚪️ ⚫️⚪️

⚪️⚫️ ⚫️⚪️
⚫️⚪️ ⚪️⚫️
```

It's first and third sub-quadrants are unique, but the second and fourth are identical, so after we have rotated:

```
⚪️⚪️
⚪️⚪️

⚪️⚫️
⚫️⚪️

⚫️⚪️
⚪️⚫️
```

If we have saved our work, we don't need to rotate

```
⚪️⚫️
⚫️⚪️
```

Again, because we have already done it and have saved the result. So we save 25% of the work to compute the first quadrant.

And it gets better. The second quadrant subdivides into:

```
⚫️⚪️ ⚪️⚪️
⚪️⚫️ ⚪️⚪️

⚪️⚫️ ⚫️⚪️
⚫️⚪️ ⚪️⚫️
```

We've seen all four of these sub-quadrants already, so we can rotate them in one step, looking up the saved result. The same goes for the third and fourth quadrants, we've seen their sub-quadrants before, so we can do each of their sub-quadrants in a single step as well.

Once we have rotated three sub-quadrants, we have done all the computation needed. Everything else is saving and looking up the results.

Le's write ourselves a simple implementation.

---

[![Index Cards](/assets/images/index-cards.jpg)](https://www.flickr.com/photos/paulk/3080211705)

*Index cards in the public library of Trinidad, Cuba, © 2008 Paul Keller, [some rights reserved][cc-by-2.0]*

---

### memoization

We are going to memoize the results of an operation. We'll use rotation again, as it's a simple case, and thus we can focus on the memoization code. We won't worry about colouring our quadtrees (although implementing colour and memoization optimizations can be valuable for some cases).

Here's our naïve quadtree rotation again:

```javascript
const rotateQuadTree = multirec({
    indivisible : isString,
    value : itself,
    divide: quadTreeToRegions,
    combine: regionsToRotatedQuadTree
  });
```

In principle, our algorithm will consists of, "Do we already know how to rotate this? Yes? Return the answer. No? Rotate it, save the answer, and return the answer."

As seen [elsewhere](https://github.com/raganwald/javascript-allonge-six/blob/master/manuscript/markdown/2.Objects%20and%20State/recipes/memoize.md), we can use `memoize` to take any function and give it this exact behaviour. Unfortunately, this won't work:

```javascript
const memoized = (fn, keymaker = JSON.stringify) => {
    const lookupTable = {};

    return function (...args) {
      const key = keymaker.apply(this, args);

      return lookupTable[key] || (lookupTable[key] = fn.apply(this, args));
    }
  };

const rotateQuadTree = memoized(
    multirec({
        indivisible : isString,
        value : itself,
        divide: quadTreeToRegions,
        combine: regionsToRotatedQuadTree
    })
  );
```

As explained, the memoization has to be applied to the function that is being called recursively. In other words, we need to memoize the function _inside_ `multirec`. If we don't, we memoize the first call (for the entire image), but none of the others.

We can do this properly with a new combinator and a function to generate keys:[^würst]

[^würst]: This is the worst possible implementation, because it always does an O _n_ log _n_ computation to derive the key for a square. But we'll come back to this later.

```javascript
function memoizedMultirec({ indivisible, value, divide, combine, key }) {
  const myself = memoized((input) => {
    if (indivisible(input)) {
      return value(input);
    } else {
      const parts = divide(input);
      const solutions = mapWith(myself)(parts);

      return combine(solutions);
    }
  }, key);

  return myself;
}

const catenateKeys = (keys) => keys.join('');

const würstKey = multirec({
    indivisible : isString,
    value : itself,
    divide: quadTreeToRegions,
    combine: catenateKeys
  });

const memoizedRotateQuadTree = memoizedMultirec({
      indivisible : isString,
      value : itself,
      divide: quadTreeToRegions,
      combine: regionsToRotatedQuadTree,
      key: würstKey
  });
```

[^y]: There is another, far more delightful way to memoize recursive functions: You can read about it in  [Fixed-point combinators in JavaScript: Memoizing recursive functions](http://matt.might.net/articles/implementation-of-recursive-fixed-point-y-combinator-in-javascript-for-memoization/).

---

[![Game of Life 6](/assets/images/glider.jpg)](https://www.flickr.com/photos/oskay/6838125520)

*Detail from "Game of Life 6," © Windell Oskay, [some rights reserved][cc-by-2.0]*

---

### <a name="quadtrees"></a>appendix: naïve quad trees and coloured quad trees

```javascript
const isOneByOneArray = (something) =>
  Array.isArray(something) && something.length === 1 &&
  Array.isArray(something[0]) && something[0].length === 1;

const contentsOfOneByOneArray = (array) => array[0][0];

const firstHalf = (array) => array.slice(0, array.length / 2);
const secondHalf = (array) => array.slice(array.length / 2);

const divideSquareIntoRegions = (square) => {
    const upperHalf = firstHalf(square);
    const lowerHalf = secondHalf(square);

    const upperLeft = upperHalf.map(firstHalf);
    const upperRight = upperHalf.map(secondHalf);
    const lowerRight = lowerHalf.map(secondHalf);
    const lowerLeft= lowerHalf.map(firstHalf);

    return [upperLeft, upperRight, lowerRight, lowerLeft];
  };

const regionsToQuadTree = ([ul, ur, lr, ll]) =>
  ({ ul, ur, lr, ll });

const arrayToQuadTree = multirec({
    indivisible: isOneByOneArray,
    value: contentsOfOneByOneArray,
    divide: divideSquareIntoRegions,
    combine: regionsToQuadTree
  });

const isString = (something) => typeof something === 'string';

const itself = (something) => something;

const quadTreeToRegions = (qt) =>
  [qt.ul, qt.ur, qt.lr, qt.ll];

const regionsToRotatedQuadTree = ([ur, lr, ll, ul]) =>
  ({ ul, ur, lr, ll });

const rotateQuadTree = multirec({
    indivisible : isString,
    value : itself,
    divide: quadTreeToRegions,
    combine: regionsToRotatedQuadTree
  });

const combinedColour = (...elements) =>
  elements.reduce((acc, element => acc === element ? element : '❓'))

const regionsToColouredQuadTree = ([ul, ur, lr, ll]) => ({
    ul, ur, lr, ll, colour: combinedColour(ul, ur, lr, ll)
  });

const arrayToColouredQuadTree = multirec({
  indivisible: isOneByOneArray,
  value: contentsOfOneByOneArray,
  divide: divideSquareIntoRegions,
  combine: regionsToColouredQuadTree
});

const colour = (something) => {
    if (something.colour != null) {
      return something.colour;
    } else if (something === '⚪️') {
      return '⚪️';
    } else if (something === '⚫️') {
      return '⚫️';
    } else {
      throw "Can't get the colour of this thing";
    }
  };

const isEntirelyColoured = (something) =>
  colour(something) !== '❓';

const rotateColouredQuadTree = multirec({
    indivisible : isEntirelyColoured,
    value : itself,
    divide: quadTreeToRegions,
    combine: regionsToRotatedQuadTree
  });
```

---

### afterward

There is more to read about `multirec` in the previous essays, [From Higher-Order Functions to Libraries And Frameworks](http://raganwald.com/2016/12/15/what-higher-order-functions-can-teach-us-about-libraries-and-frameworks.html) and [Why recursive data structures?](http://raganwald.com/2016/12/27/recursive-data-structures.html).

Have an observation? Spot an error? You can open an [issue](https://github.com/raganwald/raganwald.github.com/issues/new), or [edit this post](https://github.com/raganwald/raganwald.github.com/edit/master/_posts/2017-01-07-more-recursive-data-structures.md) yourself.

---

### notes

[anamorphism]: https://en.wikipedia.org/wiki/Anamorphism
[catamorphism]: https://en.wikipedia.org/wiki/Catamorphism
[cc-by-2.0]: https://creativecommons.org/licenses/by/2.0/
[cc-by-sa-2.0]: https://creativecommons.org/licenses/by-sa/2.0/
[reddit]: https://www.reddit.com/r/javascript/comments/5jdjo6/from_higherorder_functions_to_libraries_and/
[Ember]: http://emberjs.com/