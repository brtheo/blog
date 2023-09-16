---
tags: 
  - go
  - lit
  - wails
  - typescript
---
A while back I've experimented with Go, implementing a file system based routing to be used with the highly experimental technology that is [the router package for Lit](https://github.com/lit/lit/tree/main/packages/labs/router). This original goal of this experimentation was for a desktop app concept, so a choice was made to use [Wails](https://wails.io/fr/) an Electron-like solution to build desktop app through a web UI, but using the power of Go under the hood instead of the good old Node. 
## Folder structure
To build a file system based routing solution, we have to respect a certain hierarchy in the way we create the different files composing our app as it will reflect back to our routing system.
```bash
.
├── app.go
├── build
├── frontend
│   ├── dist
│   ├── index.html
│   ├── node_modules
│   ├── package.json
│   ├── src
│   │   ├── components
│   │   │   └── my-component.ts
│   │   ├── config
│   │   │   └── routes.conf.ts
│   │   ├── my-app.ts
│   │   ├── pages
│   │   │   ├── editor
│   │   │   │   ├── index.ts
│   │   │   │   └── sub
│   │   │   │       └── [id].ts
│   │   │   └── index.ts
│   │   └── vite-env.d.ts
│   ├── vite.config.ts
├── go.mod
├── go.sum
├── main.go
└── wails.json
```
This is roughly what we have and how we want it to behave.
Notice the folder `pages`, this is effectively the root of our router. Any `index.ts` file present in the root of a folder should contain a `lit` component that will be rendered when the current URL matches against an a [URL Pattern](https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API) Object generated in our Go App.
Nested routes and named parameters are supported as demonstrated with the path `/editor/sub/:id` mapped to `/pages/editor/sub/[id].ts`.

Ok let's begin.

## Entry point and lit router
The app entry point will be a lit component that will simply call the `outlet()` method of the `lit router` package inside its render function.

```typescript
import {html, LitElement} from 'lit'
import {customElement} from 'lit/decorators.js'
import {Router} from '@lit-labs/router/router.js';
import routes from './config/routes.conf'

@customElement('my-app')
export default class MyApp extends LitElement {
  private router = new Router(this, routes);
  override render() {
    return html`
      ${this.router.outlet()}
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'my-app': MyApp;
  }
}
```

Then let's define our routes... Hold on, go has to do that right ?

Correct, so let's create a `GetRoutes()` function in `App.go` that we will import in our `routes.conf.ts` file.

```go
type Route struct {}

func (a *App) GetRoutes() (routes []Route) {
	wd, err := os.Getwd()
	check(err)

	routesMap := make(map[string][]string)
	routesArr := make([]string, 0)

	routesMap, routesArr = recursiveRoute(filepath.Join(wd, "frontend", "src", "pages"), string(os.PathSeparator), routesMap, routesArr)

	routes = makeRoutes(routesMap)
	fmt.Println(routes)
	return
}
```

Notice the `recursiveRoute()` function meaningful in our case to handle these types of route `/pages/sub/:id`.

```go
func recursiveRoute(path, pathPrefix string, routesMap map[string][]string, routes []string) (map[string][]string, []string) {
	entries, err := os.ReadDir(path)
	check(err)
	dirs := strings.Split(path, string(os.PathSeparator))
}
```
First we read the entries in the specified path and create a `dirs` Array.

```go
if dirs[len(dirs)-1] != "pages" {
  pathPrefix += dirs[len(dirs)-1] + string(os.PathSeparator)
}
routes = mapTo(entries, fs.DirEntry.Name)
```
Then, if we're not at the root `/pages` we go deeper in the folder and we begin to append in the `routes` array using a handy util method `mapTo()`
```go
func mapTo[Type, ReturnType any] (data []Type, f func(Type) ReturnType) (res []ReturnType) {
	res = make([]ReturnType, 0, len(data))
	for _, e := range data {
		res = append(res, f(e))
	}
	return
}
```

At this stage we want to identify if we've reached any `.ts` file and remove this extension from the yielded result (as we will use the name of the file to instantiate a component) and sort the possible sub routes.
```go
routes, subRoutes := filterWithOpt(routes, strings.HasSuffix, ".ts")
routes = transformTo(routes, strings.TrimSuffix, ".ts")
```
So, again let's create those util methods :
```go
func filterWithOpt[Type any] (data []Type, f func(Type, string) bool, opt string) (res []Type, res2 []Type) {
	res = make([]Type, 0, len(data))
	res2 = make([]Type, 0, len(data))
	for _, e := range data {
		if f(e, opt) {
			res = append(res, e)
		} else {
			res2 = append(res2, e)
		}
	}
	return
}
func transformTo[Type, ReturnType any] (data []Type, f func(Type, string) ReturnType, opt string) (res []ReturnType) {
	res = make([]ReturnType, 0, len(data))
	for _, e := range data {
		res = append(res, f(e, opt))
	}
	return
}
```
We then allocate the `routes` array to the `routesMap` indexed by the current path and what is left to be done is simple re calling the `recursiveRoute()` function before returning a complete result.
```go
routesMap[pathPrefix] = routes
var tempMap map[string][]string
for _, route := range subRoutes {
  tempMap, routes = recursiveRoute(filepath.Join(path, route), pathPrefix, routesMap, routes)
  for k, v := range tempMap {
    routesMap[k] = v
  }
}
return routesMap, routes
```
`routesMap` has this shape at this point :
```json
{
  "/": [
    "/index"
  ],
  "/editor/": [
    "/index"
  ],
  "/editor/sub/": [
    "/[id]"
  ]
}
```
But the `lit-router` package expect us to provide a route in this format 
```typescript
interface BaseRouteConfig {
  name?: string | undefined;
  render?: (params: {
      [key: string]: string | undefined;
  }) => unknown;
  enter?: (params: {
      [key: string]: string | undefined;
  }) => Promise<boolean> | boolean;
}
```

We first initialize an empty struct `Route`, let's make it more relevant so that it'll provide an easy and effective api to work with on the JS side.

```go
type Route struct {
  Component string // HTML Tag name
	Path      string // URL to reached to display the component
	Filename  string // URL of the component's file needed for its dynamic import
	Args      bool // Custom Element's attribute (reflects the param routes)
}
```
With this established we can start writing some TS in the `routes.conf.ts` to map our Go `Route` struct to an expected `BaseRouteConfig` typescript type.

```typescript
import type { RouteConfig } from "@lit-labs/router/routes";
import {unsafeHTML} from 'lit/directives/unsafe-html.js'
import { GetRoutes } from '../../wailsjs/go/main/App'

const makeRoutes = async (): Promise<RouteConfig[]> => {
  return (await GetRoutes()).map(route => 
    Object.create({
      path: route.Path,
      render: () =>    
        unsafeHTML(
          `<${route.Component} ${makeArgs(route)} >
          </${route.Component}>`
        ),
      enter: async () => await import(`../pages${route.Filename}`)
    }) as RouteConfig 
  ) as RouteConfig[]
}
export default await makeRoutes()
```
The logic is there in place, we can finish up the Go side of the code by implementing the `makeRoutes()` function from earlier.

```go
func makeRoutes(m map[string][]string) (routes []Route) {
  routes = make([]Route, 0)
  for dir, files := range m {
		for i := 0; i <= len(files)-1; i++ {
			file := files[i]
			file = strings.TrimPrefix(file, string(os.PathSeparator))
			cmp := file
			args := false
			path := dir
			if strings.HasSuffix(dir, string(os.PathSeparator)) && len(dir) > 1 {
				path = strings.TrimSuffix(dir, string(os.PathSeparator))
			}
			if len(strings.Split(dir, string(os.PathSeparator))) > 2 {
				dirs := strings.Split(dir, string(os.PathSeparator))
				cmp = dirs[len(dirs)-2]
        // map /some/[id].ts to /some/:id 
				if strings.Contains(file, "[") {
					formatedFileName := ReplaceDynamicPattern(file)
					path += string(os.PathSeparator) + formatedFileName
					args = true
				}
			}
      // Route object creation
			r := Route{
				cmp + "-page",
				strings.ReplaceAll(path, string(os.PathSeparator), "/"),
				strings.ReplaceAll(dir+file, string(os.PathSeparator), "/"),
				args,
			}
			routes = append(routes, r)
		}
	}
	return
}
func ReplaceDynamicPattern(input string) string {
	re := regexp.MustCompile(`\[(\w+)\]`)
	return re.ReplaceAllString(input, ":$1")
}
```

And voila, the Go part is done. Now let's go back to typescript to implement the `makeArgs()` function we left.
We are gonna use the URL Pattern API to dynamically assign our Lit's component reactive prop bound to an attribute based on the URL's param.

Accourding to the doc for the URL Pattern API : 
```javascript
const pattern = new URLPattern("http{s}?://*.example.com/books/:id");
let match = pattern.exec("https://store.example.com/books/123");
console.log(match.pathname); // { input: "/books/123", groups: { "id": "123" } }
```
This is exactly what we want.

```typescript
const makeArgs = (route): string => { 
  let ret: string | null = ''
  if(route.Args) {
    const {origin, href} = location;
    const pattern = new URLPattern(`${origin}${route.Path}`);
    for(const [k,v] of Object.entries(pattern.exec(href).pathname.groups)) {
      ret += `${k}=${v} `; // It just works
    }
  }
  return ret;
}
```

## Putting it all together
Okay, now let's have some fun and create our page components
```typescript
// /pages/index.ts
import {html, LitElement} from 'lit'
import {customElement, property} from 'lit/decorators.js'

@customElement('index-page')
export default class IndexPage extends LitElement {
  @property({type: Number}) count = 3
  render() {
    return html`
    <section>
      <button @click=${() => this.count++}>click to   increase count and to go a different sub page
      </button>
      <h1>hello home page world count : ${this.count}</h1>
      <a href="/editor">editor page</a>
      <a href=${"/editor/sub/"+this.count}>editor sub ${this.count}</a>
    </section>
    `
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'index-page': IndexPage
  }
}

// pages/editor/sub/[id].ts
import {html, LitElement} from 'lit'
import {customElement, property} from 'lit/decorators.js'

@customElement('sub-page')
export default class SubPage extends LitElement {
  @property({type:String}) id
  override render() {
    return html`
    <section>
      <h1>prop received from url /editor/sub/:id = ${this.id}</h1>
      <a href="/">home page</a>
    </section>
    `
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'sub-page': SubPage
  }
}
```
Resulting in 
![](https://brtheo.dev/blog-pictures/golitfs.gif)

==> [Link of the repo](https://github.com/brtheo/Go-lit-with-fsRouting) <==
