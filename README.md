# joekbullard.github.io
My personal blog - uses [hugo](https://github.com/gohugoio/hugo) framework and GitHub Actions for build and deployment. Visible at [https://joekbullard.xyz/](https://joekbullard.xyz/). Hugo was chosen as a great framework for quick static page generation with templates.
![Screenshot of blog landing page](https://github.com/joekbullard/joekbullard.github.io/assets/35529853/5acafad0-0754-4e15-a916-009b46f9a6d8)
![Screenshot of blog post](https://github.com/joekbullard/joekbullard.github.io/assets/35529853/49472067-ca52-4138-add2-06e48a873080)

A GitHub Action triggers deployment when updates are pushed to the repository - see [here](https://github.com/joekbullard/joekbullard.github.io/blob/main/.github/workflows/gh-pages.yml) for the config

This blog replaces the previous project I built [here](https://github.com/joekbullard/myblog) using Django and docker containers, I opted for a static blog as there is less upkeep.

## How to install and run the project
Clone the repo, create a post using markdown then build from the project root using:
`$ hugo server --buildDrafts`
This should create a templated version of your post.

You will need to configure GitHub Actions accordingly, for more information see [here](https://gohugo.io/hosting-and-deployment/hosting-on-github/).

