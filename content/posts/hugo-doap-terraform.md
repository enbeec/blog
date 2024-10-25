---
title: "Terraforming: Hugo on DOAP"
date: 2024-09-10
lastmod: 2024-09-10
draft: false
---

# Hugo on DigitalOcean App Platform

I decided to publish the 
[Terraform module](https://github.com/enbeec/blog-infra) 
for [this site](https://github.com/enbeec/blog)
just to get a feel for using non-local modules.

It wraps the `digitalocean_app` resource with a hardcoded spec for a static
site. Hardcoding is usually a bit of a stink but in its current form the
resource isn't anything you need a "shortcut" for -- it's literally identical to
the examples in the provider documentation. 
~~I experimented with outputting the domain name from my DNS module so that it could be inputted as a variable to use for `spec.domain.zone`. I'm just not sure that that's worth it considering I do not plan to change the domain... ever? There are critical MX/SPF/DKIM records on the `valcurrie.com` domain so I'm also not worried about losing that "dependency" as the blog is a relatively lower priority.~~
I'm writing this blog post to document learning when to walk away from complexity and do
something "ugly" when it's by far the best solution. **Update!** I'm now just
passing in the domain as two strings: the zone and the shortname. Up to you if
it seems worth having the zone be stored in `terraform_data` or if you think
it's okay to just pass a string that you know matches a configured domain. I
didn't bother.

The one special thing I've done is include a module output with a cURL command
that interpolates the app ID (which is also available as an output in case I
need to try something quickly). If I were to split this module off and modify it
to be something more generic I might try to find a way to insert that app ID
into the linked repository's pipeline somehow. All that sounds like a waste of
time for my blog, though. If I ever do an `apply` that recreates the app I'll
just need to copy and paste the ID or the whole command. 

You could create a function with a known FQDN that calls the endpoint with the
proper ID. Not necessary for this project but not a terrible solution
considering it executes quickly/cheaply -- the request does not block on the
build completing. I may add this at some point
