---
date: '2025-02-09T21:34:40+01:00'
draft: false
title: '01 Blog'
---
# Blog

## Workflow

- create branch `gh-pages`

- mkdir -p .github/workflows

```yaml
# .github/workflows/deploy.yml

```

## Hugo Site

```sh
sudo apt update -y && sudo apt dist-upgrade -y
sudo apt install snapd
sudo snap install hugo
```

```sh
hugo new site ./blog
cd ./blog
```

```yaml
# /hugo.yaml
baseURL: http://blog.cyclonit.io/
languageCode: en-us
title: Cyclonit's Blog
theme: ["PaperMod"]
```

```sh
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive
git submodule update --remote --merge
```

```txt
# .gitignore
public
```

```sh
git add -A
git commit -m "chore(hugo): initialize Hugo repository"
```

## Adding Posts

```sh
hugo new docs/01-blog.md
```

```yaml
# /content/docs/01-blog.md
---
date: '2025-02-09T21:34:40+01:00'
draft: false
title: '01 Blog'
---
# Blog
...
```

```sh
git add -A
git commit -m "feature(blog): add post 01-blog.md"
```

## GitHub Repository

Create repository within GitHub.

```sh
git remote add origin https://github.com/Cyclonit/blog.git
```

```sh
git push -u origin main
```

## Deploying to GitHub Pages

Settings > Pages > Build and deployment / Source / GitHub Actions

## Sources

- <https://www.learnwithghaniy.web.id/en/posts/installing-hugo-on-ubuntu-22.04/>
- <https://www.youtube.com/watch?v=_QSr2_pxIJs>
- <https://theplaybook.dev/docs/deploy-hugo-to-github-pages/>
- <https://gohugo.io/hosting-and-deployment/hosting-on-github/>
