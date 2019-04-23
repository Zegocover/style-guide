# Building Objects with Array.reduce

As we get more comfortable with programming in a functional style, it can be tempting to write everything as terse expressions. `Array.reduce` (or just `reduce`) is used for building a single return value from an array. In contrast with `Array.map`, which always returns a new array of the same size, `reduce` can return anything -- even another array, this makes it a very powerful function.

One common use for `reduce` is to apply map-like operations to objects rather than arrays. It looks a little like this:

```javascript
// mapResult is some mapping function, similar to the callback when using Array.map
const result = Object.keys(input).reduce((acc, key) => {
  acc[key] = mapResult(input[key]);
  return acc;
}, {});
```

It's possible to use newer syntax to implement this without object mutation:

```javascript
const result = Object.keys(input).reduce(
  (acc, key) => ({
    ...acc,
    [key]: mapResult(input[key]),
  }),
  {},
);
```

The problem is that every iteration of the loop is creating a new object, which is slower than mutating the accumulator. In this case, there we don't get any value from pursuing functional purity because the scope of the accumulator is tightly controlled.

❌ **Don't:** Unnecessarily create temporary objects inside loops, such as when using `Array.reduce`.
