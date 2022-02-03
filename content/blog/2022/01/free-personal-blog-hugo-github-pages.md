---
title: "Free Blog Hosting Using Hugo + Github Pages"
description: "Upgrade your Personal Blog Game with Apex Records for No Cost"
date: 2022-01-27T11:51:20-07:00
images:
- src: img/2022/01/github-pages-logo.png
  alt: "Github Pages Logo"
  stretch: "none"
---

Since I just rode this merry-go-round here is a quick overview of how you can achieve a professional
looking personal blog with custom DNS with minimal effort at no cost and completely run out of a single Github Repository.

### Prerequisites
You'll need to do your own homework on how to use [Hugo](https://gohugo.io/) and you should have a [Github](https://github.com/) 
account and have a decent understanding of using git and getting changes committed and pushed to a repo. You should also
have a domain and know how to do some basic DNS changes. Note that the only thing you likely can't get for free is a domain name as 
there are registration costs through registrars but, you can get introductory $0.99USD domains for the first year if you shop around and don't care the [TLD](https://en.wikipedia.org/wiki/Top-level_domain)
you end up with. For reference, luth.io cost me $30/year to register with [CloudFlare](https://www.cloudflare.com/products/registrar/).

### DNS
Do this part first... it can take a while to propagate depending on your provider and it will just make your life easier
getting it out of the way. What you are trying to do is use Github Pages with an Apex record for your domain. This will let
something like https://luth.io be the place where people find your blog. Apex record support for a hosting service will let you
bypass using subdomains like https://blog.luth.io which are called CNAMEs in DNS terms. You're looking for A, ALIAS or ANAME
records which when pointed at the root of your domain need to be an IP address.

This requires Github Pages to support Apex records and provide IP addresses to point your DNS too. This was easily found 
as there is an entire write-up available from Github Pages [help documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain).
Following the advice from the article you will need to configure 4 `A` records and 4 `AAAA` records and for validation 
with Github you should add a CNAME for `www` pointed to your Github Pages username DNS e.g. `luthermonson.github.io`.

```
# A Records
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153

# AAAA Records
2606:50c0:8000::153
2606:50c0:8001::153
2606:50c0:8002::153
2606:50c0:8003::153

# CNAME www
<githubusername>.github.io
```

When configured through Cloudflare my DNS entries looked like the following, note that skipping the proxy and doing
a grey cloud "DNS Only" was intentional as the proxy will with SSL will cause an infinite redirect loop and Github
takes care of SSL for you.

![cloudflare luth.io settings](/img/2022/01/cloudflare-dns.png)

### Verifying your Domain with Github
You will need to add your domain to Github and add a DNS `TXT` record to be able to add it to a repository later. Go to your
[Github Pages settings](https://github.com/settings/pages) for your account (not repository) and Click "Add a Domain" and follow
the instructions by typing your domain, clicking next and getting the DNS `TXT` records to validate your Domain with Github Pages.

![github pages add a domain](/img/2022/01/github-pages-add-domain.png)

Grab the details from the Verification Step, add the `TXT` record to your DNS Provider and then click Verify.

### Github Repository
You will need a Github Repository to house your Hugo blog and all the content, this will be where you serve your pages.
Play with Hugo and create your blog, pick a [theme](https://themes.gohugo.io/) and get used to the Huge tool chain. Hugo 
will create a directory structure, by default when you run the `hugo` cli command it will generate your blog and place all
contents in the `./public` directory. Github Pages can NOT serve from anything other than the root `./` and the subdirectory 
`./docs` and for our purposes to serve from one Github Repository we will use `./docs`. To fix Hugo to publish to `./docs`
you can simply add to the `config.toml` a config line of `publishDir = "docs"`. When you now run `hugo` cli it will generate
all files to the `./docs` directory and prime your repository for serving via Github Pages.

To get Github Pages to serve the contents of your Blog with your domain you will need to configure the Pages settings for 
the repository. Check all your code into a branch and configure your Pages settings like the following.
![github pages repository setting](/img/2022/01/github-repository-pages-settings.png)

### Summary
If you configure your DNS properly and use the free Github Pages service you can easily host your personal blog at no cost
on your own personal vanity domain name from a single Github Repository. Configure Hugo to use the `./docs` directory to 
adhere to Pages standards then you'll be able to effortlessly set up your Blog. Go henceforth and buy up some novelty domain
names and get to blogging!

### Links
* [Hugo](https://gohugo.io/)
* [Cloudflare](https://www.cloudflare.com/) Free DNS Service
* [Github](https://github.com/)
* [Github Pages](https://pages.github.com/)
