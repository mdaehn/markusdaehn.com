---
title: Auto Deploy Hexo Blog to Google Cloud Storage Using GitHub Actions
tags: ["hexo", "blog", "autodeploy", "google cloud storage", "google cloud", "github actions", "github", "workload identity federation", "static site generator"]
---

## Introduction

Mainstream blogging platforms (e.g. WordPress) are overly bloated. Most of the time, you want a simple solution without all the fluff. This article describes how to setup a no fluff solution that will appeal to developers. Even if you are not a developer, you may still enjoy this simple approach.

## Overview 

We use Hexo as our blogging engine, Google Cloud Storage (GCS) to host our blog sit and Github as our version control and backup. We also take advantage of GitHub actions and Google Cloud's Workload Identity Federation (WIF) to auto generate and deploy our static site when the main branch is modified.

If you are a developer, this process sounds very familiar. The only thing missing is writing our blog post in an IDE like Visual Studio Code, and that is exactly what we plan to do. 

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

  










