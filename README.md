# kerrywalker.uk

Source for [kerrywalker.uk](https://www.kerrywalker.uk), the blog and web CV of Kerry Walker, a senior C#/.NET developer and team lead in Chester.

Most of the writing is about .NET, C# and SQL Server, aimed at developers who want the reasoning rather than the recipe. The rest is home automation and energy, and the occasional bike.

- [Blog](https://www.kerrywalker.uk/) and [feed](https://www.kerrywalker.uk/feed.xml)
- [CV](https://www.kerrywalker.uk/cv.html)
- [Home automation and energy](https://www.kerrywalker.uk/home-automation.html), a self-updating index
- [All tags](https://www.kerrywalker.uk/tags.html)

A few .NET posts to start with:

- [Lists, LINQ Where and Dictionaries: .NET performance in plain English](https://www.kerrywalker.uk/2026/07/17/lists-linq-where-and-dictionaries.html)
- [Async/await does not create new threads](https://www.kerrywalker.uk/2026/07/20/async-await-does-not-create-new-threads.html)
- [You're already using design patterns, you just don't know their names](https://www.kerrywalker.uk/2026/09/09/youre-already-using-design-patterns.html)
- [One screen, many searches: Strategy and keyed DI](https://www.kerrywalker.uk/2026/09/14/strategy-and-keyed-dependency-injection.html)
- [The SQL Server gotcha where LEN and RIGHT disagree](https://www.kerrywalker.uk/2026/08/24/sql-server-len-trailing-spaces-gotcha.html)

## How it is built

Jekyll on GitHub Pages, using the [Hydeout](https://github.com/fongandrew/hydeout) theme (MIT; the Hyde and Hydeout notices are in `LICENSE.md`). Posts live in `_posts/`, the CV source in `_cv/web-cv.md` with `cv.html` as its page. The theme's sample posts are kept in the repo with `published: false` so the layouts can be checked locally.

Comments run on Giscus over GitHub Discussions; analytics are Cloudflare Web Analytics. Both are described in posts on the site.
