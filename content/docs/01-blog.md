---
date: '2025-02-09T21:34:40+01:00'
draft: false
title: '01 Blog'
ShowToc: true
---

In this post, I explain how I set up this very blog using [Hugo](https://gohugo.io/), [GitHub Actions](https://github.com/features/actions) and [GitHub Pages](https://pages.github.com/). I've wanted to create my own blog for documenting some of my projects for a while, but never liked using tools like Wordpress or similar CMS. Web editors or proprietary formatting tools do not appeal to me.

The aforementioned tools fit my requirements perfectly, in that they allow me to write posts in markdown (a language I've used for documenting stuff for about a decade), track changes using git, customize my blog if I ever want to and publish it with a simple push to the respective GitHub repository.

## Installing Hugo

Start by installing Hugo. Instructions for most operating systems can be found on the [project's website](https://gohugo.io/installation/). I'll use Ubuntu 24.10. running in WSL2.

While there exist packages for Hugo in most distribution's default repositories, it is recommended to install it using [Snap](https://en.wikipedia.org/wiki/Snap_(software)). The packages distributed via Snap are generally more recent than those in the default repositories.

```sh
sudo apt update -y && sudo apt dist-upgrade -y
sudo apt install snapd
sudo snap install hugo
```

## Creating the Workspace

Once Hugo is installed, it is time to create the site's workspace. The workspace will contain everything necessary to build the blog, including the posts themselves, assets like embedded images, themeing etc.

We create the workspace using Hugo's CLI and turn it into a git repository for version control afterwards.

```sh
hugo new site ./blog
cd ./blog
git init
```

We'll also add a basic `.gitignore` to make sure no undesired files make it into our git history in the future. The directory `public` will contain the static files generated when running the site locally later on. We do not track these files using git, because GitHub Actions will take care of building them for deployment. Hugo creates and manages `.hugo_build.lock` to prevent conflicts between concurrent hugo instances.

```txt
# /.gitignore
public
.hugo_build.lock
```

Commit the current state to have a clean starting ground.

```sh
git add -A
git commit -m "chore(hugo): initialize Hugo repository"
```

## Adding a Theme

If you want a fully custom design for your blog, you can create a theme from scratch. However, this is an advanced topic I am not going to touch for now. I am content with using one of the amazing themes provided by other creators in [Hugo's theme library](https://themes.gohugo.io/). I recommend taking your time browsing through all of the options to find something you like. I decided to start out using the popular theme [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/) by [Aditya Telange](https://github.com/adityatelange/).

Hugo themes are generally distributed via git repositories and included into your workspace as a submodule.

```sh
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive
```

Add the following line to your project's `hugo.yaml` to tell Hugo which theme to use when building the static website:

```yaml
# /hugo.yaml
...
theme: ["PaperMod"]
...
```

Commit the changes.

```sh
git add -A
git commit -m "chore(hugo): add PaperMod theme"
```

## Adding Posts

While you can create every content file by scratch, it is recommended to use so called [archetypes](https://gohugo.io/content-management/archetypes/) instead. Simply put, an archetype is a template to be used for content files to avoid missing important details.

Your project should contain the default archetype `/archetypes/default.md`. It contains the most basic [front matter](https://gohugo.io/quick-reference/glossary/#front-matter) header needed for Hugo to process a content file. If you want all of your posts to contain common headers, footers or other traits, you may extend this archetype to your liking or add entirely new archetypes. I'll stick to the default for now.

```sh
hugo new docs/01-blog.md
```

The created file will look a little something like this:

```yaml
# /content/docs/01-blog.md
---
date: '2025-02-09T21:34:40+01:00'
draft: true
title: '01 Blog'
---

```

You may now add content to your first post. While markdown files are usually supposed to start with a level 1 heading, Hugo will use the value of the field `title` for the page's heading instead. Thus, you should start with level two headings in the actual content.

```yaml
# /content/docs/01-blog.md
---
date: '2025-02-09T21:34:40+01:00'
draft: true
title: '01 Blog'
---

Introduction

## Chapter 1

### Chapter 1.1

## Chapter 2

```

If you want to test what your blog looks like prior to publishing it online, you may run the hugo server from the workspace's root directory. The parameter `--buildDrafts` is required to include posts that are set to `draft: true`. Once the server is running, you can reach it via your browser at `localhost:1313`.

```sh
hugo server --buildDrafts
```

Remember to set `draft` to `false`, when you are happy with your post and want it to be included on GitHub Pages. Otherwise the GitHub Actions workflow will ignore it.

Lastly, create a new commit containing the n

## Hugo Base URL

Hugo needs to be aware of your blog's base url to generate static hyperlinks for your blog. GitHub will use the domain `<username>.github.io/<repository>` by default. I'll start by using this domain and add a custom domain later on.

In `hugo.yaml`, set `baseURL`:

```yaml
# /hugo.yaml
...
baseURL: https://<username>.github.io/<repository>/
...
```

Commit the changes.

```sh
git add -A
git commit -m "chore(hugo): set baseURL"
```

## GitHub Repository

Login to [GitHub](https://github.com/) and click on your avatar at the top right. Select `Your repositories`. At the top right, click on `New` and create a new repository for your blog. I chose the simple name "blog" for my repository.

Next add the GitHub repository as a remote to your local repository. Use either the web or SSH URL, [depending on which method of authentication you use](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github).

```sh
git remote add origin <web/SSH URL>
```

You can now push the contents from your local repository to the repository in GitHub.

```sh
git push -u origin main
```

## GitHub Actions

You can find detailled instructions for configuring GitHub Actions in [Hugo's documentation on Histing on GitHub Pages](https://gohugo.io/hosting-and-deployment/hosting-on-github/). I'll paraphrase the essentials here.

Within GitHub, navigate to your repository and go to *Settings* / *Pages*. Under *Build and deployment*, select `GitHub Actions`.

On the same page, you should also check *Enforce HTTPS*. GitHub will create a TLS certificate for all domains automatically. However, without this checkbox being checked, visitors may be able to visit your page using unencrypted HTTP regardless. By enabling this setting, GitHub will force all visitors to use HTTPS via redirects.

In your local workspace, create the file `/.github/workflows/pages.yaml` and add the following content. The example in Hugo's documentation contains some additional steps for more advanced configurations we do not need yet.

```yaml
# /.github/workflows/pages.yaml
name: Deploy Hugo site to Pages

# Run whenever changes are committed to the branch main.
on:
  push:
    branches:
      - main
  workflow_dispatch:

# Allow the workflow to deploy to GitHub Pages.
permissions:
  contents: read
  pages: write
  id-token: write

# Prevent concurrent runs of the pipeline in case lots of changes are pushed in rapid succession.
# Skip all but the latest run, but do not cancel running jobs.
concurrency:
  group: "pages"
  cancel-in-progress: false

# Use bash for all shell jobs.
defaults:
  run:
    shell: bash

jobs:

  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.143.1
    steps:

      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
          TZ: Europe/Berlin
        run: |
          hugo \
            --gc \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Upload artifact to Pages
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Commit and push.

```sh
git add -A
git commit -m "cicd(hugo): add GitHub Pages deployment"
git push -u origin main
```

You may now go to your repository in GitHub and check the deployment's progress under the *Actions* tab. You should see a new workflow run. Once it as concluded (and if everything is set up correctly), you can visit your blog under the aforementioned base URL.

## Claiming your Custom Domain

First, you should start by claiming ownership of your domain. This prevents others from using your domain on GitHub in the event of your DNS provider having been compromised. Refer to <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages> for more details.

1. Within GitHub, click on your avatar and go to *Settings* / *Pages*.
2. Click on `Add a domain`, enter your domain and confirm by clicking on `Add domain`.
3. You'll be presented with the name and value for a `TXT` record to be configured in your nameserver. You being able to create this record in an authoritative nameserver proves your control of the domain to GitHub.
4. Once you have added the requested record to your DNS, click on `Verify`. It should be done within a few minutes, but may take up to 24h in rare cases.

## Using the Custom Domain for GitHub Pages

quite verbose: <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site>

Start by going to your nameserver and creating a `CNAME` record that points to `<username>.github.io`. GitHub also supports using `A`, `AAAA` or `ALIAS` records. However, using a CNAME record is recommended, because it permits GitHub to change the servers they use for hosting pages on the fly.

Update your `hugo.yaml` to make Hugo use the new URL when generating links. In `hugo.yaml`, set `baseURL` to your custom domain:

```yaml
# /hugo.yaml
...
baseURL: https://<custom domain>/
...
```

Commit and push.

```sh
git add -A
git commit -m "cicd(hugo): use custom domain"
git push -u origin main
```

Once the respective workflow run is complete, you'll be able to visit your blog using your custom domain.

## References

- <https://www.learnwithghaniy.web.id/en/posts/installing-hugo-on-ubuntu-22.04/>
- <https://www.youtube.com/watch?v=_QSr2_pxIJs>
- <https://theplaybook.dev/docs/deploy-hugo-to-github-pages/>
- <https://gohugo.io/hosting-and-deployment/hosting-on-github/>
- <https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https>
- <https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site>
- <https://www.learnwithghaniy.web.id/en/posts/installing-hugo-on-ubuntu-22.04/>
