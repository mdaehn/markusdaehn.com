---
title: Auto Deploy Hexo Blog to Google Cloud Storage Using GitHub Actions
tags: ["hexo", "blog", "autodeploy", "google cloud storage", "google cloud", "github actions", "github", "workload identity federation", "static site generator"]
---

## Introduction

Mainstream blogging platforms, such as WordPress, are bloated and overly complicated. Most of the time, you want a simple solution without all the fluff. This article describes how to setup a no fluff solution that will appeal to most software developers. Even if you are not a software developer, you may still find this approach useful.
## Overview 

This solution includes Hexo as your blog engine, Visual Studio Code as your post editor, Google Cloud Storage (GCS) as your site's host and GitHub as your version control and backup. You will also take advantage of GitHub actions and Google Cloud's Workload Identity Federation (WIF) to auto generate and deploy your static site when changes are added (merged or pushed) to your main branch.

If you are a software developer, these tools may sound familiar. Many developers use these as part of their development life cycle, which makes writing and maintaining your blog as easy as writing and maintaining your code.

## Note

I will be using my domain name, markusdaehn.com, in the examples. Replace them with your domain name.

## Prerequisites

There are a few prerequisites before you get started. Make sure these are installed and setup on your machine.

* [Node](https://nodejs.org/en/download)
* [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [Visual Studio Code](https://code.visualstudio.com/download) (Or IDE of choice)

This article should help you setup Git and Node, "[Setting up your Development Environment: Git and Node](https://olanetsoft.medium.com/setting-up-your-development-environment-git-and-node-948613b1239a)". Hexo also has instructions on its [site](https://hexo.io/docs/#Requirements).

You will need a [GitHub](https://github.com/) account if you don't already have one and a SSH key added to it. See this [article](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) for more details.

## Setting up Static Site on Google Cloud Storage
First, lets set up Google Cloud Storage to host your static site. Use the [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http) guide if you want to setup a site with HTTP. If you want to set it up with HTTPS, see [Host a static website](https://cloud.google.com/storage/docs/hosting-static-website). 


I am going to summarize the [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http) guide. You can later change it to HTTPS, but I want to keep it simple.

1.  Select or Create Google Cloud Project. [Go to Project Selector](https://console.cloud.google.com/projectselector2/home/dashboard?_ga=2.110679239.399703744.1696799026-977856356.1696799026)

2. Make sure billing is enabled for your project. See [Verify the billing status of your projects](https://cloud.google.com/billing/docs/how-to/verify-billing-enabled#console)

3. Goto [Search Console](https://search.google.com/search-console/welcome) and enter top-level domain you plan to use (e.g. markusdaehn.com and not www.markusdaehn.com) and click verify. You need to verify you own the domain. See [Domain ownership verification](https://cloud.google.com/storage/docs/domain-name-verification#verification)

4. Goto your domain name provider (e.g https://domains.google.com/registrar/) and add a `CNAME` resource record for you domain (e.g. markusdaehn.com)

    ```
    NAME                  TYPE     DATA
    www                   CNAME    c.storage.googleapis.com.
    ```

5. Goto [Cloud Storage Buckets page](https://console.cloud.google.com/storage/browser?_ga=2.149114422.1663787659.1696908888-1096239556.1690143603) and create a bucket name with www subdomain (e.g. www.markusdaehn.com) and accept all the defaults, except uncheck "Enforce public access prevention on this bucket" when you get to step "Choose how to control access to objects". Click create.

6. Now you need to make your bucket publicly accessible. While in the Bucket details, select the `Permissions` tab and click the **GRANT ACCESS** button.  In the dialogue that appears, add `allUsers` in the new the **New Principals**, select "Cloud Storage" > "Storage Object Viewer" role, and save. You should get confirmation warning; click **ALLOW PUBLIC ACCESS**.  


7. Next, lets make sure website index file is configured. Got the [Buckets page](https://console.cloud.google.com/storage/browser?_ga=2.236843201.151245491.1697158473-929429999.1690142359). Click the vertical 3 dots to the right of your site bucket and click **Edit Website Configuration** and enter "index.html" in the "Index (main) page suffix" and click save.

7. At this point, you are ready to setup GitHub to deploy your site; however, if you want to test out your site setup, you can upload an test index.html file. On the Bucket details page. There should be a **Upload FILES** button. Create a "Hello World" index.html file like the example below and upload it.

    ```html
    <!DOCTYPE html>
    <html>
        <head>
            <title>Hello World</title>
        </head>
        <body>
            <h1>Hello World!</h1>
        </body> 
    </html>
    ```
    Now, if you visit your site in your browser using your domain that you setup (e.g. http://www.markusdaehn.com), it should display "Hello World!". 

## Creating your Hexo Blog Site

Now, lets get into the meat of it and create your blog site. To do this, install Hexo command line interface, hexo-cli:

```shell
npm install -g hexo-cli
```

To create your blog site, run the hexo init command. You will be using your domain name (e.g. markusdaehn.com) as the blog's project name and subsequently as your GitHub Repository name. This is not strictly required but keeps things consistent.

```shell
hexo init markusdaehn.com
```

Make sure you replace `markusdaehn.com` with your domain name. 

Now change into your projects directory and run it locally. 

```shell
cd markusdaehn.com
hexo server
```

You should be able to view your locally running blog site in the browser with the following URL: http://localhost:4000/.

As you can see, there is a "Hello world" blog post already in your blog project. You are going to keep if for now, but you will be replacing it later with a legitimate post.

## Create and Import Your GitHub Repository

Next, you need to create your GitHub repository. I assume you already have a GitHub account with SSH key setup. If not, then do this now. See [Adding a new SSH key to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) if you need to add a SSH key or [Generating a new SSH key and adding it to the ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) if you need to also generate one before you add it. 

At this point, you can open your project in Visual Studio Code (VSC), or your preferred editor. In VSC, right-click the Explorer window, and then select the "Add Folder to Workspace..." option. Locate your project folder and choose it. 

Now, open Visual Studio Code's integrated terminal, **Terminal > New Terminal**, and choose your project. In the terminal, run the following commands to initialize, add, and commit your git repository (This assumes Git 2.28.0 or later):

```shell
git init -b main
git add .
git commit -m "Initial Release"
```
Going forward, most actions described in this article can be done through VSC.

So far, we have our blog project in Git locally but we need to add our local repository to GitHub. To do this, goto to [GitHub.com](https://github.com), login, and [create a new repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository). Name it with your domain name (e.g. `markusdaehn.com`), and don't initialize it with README, license, or gitignore files. Also, depending on your preference, select private or public access. 

After you create your blog GitHub Repository, it should have opened it with some instruction on setting up your repository. In the "Quick Setup", copy the SSH Git remote URI (e.g git@github.com:mdaehn/markusdaehn.net.git), and go back to your VSC integrated terminal that is in your current local project and enter the following command with the Git Remote URI you copied: 

```shell
git remote add origin REPLACE-WITH-COPIED-GIT-REMOTE-URI
```

And run the following to push it to the remote GitHub repository

```shell
git push -u origin main
```

If everything went as planned, your local main branch is setup to track the remote main branch, and if you go back to your GitHub repository in the browser and refresh, you should see that is updated with your Hexo blog source code. Sweet!

## Creating Your First Post

If you are itching to create a post, or just do not want to deploy your blog site with the default "Hello World" post, then this is the time to make some changes.

To remove the "Hello World" post, delete the `source/_posts/hello-world.md` file in your project and run the following commands (Replace "Your new post title" with your title):

```shell
hexo clean
hexo new post "Your new post title"
```

This will clean up hexo blog and create a new post named `Your-new-post-title.md` in the `source/_posts` directory. 

In Visual Studio Code, edit your new post `source/_posts/Your-new-post-title.md`. I recommend adding a markdown extension (e.g [markdownlint](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint)) to VSC to help with writing post. Once you are done, simply git add, commit and push it to GitHub:

```shell
git add .
git commit -m "Published post 'Your new post title'"
git push
```

Since you made the changes to the main branch, it would have deployed to your Google Cloud Storage static site if auto deployment was set up. 

However, if you want to merge in changes to the main branch, but do not want to make it visible on your site, you can use `new` command with the `draft` layout instead of `post`.

```shell
hexo new draft "Your new post title"
```
 This will create the post file in the `source/_drafts` folder, making it not visible until you publish it. 

And once you ready to make it available to the world, run the `publish` command to move it to the `source/_posts`, and then git add, commit, and push to the main branch:

```shell
hexo publish draft "Your new post title"
git add .
git commit -m "Published 'Your new post title'"
git push
```
Before we move on, I need to point out that Hexo needs at least one post to be visible to display the site correctly.

That said, there are more than one way to skin a cat. You could have utilized a branching strategy per post to control when it is visible, merging the post/feature branch into the main branch when you are ready to make it visible. You can figure out what best suits your needs.

Also, you may want to update Hexo config to add your site information before you deploy it. See Hexo [config documentation](https://hexo.io/docs/configuration). 

## Resources

- [How to use GitHub Actions with Google's Workload Identity Federation](https://www.youtube.com/watch?reload=9&app=desktop&v=ZgVhU5qvK1M)
- [Host a static website (using HTTPS)](https://cloud.google.com/storage/docs/hosting-static-website)
- [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http)
- [Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github#about-adding-existing-source-code-to-github)

  










