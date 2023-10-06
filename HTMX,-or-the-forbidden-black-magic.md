---
tags: 
  - htmx
  - deno
  - typescript
  - javascript
---
With all the hype surrounding `HTMX` lately and its clever way to do something, we were already doing 10~15 years ago, in a way that simply looked like cheating, I quickly hopped on the train and off I go to learn some black magic on the way.
![adava kedavra](https://media.tenor.com/yhFq6N5tvUEAAAAC/avada-kadavra-star-wars.gif)
## DHX
Stands for <s>stop naming tech product by its acronym</s> **D**eno **H**tm**X**

Sounds fun right ?

So that's it for the stack. As for the data persistency I'll stick to Deno's Web Storage API as [their really promising KV database](https://deno.com/kv) is under closed test.

Quick recap about [`HTMX`](https://htmx.org/) for those who slept in class.

`HTMX` is offering a declarative syntax that lets you write app logic inside your markup. The reactivity is done through case-by-case replacement of HTML content received through ajax call to the server.

## Foundation
To test the capabilities of `HTMX` I'll implement a basic TODO App.

Let's start by creating a `server.ts` file where we'll centralize the route handling.
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
But for now, the only thing you'll see will be that big *HELLO WORLD* in tiny 16px font size on a white background.
![light mode](https://media.tenor.com/p0FDLRJ5x3MAAAAC/light-theme.gif)

So let's take care of our UI. 
And we'll do that with the module [`deno_html`](https://deno.land/x/html@v1.2.0) that lets you write your views using ES6 string templating syntax.

So I'm gonna create a new file called `index.html.ts`
```html
import { html } from "https://deno.land/x/html/mod.ts";
export default html`
  <!DOCTYPE html>
  <html lang="en">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DHX</title>
    <script src="https://unpkg.com/htmx.org@1.9.3"></script>
    <style>
      @import url('https://fonts.googleapis.com/css2?family=Open+Sans&display=swap');
      :root{
        zoom: 150%; 
        background: #242424;
        color: white;
        font-family: 'Open Sans'
      }
      .completed {
        text-decoration:line-through;
      }
    </style>
  </head>
  <body>
    <ul id="todolist"></ul>
    <form>
      <input type="text" name="todo-name">
      <button type="submit">Create</button>
    </form>
  </body>
  </html>
`
```

Then let's update our `server.ts` index route handler method accordingly :

```typescript
import index from './index.html.ts';
Deno.serve((req: Request) => routeMatching(req, {
  '/': () => new Response(index, {
    headers: new Headers([["content-type","text/html"]])
  }),
  ...
}));
```

Here's our basic UI that looks like this : 
![basic ui](https://brtheo.dev/blog-pictures/dhx-basic-ui.png)

> Note that I've already included HTMX, a simple script tag is needed> 

## Creating a Todo

Here comes the fun part. We're actually really start using `HTMX` at this point.

Let's write the front-end logic first
```html
<ul id="todolist"></ul>
<form >
  <input type="text" name="todo-name">
  <button 
    type="submit" 
    hx-post="/create" 
    hx-include="[name='todo-name']" 
    hx-target="#todolist" 
    hx-swap="beforeend"  
  >Create</button>
</form>
```
As you can see with this declarative syntax, everything is happening in the markup through the use of custom attributes that can be prefixed with `data-` if you prefer.

It reads as : *On the click event of the button, do an ajax post request to /create that'll include the value of the input 'todo-name', then print out the response onto the #todolist HTML Element by using the insertAdjacentHtml('beforeend', response) method*

Simple right ? HTMX abstracts what's would normally take like 10 to 20 line of code into only **4**. Amazing.

As for the backend code, that's not so much complicated.

First I'm gonna create a new file `todo.ts` to hold our 'Database' manipulation code.

```typescript
export interface Todo {
  id: number,
  todoName: string
  state: 'completed' | 'deleted' | 'created'
}
```
Let's first define a simple `Todo` type.

```typescript
const todosRef = localStorage.getItem('todos');

export const todos: Array<Todo> = todosRef === null 
  ? (_ => {
    localStorage.setItem('todos',JSON.stringify([]));
    return []
  })() 
  : JSON.parse(localStorage.getItem('todos')!);
```
And here I define my array of todos read from the localStorage if it exists.

```typescript
export function createTodo(todoName: Todo['todoName']) {
  const _todos: Array<Todo> = JSON.parse(localStorage.getItem('todos'));

  const todo = {
    id: parseInt(`${_todos.length +1}`),
    todoName,
    state: 'created',
  } as Todo;

  _todos.push(todo);

  localStorage.setItem('todos',JSON.stringify(_todos));

  return todo;
}

export function completeTodo(id: Todo['id']) {}

export function deleteTodo(id: Todo['id']) {}
```

And here I'm exporting 3 functions that we'll cover along the way.

For now the `createTodo` one as you might've guessed is simply re-setting the todos array to which we've added the newly created todo, to the localStorage.

Back to `server.ts`, we need to call this method to send back the html response we'll insert into our view.

```typescript
'/create': async (req: Request) => {
  const form = await req.formData();
  const todoName = await form.get('todo-name') as string;
  const todo = createTodo(todoName);
  return new Response(???);
```
Here, we're getting the value of the `todo-name` input sent by HTMX and calling our `createTodo` function.
For the response we can easily figure that our, by creating a Todo *component* that will return the needed markup.

```html
//index.html.ts
export function $Todo(todo: Todo) {
  return html`
    <li>  
      <span id="todo-${todo.id}">${todo.todoName}</span>
      <button name=${todo.id}>:OK_hand:</button>
      <button name=${todo.id}>:cross_mark:</button>   
    </li>
  `
}
```
```typescript
//server.ts
import index, { Todo } from './index.html.ts';
...
return new Response(Todo(todo));
...
```

Now upon clicking the create button it should reflect like that on the ui 
![dhx-created](https://brtheo.dev/blog-pictures/dhx-todo.png)

We're still missing one thing though, that is displaying our todos after a page refresh. Simple enough : 
```html
//index.html.ts
import { todos } from "./todo.ts";
<ul id="todolist">${todos.map(todo => $Todo(todo)).join('')}</ul>
```

## Completing a todo
Okay now the second method, again let's focus on the frontend code first.

```html
<span id="todo-${todo.id}">${todo.todoName}</span>
<button 
  name=${todo.id}
  hx-put="/completed"
  hx-swap="none"
>👌</button>
```
*On the click event of the button, do an ajax put request to /completed that'll send the trigger target's name, then do nothing*

As for the handling this request : 
```typescript
//todos.ts
export function completeTodo(id: Todo['id']) {
  const _todos: Array<Todo> = JSON.parse(localStorage.getItem('todos')!)
  .map((todo: Todo) => {
    if(todo.id === id) {
      return {
        ...todo,
        state: todo.state === 'created' ? 'completed' : 'created'
      }
    } else return todo;
  })!;

  localStorage.setItem('todos',JSON.stringify(_todos));
}

//server.ts
...
'/completed': (req: Request) => {
  const todoId = parseInt(req.headers.get('hx-trigger-name')!);
  completeTodo(todoId)
  return new Response('ok');
},
...
```
As you can see, HTMX will send out the infos we need (the attribute `name` of the button that triggers the call, whose value was `todo.id`) in the request Headers.

Ok, but we want to reflect back the updated state of our todo to our view, so let's bring a new member to the party : [`hyperscript`](https://hyperscript.org/). Hyperscript let's you write declarative JS code in the markup, useful for event handling. Like HTMX, only one script tag is enough to start using hyperscript
```html
<script src="https://unpkg.com/hyperscript.org@0.9.9"></script>
```

That perfectly suits our needs as HTMX will dispatch a bunch of custom events along the way, that we can simply take advantage of in hyperscript.

```html
...
<style>
  .completed {
    text-decoration: line-through;
  }
</style>
...
<span id="todo-${todo.id}">${todo.todoName}</span>
<button 
  ...
  _="on htmx:afterRequest toggle .completed on 
  #todo-${todo.id}"
```
would translate to something like 
```typescript
button.addEventListener('html:afterRequest', () => todoRef.classList.toggle('.completed'))
```

But again, upon reloading the page, we'll lose track of the completed state, so let's write a simple `classMap` function.
```typescript
function classMap(input: string) {
  return {
  'completed': 'completed'
  }[input] ?? ''
}
...
 `<span 
  class=${classMap(todo.state)}
  id="todo-${todo.id}"
>${todo.todoName}</span>`
```

## Delete
You know the way already
```html
<li id="row-${todo.id}">
  ...
  <button
    name=${todo.id}
    hx-put="/delete"
    hx-swap="none"
    _="on htmx:afterRequest remove 
    #row-${todo.id}"
  >:cross_mark:</button>
...
```
```typescript
//server.ts
...
'/delete': (req: Request) => {
  const todoId = parseInt(req.headers.get('hx-trigger-name')!);
  deleteTodo(todoId);
  return new Response('ok');
},
...
//todo.ts
export function deleteTodo(id: Todo['id']) {
  let _todos: Array<Todo> = JSON.parse(localStorage.getItem('todos'));

  _todos = _todos.filter((todo: Todo) => todo.id !== id);

  localStorage.setItem('todos',JSON.stringify(_todos));
}
```

Last thing we can add is this line of hyperscript to reset the input value after creating a new todo
```html
<form _="on htmx:afterRequest reset() me">
```
Voilà !

Only thing left would be to throw a fully customized UI using TailwindCSS at it and we're ready to call ourselves true *HTML DEVELOPER*.

Joke aside, I really see a future where we would only write frontend logic in the HTML directly. 

Feel free to look at the code on my [github](https://github.com/brtheo/dhx).
Bye :waving_hand: