---
title: Auto Deploy Hexo Blog to Google Cloud Storage Using GitHub Actions
tags: ["hexo", "blog", "autodeploy", "google cloud storage", "google cloud", "github actions", "github", "workload identity federation", "static site generator"]
---

## Overview 

When starting a blog, you have a plethora of choices; unfortunately, many of the mainstream blogging platforms, such as WordPress, are overly bloated. As a software developer, you don't need all the fluff and would prefer a simple solution that mimics how you write code, from the tooling to the workflow. 

If that is you, then you are in the right place. We are going to setup a static blog site using Hexo, use Google Cloud Storage to host it and GitHub as our version control system. We plan to take advantage of GitHub actions and GCS Workload Identity Federation (WIF) to automatically generate our static site and deploy it to GCS.  

As a side note, I chose Hexo because it is developed using nodejs, which is in my wheelhouse, and for its simplicity. It has basic static site generator and plugin architecture. It also has a nice command line interface that allows automation. Hexo's post are simple markdown files, so it allows us to write our blog posts straight from our IDE of choice. In my case, it is Visual Studio Code. 

That said, you don't necessarily have to use Hexo. You can swap it out for any of the many other blog frameworks that support static site generation (e.g. Jekyll), or with any framework, for that mater, that supports static site generation, such as next.js. Similarly, you don't have to use Visual Studio code, and can use what ever you feel comfortable using to edit your files.

## Prerequisites

There are a few prerequisites before we get started. Make sure these are installed and setup on your machine.

* [Node](https://nodejs.org/en/download)
* [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [Visual Studio Code](https://code.visualstudio.com/download) (Or IDE of choice)

This article should help you setup Git and Node, "[Setting up your Development Environment: Git and Node](https://olanetsoft.medium.com/setting-up-your-development-environment-git-and-node-948613b1239a)". Hexo also has instructions on its [site](https://hexo.io/docs/#Requirements).

You will need a [GitHub](https://github.com/) account if you don't already have one and a SSH key added to your account. See this [article](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) for more details.

## Setting up Google Cloud Static Site

## Setting Up Your Hexo Blog Site

Now, lets get to it. Install Hexo command line interface, hexo-cli:

```bash
$ npm install -g hexo-cli
```

Lets create a hexo site by running the hexo init command. I recommend naming it using your domain name. For example, my blog site's domain is markusdaehn.net, so I would run the following command. Hereafter, replace markusdaehn.com with your domain.

```bash
$ hexo init markusdaehn.net
```

Now change into the directory Hexo generated and run it locally. 

```bash
$ cd markusdaehn.net
$ hexo server
```

You should be able to view your locally running site in the browser with the following URL: http://localhost:4000/.

As you can see, there is a "Hello world" blog post already setup for us. We are going to keep it for now, since we are only concerned about the process of generating the static site in this article; however, if you want to delete and replace it with a legit post, simply delete the `source/_posts/hello-world.md` file and run the following commands to clean up and create a new post. 

```bash
$ hexo clean
$ hexo new post "Your new post title"
```

You might also want to update the config with your site information. See Hexo [config documentations](https://hexo.io/docs/configuration). 

  










