---
title: Auto Deploy Hexo Blog to Google Cloud Storage Using GitHub Actions
tags: ["hexo", "blog", "autodeploy", "google cloud storage", "google cloud", "github actions", "github", "workload identity federation", "static site generator"]
---

## Introduction

Mainstream blogging platforms, such as WordPress, are bloated and overly complicated. Most of the time, you want a simple solution without all the fluff. This article describes how to setup a no fluff solution that will appeal to software developers. Even if you are not a developer, you may still enjoy this simple approach.

## Overview 

We use Hexo as our blogging engine, Visual Studio Code to edit our blog posts, Google Cloud Storage (GCS) to host our blog site and Github as our version control and backup. We will also take advantage of GitHub actions and Google Cloud's Workload Identity Federation (WIF) to auto generate and deploy our static site whenever a change is made to the main git repository branch. This setup may seem eerily familiar to developers, since it utilizes similar tools and methods in a typical development process.


## Prerequisites

There are a few prerequisites before we get started. Make sure these are installed and setup on your machine.

* [Node](https://nodejs.org/en/download)
* [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [Visual Studio Code](https://code.visualstudio.com/download) (Or IDE of choice)

This article should help you setup Git and Node, "[Setting up your Development Environment: Git and Node](https://olanetsoft.medium.com/setting-up-your-development-environment-git-and-node-948613b1239a)". Hexo also has instructions on its [site](https://hexo.io/docs/#Requirements).

You will need a [GitHub](https://github.com/) account if you don't already have one and a SSH key added to your account. See this [article](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) for more details.

## Setting up Static Site on Google Cloud Storage
First, lets set up Google Cloud Storage to host our static site. Use the [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http) guide if you want to setup a site with HTTP. If you want to set it up with HTTPS, see [Host a static website](https://cloud.google.com/storage/docs/hosting-static-website). 


I am going to summarize the [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http) guide. We can later change it to HTTPS, but I want to keep it simple.

1.  Select or Create Google Cloud Project. [Go to Project Selector](https://console.cloud.google.com/projectselector2/home/dashboard?_ga=2.110679239.399703744.1696799026-977856356.1696799026)

2. Make sure billing is enabled for your project. See [Verify the billing status of your projects](https://cloud.google.com/billing/docs/how-to/verify-billing-enabled#console)

3. Goto [Search Console](https://search.google.com/search-console/welcome) and enter top-level domain you plan to use (e.g. markusdaehn.net and not www.markusdaehn.net) and click verify. You need to verify you own the domain. See [Domain ownership verification](https://cloud.google.com/storage/docs/domain-name-verification#verification)

4. Goto your domain name provider (e.g https://domains.google.com/registrar/) and add a `CNAME` resource record for you domain (e.g. markusdaehn.net)

    ```
    NAME                  TYPE     DATA
    www                   CNAME    c.storage.googleapis.com.
    ```

5. Goto [Cloud Storage Buckets page](https://console.cloud.google.com/storage/browser?_ga=2.149114422.1663787659.1696908888-1096239556.1690143603) and create a bucket name with www subdomain (e.g. www.markusdaehn.net) and accept all the defaults. When you click create,  uncheck "Enforce public access prevention on this bucket". 

6. Now we need to make our bucket publicly accessible. While in the Bucket details, select the `Permissions` tab and click the **GRANT ACCESS** button.  In the dialogue that appears, add `allUsers` in the new the **New Principals**, select "Cloud Storage" > "Storage Object Viewer" role, and save. You should get confirmation warning; click **ALLOW PUBLIC ACCESS**.  




## Setting up Workload Identity Federation

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



## Resources

- [How to use Github Actions with Google's Workload Identity Federation](https://www.youtube.com/watch?reload=9&app=desktop&v=ZgVhU5qvK1M)
- [Host a static website (using HTTPS)](https://cloud.google.com/storage/docs/hosting-static-website)
- [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http)

  










