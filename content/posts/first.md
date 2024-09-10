---
title: Hello World
date: 2024-09-09
lastmod: 2024-09-10
draft: false
---

Recently, I started a project using DigitalOcean and discovered the App
Platform's static site feature. I think I had heard about it before I discovered
it in their documentation but it's exactly what I needed -- GitHub pages (for my
traffic needs which fit into the free tier) with any public Git provider. I can
deploy a static site automatically with a simple platform-agnostic CI/CD step
(one `curl` command that runs on push).

DigitalOcean's Terraform support is robust and I've got a bit of fragile
configuration (DNS records for email, namely) that I'd like to be less scared of
touching. I was a self-taught Ansible user about 6 years ago and for my minimal
needs (well within Hashicorp's free tier limits) this really does feel a bit
like free magic.

As for the actual blog it's a much smaller template that makes JavaScript
optional. I really don't need more than that and am already feeling a little
lighter knowing I can just write Markdown and publish HTML without `npm`.

Can't wait to tell y'all about what I'm building next!
