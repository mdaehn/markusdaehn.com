---
title: Auto Deploy Hexo Blog to Google Cloud Storage Using GitHub Actions and WIF
tags:
  - hexo
  - blog
  - auto deploy
  - google cloud storage
  - google cloud
  - github actions
  - github
  - workload identity federation
  - static site generator
  - github workflow
  - static site
date: 2023-10-28 13:59:05
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

First, lets set up Google Cloud Storage to host your static site. We are going to setup our site without HTTPS, using instructions provided by the guide, [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http); however, if you want to set it up with HTTPS, see [Host a static website](https://cloud.google.com/storage/docs/hosting-static-website). 


Below is the condensed version of [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http) guide with instruction specific to our needs. See the article mentioned for more details. 

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

In Visual Studio Code, edit your new post `source/_posts/Your-new-post-title.md`. I recommend adding a markdown extension (e.g [markdownlint](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint)) to VSC to help with writing posts. Once you are done, simply git add, commit and push it to GitHub:

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
 This will create the post file in the `source/_drafts` folder, making it not visible when your site is deployed until you publish it.

Once you ready to make it available to the world, run the `publish` command to move it to the `source/_posts`, and then git add, commit, and push to the main branch to make is visible on your GCS site:

```shell
hexo publish draft "Your new post title"
git add .
git commit -m "Published 'Your new post title'"
git push
```
Before we move on, I need to point out that Hexo needs at least one post to be visible to display the site correctly. Since we are deleting the default post, we need to insure there is at least one post that is published.

That said, there are more than one way to skin a cat. You could have utilized a branching strategy per post to control when it is visible, merging the post/feature branch into the main branch when you are ready to make it visible. I leave it to you to figure out the best strategy.

Also, you may want to update Hexo config to add your site information before you deploy it. See Hexo [config documentation](https://hexo.io/docs/configuration). There are also a variety of [themes](https://hexo.io/themes/) and [plugins](https://hexo.io/plugins/) to allow you to customize you site. 

## Setting up GCS WIF

We are going to use [Workload Identity Federation (WIF)](https://cloud.google.com/iam/docs/workload-identity-federation) to allow GitHub to deploy your Google Cloud Storage static site. This is a more secure and easy-to-manage alternative  to service account keys that utilizes short-lived access tokens. 

But, before we get started, I wanted to give credit to this great video, [How to use Github Actions with Google's Workload Identity Federation](https://www.youtube.com/watch?reload=9&app=desktop&v=ZgVhU5qvK1M). I adapted their instructions for Cloud Run to target Cloud Storage.

Now that is out of the way, you will first need to enable _IAM Service Account Credentials API_. In the [Google Cloud Console](https://console.cloud.google.com/) search box, type the following in the search field, "IAM Service Account Credentials API". In the dropdown list that appears, you should see "IAM Service Account Credentials API" in the **MARKETPLACE** section. Click it. It should take you to the _IAM Service Account Credentials API_ page. Click the **Enable** button to enable the API for your Google Cloud project.

Next, you need to configure the WIF. Enter "workload identity federation" in the Google Cloud search field and select the "Workload Identity Federation" in the **PRODUCTS & PAGES** section. Once on the page, you need to create an identity provider and pool. 

Click the big blue "Get Started" button. It should open the "New workload provider and pool" screen. Enter "GitHub Actions Cloud Storage" as the name and "github-actions-cloud-storage" as the description and click **Continue** button.

In the "Add a provider to pool"s step, select the "OpenID Connect (OIDC)" provider option, enter "github" in the _Provider name_ field,`https://token.actions.githubusercontent.com` _Issuer (URL)_ field  (See GitHub's [Configuring OpenID Connect in Google Cloud Platform](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform) for more detail) and click **Continue** button.

Next, we are going to map some provider attributes sent by Github to Google Clouds attribute in order to grant access to your GitHub repository. In "Configure provider attributes" step, the corresponding "Google 1 *" field should be set to "google.subject"; all you need to do is enter "assertion.sub" in the "OIDC 1 *" field. Now click the "+ ADD MAPPING" button, and enter  "attribute.actor" in both  "Google 2 *" and "OIDC 2 *" fields. Again, click "+ ADD MAPPING" button and enter "attribute.repository" in the "Google 3 *" and "OIDC 3 *" fields.

Now while still in the "Configure provider attributes" step, we need to add a CEL condition to restrict access to only our GitHub repository. Click the "ADD CONDITION" button and enter the following in the "Condition CEL field: `assertion.repository=='mdaehn/markusdaehn.com'`. Make sure you replace `mdaehn/markusdaehn.net` with your repository name, and click the "Save" button.
 

## Create Service Account

One more thing you need to do before your WIF setup is complete. You need to create a _Service Account_, which WIF will impersonate, to deploy your static site to the _Google Cloud Storage_ bucket.

While still on the "GitHub Actions Cloud Storage pool details" page, click the **`+ GRANT ACCESS`** and then click the **`CREATE A SERVICE ACCOUNT`** action in the drawer that opens. This will open a new "Create service account" window. 

In the "Service account details" section, enter "SA Blog Deployer" as the `Service account name`, or whatever makes sense to you. It should have filled in the `Service account ID *` with "sa-blog-deployer". Also add a description in the `Description` field, and then click **`CREATE AND CONTINUE`** button. 

In the "Grant this service account access to project" section, you need to select two roles: "Service Account Token Creator" and "Storage Object Admin". Do this now and click the **`CONTINUE`** button. 

This will take your "Grant users access to this service account" section. We are not going to add any users. Instead, lets go back to your "GitHub Actions Cloud Storage pool details" window and refresh the page, so our new service account will show up. Now click **`+ GRANT ACCESS`** button again. You should see your "SA Blog Deployer" in the `Service Account` dropdown. Select it and click `SAVE`. A "Configure you application" dialogue should appear, click the "DISMISS". 

## Setting Up GitHub Actions

Now you have your WIF set up; the last thing to do is setup your GitHub actions to trigger a deployment when changes are made to the main branch of your Blog's repository. 

First, Go to your GitHub project online and click the `Settings` tab. Select `Secret` > `Secrets and variables` > `Actions`, and add the following `Repository secrets`. These will be used in your GitHub Actions Workflow to deploy your site

- **GCS_BUCKET_NAME**: This is the Google Cloud Storage bucket name you created to host your static site. It should be something like this, "www.markusdaehn.com". If you are unsure, you can goto `Cloud Storage` > `Buckets`. You should see it in the list of buckets.
- **GCS_SA_EMAIL**: This is the email of your _Service Account_ connected to your WIF. Click the green `New repository secret` button, type in "GCS_SA_EMAIL" as `Name *`. Now Goto to `IAM & Admin` > `Service Accounts` and hover over your service account email in the list and click the copy icon that appears. Go back to the "New Secret" window and paste this as the value of `Secret *`
- GCS_WIF_PROVIDER: This is the WIF GitHub provider and needs to be in the format `project/<project ID>/locations/global/workloadIdentityPools/<workload identity pool name>/providers/<workload identity provider name>`. The easiest way to get this value is to goto the "Workload Identity Pools". Select `IAM & Admin` > `Workload Identity Federation` and select the pool we created. On the page, you will see the `Providers`, click the edit icon near the _github_ provider we created, and copy the URL in the `Default audience`, starting from projects to the end of the URL. Now, go back to the 

Once you got your GitHub secrets setup, open Visual Studio Code, and add a workflow file named `gcs-deploy.yml` in the `.github/workflows` directory of your blog project. Paste the following in it:

```yaml
name: Generate and Deploy blog to GCS
on: 
  push:
    branches: 
      - 'main'
jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository 
        uses: actions/checkout@v3

      - name: Setup NodeJs
        uses: actions/setup-node@v3
        with:
          node-version: 'latest'

      - name: Install Hexo
        run: npm install -g hexo-cli

      - name: Install dependencies
        run: npm install

      - name: Generate site
        run: hexo g

      - name: Upload generated site
        uses: actions/upload-artifact@v3
        with:
          name: generated-site
          path: ./public
  deploy:
    needs: generate
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - name: Download generated site
        uses: actions/download-artifact@v3
        with:
          name: generated-site
          path: ./public

      - name: Authorize GCS using WIF
        uses: 'google-github-actions/auth@v1'
        with:
          workload_identity_provider: '${{ secrets.GCS_WIF_PROVIDER }}'
          service_account: '${{ secrets.GCS_SA_EMAIL}}'

      - name: Upload files to GCS
        uses: 'google-github-actions/upload-cloud-storage@v1'
        with:
          path: './public'
          destination: '${{ secrets.GCS_BUCKET_NAME }}'
          process_gcloudignore: false
          parent: false

```

Commit and push your changes to your GitHub Repository.

If you go back to GitHub `Actions` online, you will see your workflow running to deploy your site. Wait a few minutes and if everything goes right, your site should be available to the world.
## Resources

- [How to use GitHub Actions with Google's Workload Identity Federation](https://www.youtube.com/watch?reload=9&app=desktop&v=ZgVhU5qvK1M)
- [Goodbye Service Account Keys, Hello Workload Identity Federation](https://www.youtube.com/watch?v=XcKT_0kFqBw)
- [Host a static website (using HTTPS)](https://cloud.google.com/storage/docs/hosting-static-website)
- [Hosting a static website using HTTP](https://cloud.google.com/storage/docs/hosting-static-website-http)
- [Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github#about-adding-existing-source-code-to-github)
- [Enabling GitHub Actions with Google Cloud Storage](https://docs.github.com/en/enterprise-server@3.10/admin/github-actions/enabling-github-actions-for-github-enterprise-server/enabling-github-actions-with-google-cloud-storage)
- [About Workflows](https://docs.github.com/en/actions/using-workflows/about-workflows)
[Enabling keyless authentication from GitHub Actions](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions)
  










