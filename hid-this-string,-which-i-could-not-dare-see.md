---
tags: 
  - rust
  - javascript
  - jsdoc
  - js
---
When learning about a new technology and understanding its advantages and ingenuity over what you were previously used to, you kinda feel the urge to use your new super powers all over the place.

A good example for me is the really clever approach the Rust language took with its pattern matching oriented syntax. 
```rs
match time_of_day {
  1..=21 => Some(5),
  22 ..= 24 => Some(0),
  _ => None
}
```

I mostly write JavaScript for a living but really wanna live in the future already. I'll show you my attempt at it. 🦀

## Pattern matching in JS ?
Quick disclaimer, obviously type checking is out of the question right away. So no matching on enum and crazy thing like that.

> The most we can do in JS would be to throw a `typeof` in the mix, which has some limited results as we know it.

So, let's say we have a basic function like : 

```typescript
function myFunc(code: number): string {
  let ret;
  if(code === 200) 
    ret = 'something';
  else if(code === 404)
    ret = 'something else';
  else if(code === 401)
    ret = 'also something else';
  return ret; 
}
```
Ugly imho.

So let's do that instead
```typescript
function myFunc(code: number): string {
  return {
    200: 'something',
    404: 'something else',
    401: 'also something else'
  }[code] ?? 'No matching';
}
```
The `?? 'No matching'` part is the default case.

## Why not just Switch statement ?
Yeah why not (also, spoiler: switches are boring) 

But I wanted this first example to be an introduction as I will follow the same pattern to create a `Match` function and to have the fizzBuzz thing working.

```typescript
type Pattern<Test extends Function,Value> = [Test,Value];

const Match = <Test extends Function,Value>(target: any, patterns: Array<Pattern<Test,Value>>): Value =>
      (patterns.filter(pattern => pattern[0](target)) as Array<Pattern<Test,Value>>).map(pattern => pattern[1])

function fizzBuzz(num: number) {
  return Match(num, [
    [n => n % 3 == 0, 'fizz'],
    [n => n % 5 == 0, 'buzz']
  ]).reduce((prev, curr) => prev += curr);
} 
```
I know this kinda looks bad, but don't worry I won't make an npm package out of it. 🤓