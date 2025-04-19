Recursively Flattening an Array in Javascript
============================================

It is a common anti-pattern to create recursive functions which call themselves over and over again.

Javascript, in general, does not support [tail call optimization (TCO)](https://en.wikipedia.org/wiki/Tail_call). Actually, [it depends](https://stackoverflow.com/a/54721813), but it's unlikely you'll actually be building something for an engine which supports it. Therefore, the following code will result in a stack-overflow for sufficiently large values of `x`:

```javascript
function factorial(x) {
    if (x <= 0) {
        return 1
    } else {
        return x * factorial(x-1)
    }
}
```

For example:

```
> factorial(333800)
Uncaught RangeError: Maximum call stack size exceeded
    at factorial (REPL7:2:5)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
    at factorial (REPL7:5:20)
```

A common example of infinite recursion resulting in a stack-overflow is in functions which flatten arrays. I have seen two types of these cases:

```javascript
function flatten(array) {
  if (!Array.isArray(array)) return [array]
  return array.reduce(function (a, b) {
    return a.concat(flatten(b))
  }, [])
}
```
and
```javascript
function flattenArray(array) {
  if (!Array.isArray(array)) {
    return [array]
  }
  const resultArr = []
  const _flattenArray = arr => {
    arr.forEach(item => {
      if (Array.isArray(item)) {
        _flattenArray(item)
      } else {
        resultArr.push(item)
      }
    })
  }
  _flattenArray(array)
  return resultArr
}
```

In both cases, we can create a deeply nested array, and attempt to flatten it. This is an example of the first one:

```
> function flatten(array) {
   if (!Array.isArray(array)) return [array]
   return array.reduce(function (a, b) {
     return a.concat(flatten(b))
   }, [])
 }
> const createDeepNestedArray = (depth) => {
   let arr = 1
   for (let i = 0; i < depth; i++) {
     arr = [arr]
   }
   return arr
 }
> flatten(createDeepNestedArray(10000000))
Uncaught RangeError: Maximum call stack size exceeded
    at Function.isArray (<anonymous>)
    at flatten (REPL6:2:14)
    at REPL6:4:21
    at Array.reduce (<anonymous>)
    at flatten (REPL6:3:16)
    at REPL6:4:21
    at Array.reduce (<anonymous>)
    at flatten (REPL6:3:16)
```

There are a few solutions to this problem, but my solution is to use a stack-based approach within the flatten function.

```javascript
function flatten(array) {
  if (!Array.isArray(array)) {
    return [array]
  }

  const result = []
  const stack = [array]

  while (stack.length > 0) {
    const value = stack.pop()

    if (Array.isArray(value)) {
      for (let i = value.length - 1; i >= 0; i--) {
        stack.push(value[i])
      }
    } else {
      result.push(value)
    }
  }

  return result
}
```
