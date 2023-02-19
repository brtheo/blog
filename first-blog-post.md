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

<section>
<div class="codepen" data-height="300" data-theme-id="dark" data-default-tab="css,result" data-slug-hash="RwYWqbw" data-user="brtheo"  data-prefill='{"title":":has() nav menu","tags":[],"scripts":[],"stylesheets":[]}'>
  <pre data-lang="html">&lt;nav>
  &lt;ul>
    &lt;li>Super&lt;/li>
    &lt;li>Cool&lt;/li>
    &lt;li>Effect&lt;/li>
    &lt;li>With&lt;/li>
    &lt;li>:has()&lt;/li>
  &lt;/ul>
&lt;/nav></pre>
  <pre data-lang="css">:root {
  background: black;
  color: white;
  font-family: Sans-Serif;
  font-size: 1.5rem;
}
body {
  display: grid;
  place-content: center;
  min-height: 200px;
}
nav li {
  transition: translate 0.42s ease, scale 0.42s ease;
}
nav li:hover {
  translate: 2rem 0;
  scale: 1.2;
}
nav li:has(~ li:hover) {
  translate: 0 -10vh;
  scale: 0.9;
}
nav li:hover ~ li {
  translate: 0 10vh;
  scale: 0.9;
}
</pre></div>
<script async src="https://cpwebassets.codepen.io/assets/embed/ei.js"></script>
</section>