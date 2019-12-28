---
title: "Hugo, GitHub Pages, CircleCI and Cloudflare"
date: 2019-11-30T19:30:00Z
draft: false
categories: ["DevOps"]
---

This Blog is created and updated with the same processes outlined here. I will explain how you can create your own Blog using Hugo, Github Pages (Custom Domain) with CI/CD using CircleCI and fronted by Cloudflare.

Static websites have many advantages over their dynamic counterparts, to list a few:
 - More secure due to them being pure HTML/CSS, no dynamic rendering
 - Faster to load and cache with CDNs
 - Do not require a Server or Database
 - Can be hosted using S3 Buckets, Github/Gitlab Pages

# Step 1 - Creating Hugo Site/Files

This is a popular open-source static site generator. I like it due to the ability to locally test and validate what your site will look like before committing code.

There is a [quick start guide](https://gohugo.io/getting-started/quick-start/) on how to create a site using this framework. If you want to use my Blog as a reference, as it's based on Hugo you can find it here:
 - https://github.com/xrsanet/xrsanet.github.io/tree/source

 If you do use my code as a reference, ensure within the `config.toml` file that you change `baseurl` to your own domain, edit as you wish and remove everything in the `content` directory.

 Before moving onto the next section ensure you have the files ready to commit to a Github repo.

# Step 2 - Create GithHub Repo for GitHub Pages

The plan here is to keep the source code and gh-pages within a single repo, but separate them by branch. This is done by creating a source branch where the Hugo files go, leaving the master branch for the HTML pages, which CircleCI will generate and commit to.

Create a blank repo `<USERNAME>.github.io`. This must be your username, for example, mine is `xrsanet.github.io`.

Once that is done, you will want to switch to the source branch and then copy your Hugo files over.
```
#
# Example, Hugo files are in the hugo-site directory, yours will be different
#

$ git clone git@github.com:xrsanet/xrsanet.github.io.git
Cloning into 'xrsanet.github.io.git'...
warning: You appear to have cloned an empty repository.
$ cd xrsanet.github.io.git
$ git checkout -b source
Switched to a new branch 'source'
$ cp -rT ../hugo-site .
$ ls
archetypes  config.toml  content  README.md  resources  static  themes
$ git commit -m "Initial commit"
$ git push origin source
```
> Note: GitHub pages will still not work yet as we have not got CircleCI to generate the HTML pages from the Hugo files.

# Step 3 - CircleCI Configuration for CI/CD

**3.1 - Create a new key pair**

Create a new key pair:
```
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/home/blackhole/.ssh/id_rsa): circleci
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in circleci.
Your public key has been saved in circleci.pub.
The key fingerprint is:
...
$
```

**3.2 - Create CircleCI Account and add Github Repo**

Once you have created your account, add your .github.io project, then do the following:
 - Goto Project Settings
 - Under SSH Permissions, add the `circleci` Private Key you just generated, ensure you copy the Fingerprint as you will need this in a bit

**3.3 - Add Public Key to Github Repo**

Now go to your Github project settings, under Deploy Key, add the `circleci.pub` Public Key you just generated.

**3.4 - Create CircleCI Configuration**

Create a CircleCI configuration file within the same source branch as before:
```
$ mkdir .circleci
$ cat > .circleci/config.yml << EOF
version: 2.0

jobs:
    build:
        docker:
            - image: cibuilds/hugo:0.60
        working_directory: ~/<USERNAME>.github.io
        steps:
            - add_ssh_keys:
                fingerprints:
                    - "<FINGERPRINT>"
            - checkout
            - run:
                name: Get current site
                working_directory: ~/
                command: git clone -b master git@github.com:<USERNAME>/<USERNAME>.github.io.git public
            - run:
                name: Generate site
                working_directory: ~/<USERNAME>.github.io
                command: HUGO_ENV=production hugo -d ~/public
            - deploy:
                name: Deploy to Github Pages
                working_directory: ~/public
                command: |
                    git config credential.helper 'cache --timeout=120'
                    git config user.email "<EMAIL>"
                    git config user.name "CircleCI Deployment"
                    git add .
                    git commit --allow-empty -m "CircleCI Deployment - [ci skip]"
                    git push -q git@github.com:<USERNAME>/<USERNAME>.github.io.git master

workflows:
  version: 2
  main:
    jobs:
    - build:
        filters:
          branches:
            only: source

EOF
```
> Note: Change `<USERNAME>`, `<EMAIL>` to your respective values. Also, note `[ci skip]`, will ensure that CircleCI skips the master branch when building/pushing the HTML files and will only build from the source branch.

**3.5 - Commit CircleCI Configuration**

Once you have made all the required changes to `.circleci/conig.yml` commit the configuration to the source branch.

This will trigger CircleCI to build and commit your site under: `https://<USERNAME>.github.io`. If you get failures, check what errors your Pipeline is getting, could be Hugo build errors or permission errors when deploying.

# Step 4 - Configure Custom Domain and Cloudflare

**4.1 - GitHub Pages Custom Domain**

Getting GitHub Pages working with a custom domain is simple, all you do is create a `CNAME` file in the `master` branch. This can only be one domain on a single line, ensure you do not include `www` as we tackle that with Cloudflare. For example, mine is `xrsa.net`.

**4.2 - Cloudflare DNS**

Cloudflare requires you to move your Nameservers over but once done all you do is the following to get it all working as expected.

![cloudflare_xrsa_dns](/images/cloudflare_xrsa_dns.png)

**4.3 - Cloudflare SSL/TLS**

Ensure in the SSL/TLS section that you configure mode `Full` and it will work with both `www.<DOMAIN>` and `<DOMAIN>`.

# Final Notes

That is it, hopefully, this has been helpful. You should now have a fully working blog which is free, fast, secure and easy to update due CI/CD.