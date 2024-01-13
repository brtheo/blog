---
tags:
  - go
  - htmx
  - templ
  - echo
  - blueprint
---
There's some special moments when for example a handful of successive nice events occur or planets mysteriously perfeclty align or, *hey*, a new terribly simple yet efficient techstack pops up.

That's it you've guessed it right, the last option it'll be.

Introducing the magical techstack : 
* Go
* Templ
* HTMX
* WebComponents (for front-end only logic)

To get a bit more familiar whit those technologies I thaught that it could be a good idea to rewrite (again) my blog using this stack..

# Okay, Go is basically a better TypeScript
![](https://brtheo.dev/blog-pictures/golang.jpg)

So, a while ago, [I wrote about how I implemented my blog](https://brtheo.dev/blog/blogception) relying solely on the Github API to fetch articles and metadata associated with them, stored in a repo as `.md ` files. It was done in [Deno](https://deno.land) and the rendering was done through their framework, [fresh]([https.sh](https://fresh.deno.dev/)).
And then as a proof of concept I rewrote the whole thing using [Bun](https://bun.sh) in a ['server event driven'](https://github.com/brtheo/ServerWebComponent-htmx-lit-bun-elysia) way, by mixing server side rendering of `Lit` WebComponents and the use of `HTMX`.

But you know, as a developer, you're always eager to learn and make new things right ? 

So yeah, I had a pretty strong basis to itterate on, for this new `Go` version of my blog. 

The first goal was to port the Github API wrapper I wrote in `TypeScript` to `Go`, and that's on which topic this article will focus on (the whole templ + HTMX rendering part might come in the future).

My previous experiences with Go were *ok*, not great, just *comme ci comme ça*. 

The main reason I had this opinion on `Go` was surely a lack of knowledge about the language in general. 

After learning about its design patterns a bit more in depths and after having found an [iterator module](https://github.com/BooleanCat/go-functional), I can now confidently say that I'm having a blast with this language. And to be honest, it feels like a superior version of TypeScript honestly.

Some part of the API Wrapper code are almost exactly the same (thanks to the iter module) and the magic of TS custom types that you can replicate in Go.



As an example here's how is done the `getPostBySlug(slug)` method

In Typescript :
```typescript
public static async getPostBySlug(slug: string): Promise<IPost> {
    slug = `${slug}.md`
    const [commitForPost, contentForPost] = await Octoblog.getSeveral([
      {
        endpoint: ENDPOINTS.COMMIT,
        params: {slug}
      },
      {
        endpoint: ENDPOINTS.CONTENT,
        params: {slug,ref:Bun.env.GITBRANCH}
      },
    ]);
    return Octoblog.makePost((contentForPost as IContent), (commitForPost as Array<ICommit>), slug);
  }
```
And in Go:
```go
func (o *Octogo) GetPostBySlug(_slug string) *Post {
	slug := _slug + ".md"
	octoReqs := o.octoFetches(
		OctoReqs{
			One: &OctoRequest{
        verb:     "GET",
        endpoint: COMMIT_URL + _slug,
      },
			Two: &OctoRequest{
        verb:     "GET",
        endpoint: CONTENT_URL + _slug,
      },
		},
	)
	return o.newPost(Commits_B64{
		One: parseResponseInto[[]CommitResponse](octoReqs.One),
		Two: parseResponseInto[ContentResponse](octoReqs.Two).Content,
	}, slug)
}
```

We can see it's almost identical.

Same for when we have to map data, with the iter go module ...

```typescript
public static async getAllPosts(): Promise<IPost[]> {
    const treeElements: TreeElement[] = (
      await Octoblog.get(ENDPOINTS.TREE)
    ).tree.filter(treeElement => 
      treeElement.path.includes('.md')
    );
    const path_sha = treeElements.map(post => [
      post.path, 
      post.sha
    ]);
    return (
      await Promise.all(path_sha.map(tuple => 
        Octoblog.getPost(tuple)
      ))
    ).sort((postA, postB) => 
      postB.publishedAt - postA.publishedAt
    );
  }
```
```go
func (o *Octogo) GetAllPosts() []*Post {
	treeElements := iter.Lift[TreeElement](
		parseResponseInto[TreeResponse](
      o.octoFetch(
        NewOctoReq(
          Req().Endpoint(TREE_URL)
        )
      )
    ).Tree).Filter(func(treeElement TreeElement) bool {
		return strings.Contains(treeElement.Path, ".md")
	})

	slug_id := iter.Map[TreeElement, Slug_Id](
      treeElements, func(treeElement TreeElement) Slug_Id {
        return Slug_Id{
          One: treeElement.Path,
          Two: treeElement.Sha,
        }
	  },
    )

	iter.Map[Slug_Id, *Post](slug_id, o.getPost)

	p := iter.Map[Slug_Id, *Post](slug_id, o.getPost).Collect()
	By[Post](func(a, b *Post) bool {
		return a.PublishedAt > b.PublishedAt
	}).Sort(p)
	return p
}
```

One can argue that the Go version is slightly more verbose but that may be the cost of efficiency ?

So far I'm enjoying my time playing around with that iter module and trying out things with generics in Go.

