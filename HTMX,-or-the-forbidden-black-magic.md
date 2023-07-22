---
tags: 
  - htmx
  - deno
  - typescript
  - javascript
---
With all the hype surrounding htmx lately and its clever way to do something, we were already doing 10~15 years ago, in a way that simply looked like cheating, I quickly hopped on the train and off I go to learn some black magic on the way.
![adava kedavra](https://media.tenor.com/yhFq6N5tvUEAAAAC/avada-kadavra-star-wars.gif)
## DLHX
Stands for <s>stop naming tech product by its acronym</s> **D**eno **L**ightsaber **H**tm**X**

Sounds fun right ?

So that's it for the stack. As for the data persistency I'll stick to Deno's Web Storage API as their really promising KV database is under closed test.

Quick recap about HTMX for those who slept in class.

HTMX is offering a declarative syntax that lets you write app logic inside your markup. The reactivity is done through case-by-case replacement of HTML content received through ajax call to the server.

## Foundation
To test the capabilities of HTMX I'll implement a basic TODO App.

Let's start by creating a server.ts file where we'll centralize the route handling.
```typescript
interface Routes {
  [routeName: string]: (req: Request) => Response | Promise<Response>
}
function routeMatching(req: Request, routes: Routes) {
  const {pathname} = new URL(req.url);
  const route = routes[pathname] ?? (() => new Response("Too bad"))
  return route(req)
}
```
I'm first creating a handy route matching function that will be receiving a `Routes` Object.
```typescript
Deno.serve((req: Request) => routeMatching(req, {
  '/': () => new Response('HELLO WORLD', {
    headers: new Headers([["content-type","text/html"]])
  }),
  '/completed': (req: Request) => {},
  '/delete': (req: Request) => {},
  '/create': async (req: Request) => {}
}));
```
Here we've set the 4 routes needed for our app. 
But for now, only you'll see would be that big *HELLO WORLD* in tiny 16px font size on a white background.
![light mode](https://media.tenor.com/p0FDLRJ5x3MAAAAC/light-theme.gif)

So let's take care of our UI. 
And we'll do that with the module `deno_html` that lets you write your views using ES6 string templating syntax.

So I'm gonna create a new file called `index.html.ts`
```typescript
import { html } from "https://deno.land/x/html/mod.ts";
export default html`
  <!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DLHX</title>
    <script src="https://unpkg.com/htmx.org@1.9.3"></script>
  </head>
  <body>
    <ul id="todolist"></ul>
    <form>
      <input type="text" name="todo-name">
      <button>Create</button>
    </form>
  </body>
  </html>
`
```
Here's our basic UI.

> Note that I've already included HTMX, a simple script tag is needed