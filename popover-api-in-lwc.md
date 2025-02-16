---
tags:
  - lwc
  - salesforce
  - experience-cloud
  - lwc
---
# Popover API in LWC: A Modern Way to Create Popovers

The Popover API is an exciting new web platform feature that brings native support for creating popup content without the need for complex JavaScript logic or third-party libraries. This API provides built-in animations, keyboard navigation, and focus management - all with just a few HTML attributes. While currently in limited browser support, it represents the future of how we'll handle tooltips, menus, and other popup content on the web.

The API shines in its simplicity - by adding `popover` and `popovertarget` attributes to your elements, you get a smooth, animated popup that follows all the best practices for accessibility and user interaction. For a deep dive into the API's capabilities, check out the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/API/Popover_API).

## The Challenge in LWC

While implementing the Popover API in Lightning Web Components (LWC) seems straightforward at first, we quickly run into a challenge. The API requires that we link our trigger element to our popover using an ID:

```html
<button popovertarget="my-popover">Open Popover</button>
<div id="my-popover" popover>Popover content</div>
```

However, the LWC rendering engine will generate dynamically element's ID to ensure uniqueness across components. This means we can't hardcode IDs in our template, as they'll be transformed at runtime... Kind of...

For example, if we write:

```html
<button popovertarget="my-popover">Open Popover</button>
```

LWC will generate something like:
```html
<button popovertarget="my-popover-1234">Open Popover</button>
```

## The Solution

Render the element that has the attribute `[popovertarget]` only when our component is aware of that `[popover]` element's dynamic ID

```html
<template>
  <template lwc:if={popoverId}>
    <button
      popovertarget={popoverId}>
    </button>
  </template>
  <aside
    id="popover"
    popover="manual">
  </aside>
</template>
```

Then, on the JS side

```js
import { LightningElement, api, track } from 'lwc';

export default class myLWC extends LightningElement {
 @track $popover;

 set popover(v) {
  this.$popover = v;
  if (v.id) this.renderedCallback();
 }
  get popoverId() {
    return this.$pop?.id;
  }

  renderedCallback() {
    if(this.popoverId === undefined)
      this.popover = this.template.querySelector('[popover]');
  }
}
```

## Anchor positioning
The Popover API provides powerful positioning capabilities through the anchor attribute. This attribute determines how the popover positions itself relative to its trigger element.

The way it works is by setting the same archor name on the triggering and the target element.
```css
/*triggering element*/
[popovertarget] {
  anchor-name: --uniqueName;
}
/*target element*/
[popover] {
  position-anchor: --uniqueName;
}
```
Doing that, both elements are now linked, meaning we can position the target element based on the triggering element position.
```css
/*triggering element*/
[popovertarget] {
  anchor-name: --uniqueName;
}
/*target element*/
[popover] {
  position-anchor: --uniqueName;
  top: anchor(top);
  bottom: anchor(bottom);
  right: anchor(right);
  left: anchor(left);
}
```
Now to ensure uniqueness for our anchor name in LWC we could simply reuse our `popoverId` property.
```js
get anchorName() {
  return `anchor-name: --${this.popoverId}`;
}
get positionAnchor() {
  return `position-anchor: --${this.popoverId}`;
}
```
```html
<template>
  <template lwc:if={popoverId}>
    <button
      popovertarget={popoverId}
      style={anchorName}>
    </button>
  </template>
  <aside
    id="popover"
    style={positionAnchor}
    popover="manual">
  </aside>
</template>
```

## Adding animations
To animate the entering and exiting state is by leveraging the new css rule `@starting-style` and the property `transition-behavior: allow-discrete`.

You can add to the head of your community these styles:
```css
[popover] {
  transition:
    display .2s allow-discrete,
    opacity .2s ease,
    scale .2s ease;

  &:not(:popover-open) {
    opacity: 0;
    scale: 0;
  }

  &:popover-open {
    opacity: 1;
    scale: 1;
  }

  @starting-style {
    &:popover-open {
      opacity: 0;
      scale: 0;
    }
  }
}
```
Of course you can customize the different behaviours according to your needs or even creating custom properties to allow for more in depths customization.

## Creating a Reusable Popover Component

Let's create a dedicated popover component that we can reuse across our application:

```javascript
// c-popover/popover.js
import { LightningElement, api } from 'lwc';

export default class Popover extends LightningElement {
  @track $popover;

 set popover(v) {
  this.$popover = v;
  if (v.id) this.renderedCallback();
 }
  get popoverId() {
    return this.$pop?.id;
  }
  get anchorName() {
    return `anchor-name: --${this.popoverId}`;
  }
  get positionAnchor() {
    return `position-anchor: --${this.popoverId}`;
  }

  renderedCallback() {
    if(this.popoverId === undefined)
      this.popover = this.template.querySelector('[popover]');
  }
}
```

```html
<!-- c-popover/popover.html -->
<template>
  <template lwc:if={popoverId}>
    <button
      style={anchorName}
      popovertarget={popoverId}>
      <slot name="thumb"></slot>
    </button>
  </template>
  <aside
    style={anchorConsumer}
    id="popover"
    popover="manual">
    <slot></slot>
  </aside>
</template>
```

```css
/* c-popover/popover.css */
button[popovertarget] {
  all: unset;
  cursor:pointer;
}

[popover] {
  top: anchor(var(--popoverTop));
  bottom: anchor(var(--popoverBottom));
  right: anchor(var(--popoverRight));
  left: anchor(var(--popoverLeft));
  translate: var(--popoverX) var(--popoverY);
}
```

Now we can use our popover component like this:

```html
<c-popover>
    <h2 slot="thumb">Popover Title</h2>
    <p>This is the popover content that will be displayed when the button is clicked.</p>
</c-popover>
```
So far the popover will show when clicking on the triggering element, but what if we want to show it on hovering it ?

I got you covered.
You can add
```css
/* c-popover/popover.css */
:host[data-hover] button[popovertarget]:hover + [popover] {
  opacity: 1;
  scale: 1;
}
```
```html
<c-popover data-hover>
    <h2 slot="thumb">Popover Title</h2>
    <p>This is the popover content that will be displayed when the button is hovered.</p>
</c-popover>
```

Here's a demo of how it looks like in our own application:

![popover api demo](https://raw.githubusercontent.com/brtheo/blog/refs/heads/master/assets/popover-lwc.gif "popover api lwc")

This implementation gives us a reusable popover component that:
- Handles ID generation and linking automatically
- Supports customizable positioning
- Allows for custom content through slots
- Provides basic styling that can be customized
- Takes advantage of the native Popover API animations and behavior

With this foundation, you can build more complex popovers by adding features like:
- Custom triggers (not just buttons)
- Different animation styles
- Advanced positioning options
- Close buttons
- Backdrop options

Remember to check browser compatibility and consider providing a fallback for browsers that don't yet support the Popover API. Despite limited current support, implementing the API now positions your application to take advantage of this native functionality as it becomes more widely available.
