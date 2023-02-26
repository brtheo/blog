---toml
tags = ['deno','github','api','typescript']
---
There's several ways of creating a blog nowadays, whether it is with a legacy CMS such as [Wordpress](https://wordpress.com) whether it is by taking a more modern approach using a headless CMS solution like [Strapi](https://strapi.io) or even building it with [Astro](https://astro.build) through their brand new feature [Content Collections](https://docs.astro.build/en/guides/content-collections/).

Bref, there're plenty of solutions.

## Why not build it ourselves ?
I know right, I had the same idea 🤓

I wanted a solution that would be CMS-like, with automatic time tracking of blog post creation/edition possibility of several authors and easy content management.

When describing like this, it does sound familliar right ? <br>
*Excatly*, I'm basically describing **Git** and **Github**.

So, now that's it's clearer in our head let's summarize what we are going to do :
- Use Deno + Fresh
- Use the Github API
- Implements type safety