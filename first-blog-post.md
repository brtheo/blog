## New css `:has()` pseudo-classe, a concrete use case
As demonstrated on this wabsite's nav menu, we can do achieve a very cool hover effect by using only few lines of css with the new `:has()`` pseudo-classe.

```css
nav li:has(~ li:hover) {}
nav li:hover ~ li {}
```

Okay it looks tedious at first but easy enough.

The effect we want to achieve is to lift up every element that comes before the hovered element and to push down every elements that comes after the hovered one.

As you can see, we used the `:has()` pseudo-class only in the first line that is meant to lift everything that comes **before**.

The important word here is *before*, prior to the indtroduction of this new pseudo-class we had no way in css of going up the DOM tree.

So let's easily break down the second line to push down every elements that comes after the hovered one.

```css
nav li:hover ~ li
```
May be unknown but rather useful, the sibling selector `~` selects all elements that are **next** siblings of a specified element, in our case the hovered `li`.

Okay, now to lift everything that comes before ?

If forgot the existence of the sibling selector, now that's fully reminded, the problem becomes extremely simple.

We would want to target each elements that **has** a **hovered sibling**.

Fair enough, let's translate it into css
```css
nav li:has(~ li:hover)
```

And that's it !

<iframe height="300" style="width: 100%;" scrolling="no" title=":has() nav menu" src="https://codepen.io/brtheo/embed/RwYWqbw?default-tab=css%2Cresult&theme-id=dark" frameborder="no" loading="lazy" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href="https://codepen.io/brtheo/pen/RwYWqbw">
  :has() nav menu</a> by theo (<a href="https://codepen.io/brtheo">@brtheo</a>)
  on <a href="https://codepen.io">CodePen</a>.
</iframe>