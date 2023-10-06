---
tags:
  - deno
  - github
  - api
  - typescript
---
There's several ways of creating a blog nowadays, whether it is with a legacy CMS such as [Wordpress](https://wordpress.com) whether it is by taking a more modern approach using a headless CMS solution like [Strapi](https://strapi.io) or even building it with [Astro](https://astro.build) through their brand new feature [Content Collections](https://docs.astro.build/en/guides/content-collections/).

Bref, there're plenty of solutions.

## Why not build it ourselves ?
I know right, I had the same idea :nerd_face:

I wanted a solution that would be CMS-like, with automatic time tracking of blog post creation/edition possibility of several authors and easy content management.

When describing like this, it does sound familliar right ? <br>
*Excatly*, I'm basically describing **Git** and **Github**.

So, now that's it's clearer in our head let's summarize what we are going to do :
- Use the Github API
- Use a dedicated repo to store our blog posts
- Use Deno + Fresh
- Implements type safety

## The Github part 
First thing first when it comes to external API, let's grab ourselves a key to authenticate our requests, for github you'll be able to generate one [here](https://github.com/settings/tokens?type=beta).
You can refine the token accesses or just checking every permissions it's up to you. 

Then let's create a new repo, I simply named it `blog`. Create a basic `.md` file and we are good for now.

## The code part
> I'll let you refer to the [official Fresh doc](https://fresh.deno.dev/docs/getting-started/create-a-project) to create a new project and grabbing the basics understanding of this framework as it's beyond the scope of this article.

After, creating a new project with this command
```bash
$ deno run -A -r https://fresh.deno.dev github-blog
```
We're left with this folder structure
```bash
.
├── README.md
├── components/
├── deno.json
├── deno.lock
├── dev.ts
├── fresh.gen.ts
├── import_map.json
├── islands/
├── main.ts
├── routes/
│   └── index.tsx
├── static
│   └── favicon.ico
└── twind.config.ts
```

To start off, let's create a new folder `utils` at the root directory of our project and create a new file that will hold the logics of fetching our blogposts, I came with the funny name of `octoblog.ts`.

First thing we are gonna do is to create the `Headers` object that we'll reuse for all our requests. We need it to of course authenticate our requests.
```typescript
const headers = new Headers();
headers.append('Authorization', `token <TOKEN>`);
headers.append('Accept','application/vnd.github+json');
```
That probably won't be a good idea to inline your private access token right here in the code, rather add it to you env variable and access it through `Deno.env.get(key: string)`.

Then let's create a fetch wrapper so that it's all safely typed.
> Remember, all of this is executed on server side in fresh
> 
```typescript
export class Octoblog {
  private static get<T extends Endpoint>(endpoint: T): Promise<ResponseType<T>> {
    const data = await fetch(endpoint, {
      headers, method: 'GET'
    });
    return await data.json();
  }
}
```

Ok, the idea here is simple : we will define the `Endpoint` type and based on which endpoints we queried, we will receive a proper `ResponseType`.

We will need to hit 3 different endpoints for this : <br> 
`/contents` used to get base64 string of a file
<br>
`/commits` usedd to get metadata like commit date and author
<br>
`/git/trees` used to list all files in a repo

```typescript
type GITBRANCH = "master" | "dev";

const CONTENT = 
  ".../contents/" as const;

const COMMIT = 
  `.../commits?sha=${GITBRANCH}&path=` as 
  `.../commits?sha=${GITBRANCH}&path=`;

const TREE = 
  `.../git/trees/${GITBRANCH}` as 
  `.../git/trees/${GITBRANCH}`;

const ENDPOINTS = {CONTENT, COMMIT, TREE} as const;
type Endpoint = typeof ENDPOINTS[keyof typeof ENDPOINTS];

type EndpointResponseMap = {
  "../contents/": IContent;
  "../commits?sha=dev&path=": ICommit | ICommit[];
  "../commits?sha=master&path=": ICommit | ICommit[];
  "../git/trees/master": ITree;
  "../git/trees/dev": ITree;
};

type ResponseType<T extends Endpoint> = T extends keyof EndpointResponseMap
  ? EndpointResponseMap[T]
  : unknown;
```
That was a lot of boilerplate but necessary ... hold on we're not done yet. We need to define each response types `IContent`, `ICommit` and `ITree`. But luckily for us there's a handy tool to generate a typescript interface based on a json that exists : [quicktype](https://quicktype.io/typescript).

And since we're at it, let's define our `IPost` interface, the actual representation of a blog post in the context of our app.

```typescript
export interface IPost {
  slug: string;
  title: string;
  publishedAt: number;
  content: string;
  author: string;
} 
```

Okay, really we're done with types now.

Let's create implement a new method in our `Octoblog` class to get one post in a repo by its slug.

```ts
public static async getAllPosts(): Promise<IPost[]> {
    const treeElements: TreeElement[] = (await Octoblog.get(ENDPOINTS.TREE)).tree;
    return (await Promise.all(
      treeElements.map( treeElement => Octoblog.getPost(treeElement.path))
    )).sort((postA, postB) => postB.publishedAt - postA.publishedAt);
  }
```
Here we use our fetch wrapper we created earlier `get()` to make a request to the `TREE` endpoint, resolving all the promises returned by `getPost()` (soon) and returning this awaited array of `IPost` ordered by publishing date.

To get a post we need to merge datas we get from two different responses, `ICommit` and `IContent`. To execute them at the same time let's create a helper function around our `get()` in order to do that.

```ts
//yes I lied, another type..
type Param<T> = {
  endpoint: T;
  params?: {
    slug?:string
  }
}
//helper method to do more than one request at a time
private static async getSeveral<T extends Endpoint>(param: Array<Param<T>>): Promise<Array<ResponseType<T>>>{
    return await Promise.all(param.map(req => Octoblog.get(req.endpoint, req.params)));
  }

public static async getPost(slug: string): Promise<IPost> {
    const [commitForPost, contentForPost] = await Octoblog.getSeveral([
      {
        endpoint: ENDPOINTS.COMMIT,
        params: {slug}
      },
      {
        endpoint: ENDPOINTS.CONTENTS,
        params: {slug}
      },
    ]);
    return Octoblog.makePost((contentForPost as IContent), (commitForPost as Array<ICommit>), slug);
  }
```

Finally we need this other helper function `makePost()` to wrap up the `IPost` object creation.

```ts
  private static makePost(contentObject: IContent, commits: Array<ICommit>, slug: string): IPost{
    slug = slug.replace('.md','');
    const title = slug.replaceAll('-',' ');
    const makeDate = (index = -1) => new Date((commits.at(index)?.commit.author.date as Date)).getTime();
    const content = Base64.fromBase64String(contentObject.content).toString();
    const author = commits.at(-1)?.committer.login;
    return {
      slug,
      title,
      publishedAt: makeDate(),
      content, 
      author // possibly here we can add more fields to IPost like authorPicture or lastUpdatedAt contained in ICommit
    } as IPost
  }
```

## Results ! 
And that's it ... *Almost* 
<br>
At least on the github api operations side. We still need to display these beautiful posts to our UI.

So, in Fresh the implementation would look like that, but you can easily figure out how to integrate it with your framework of choice.
```ts
// routes/blog/index.tsx
export const handler: Handlers<IPost[]> = {
  async GET(_req, ctx) {
    const posts = await Octoblog.getAllPosts()
    return ctx.render(posts as IPost[])
  }
}

// And simply define our page component in TSX as Fresh uses Preact to render UI
export default function Blog(props: PageProps<IPost[]>) {
  const posts = props.data;
  return (
    <>
      <Head>
        <title>My octoblog</title>
      </Head>
      <main>
        {posts.map(post => <Post post={post}/>)}
      </main>
    </>
  );
}
```

And voilà, really DONE.

## Conclusion
Well, was it worth it ? <br>
Why not. <br>
Is it optimized ? <br>
I probably don't think so. Having a limited amount of files in a repo is fine but since we are doing 2 fetch requests for each files, it can be limiting for sure.

You can see the source of this very blog implementing this logic on this repo : [coding-room](https://github.com/brtheo/coding-room)
