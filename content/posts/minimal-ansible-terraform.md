---
title: Minimal Terraform+Ansible
date: 2024-09-20
draft: false
---

# Minimal Interop from Terraform to Ansible

*I have plans to publish more code and maybe eventually write larger posts about this once grinding out and reading a lot of code isn't my full time priority. Currently I'm focusing most of my efforts on an upcoming project. I've taken brushing up on Infrastructure as Code on as a side project in anticipation of needing a "set and forget" backend deployment solution. MVP for this capability is: compute, storage, firewall, VPN admin access, onsite+offsite backups and monitoring.*

I picked up Terraform to add it to my resume but kept it because it's still got plenty of upside for a solo developer. Despite the marketing focus on cloud-enabled state sharing, it's still a tool that shines when it comes to provisioning resources.

Configuration management is actively discouraged, though. I ran in to this personally trying to write a module that deployed a web stack I've used before. Providers for configurable infrastructure software like Docker expect the remote node to exist. If the remote host referenced in your provider configuration isn't up and running, you'll probably get an error. Maybe you can you can use targeting or clever tiering to get around it the issue but to me it reinforced something I'd seen all over the internet:

**Provision** with Terraform, then **configure** with Ansible.

It's probably overkill if you're starting from scratch, but I first used Ansible in anger at a job 5 years ago and have just decided to limit the scope of my Terraform needs so I think it plays to my strengths to use both tools. Both tools are outstanding when things scale to the medium-large end of what a small team can handle. As a solo developer without real world experience in these tool the benefits are **not** in the time saved automating either setup or maintainance (you're really just shifting both around conceptually and probably not coming out ahead).

Rather, you gain some fundamentals that could help you scale **and** whatever you do manage using Infrastructure as Code is declared and self-documenting. You'll thank yourself later that you took an extra 15 minutes to write that modification into the playbook instead of cowboy configuring the live instance.

I also happen to think it's a pretty easy toolset to hand to someone if you can make the glue reusable. Deploying some VPSs with Terraform and then a stack using a well-known community role is easy for a motivated or tangentially experienced beginner. The biggest hurdle for Terraform is usually syntax and internals. On the other side, the literal file structure and workflow of Ansible are daunting when you know what you're doing amounts to a handful of shell commands. Until you're nimble with both you have to keep good discipline and adjust quicly when things aren't working.

## How I'm Doing It

How any solo developer saves complexity and time budget -- loose coupling!

All Ansible code is written with knowledge of Terraform code which passes forward any required state by writing it into the inventory. There is no magic Terraform incantation that triggers Ansible behavior. You add resources to the inventory and update playbooks accordingly.

All that means is the only real difference is that the inventory is fully managed by Terraform, which possesses a great set of tools for basic templating and file manipulation.

#### Note: Existing Ansible Providers

There are apparently providers that "call" Ansible from Terraform but I found there are tricky ways to paint yourself into corners manipulating dependent state in Terraform. This avoids that by having short dependency chains on the Terraform side. The only real programatic link between the two is variable passing. The rest is hand-written glue.

Thankfully, Ansible makes it easy to target groups with roles. It's a neat fit with no hacking required beyond some file generation.

The other thing I tried with one of the Ansible providers was creating vault files. Unfortunately, it wasn't easy to install the provider on my machine and I ran in to strange errors. After a little thinking on the problem I decided to side-step it by tapping in to my decision to limit the link between tools to just passing variables.

All I need to write "by hand" is an easily scripted usage of the `ansible-vault` CLI. Trying to build the provider for my architecture took about 45 minutes of fiddling before I gave up (after thinking it was working on the first try until I got halfway into the apply). **Writing the one thing I needed took 10 minutes.**

#### Note: Dynamic Inventories

No interest for my current needs. Text files do not ([usually](https://www.crowdstrike.com/en-us/)) throw errors or require debugging.

### Example: `ansible/inventory/hosts.yaml`

This is very simple -- just pass a map into a `local_file`.

```hcl
resource "local_file" "ansible_hosts" {
  filename = "${path.root}/ansible/inventory/hosts.yaml"
  content  = yamlencode({
    # ...
  })
}
```

Where it gets hard is when you need to do post-processing like encryption...

### Module: `host_vault`

Here's the code. It's very easy to copy and split into three files. It also is a "sink" so no outputs are required. Drop in the required params and provide the password for the specified vault using a CLI flag when calling `ansible-playbook`. Your secrets will be imported automatically as host_vars. I use it for Tailscale auth keys! 

It may seem like you'd want to use the `local_sensitive_file` resource for the initial file write, but think a bit about what that means. Sensitive simply means redacted in log output -- the *contents* remain in state. But since we use a provisioner to "process" the contents after visiting the base resource there will **always be a state mismatch, triggering a replacement**. The `terraform_data` resource stabilizes around the `input` which can also conveniently be referenced in a `when = destroy` execution (unlike variables, locals or resources). You then perform the initial write as a prequel side effect to the `ansible-vault` invocation.

It's a little hacky but I get to move on and cement my knowledge of how Terraform's `declaration + state = convergence` schtick fits together.

```hcl
variable "host_name" {
  type        = string
  description = "The exact name you use in your hosts file."
}

variable "vault_contents" {
  type        = map(any)
  sensitive   = true
  description = "A map of strings, numbers, lists and maps. Rougly speaking."
}

variable "ansible_vault" {
  type        = string
  sensitive   = true
  description = "A vault-id in the form 'VAULT@/path/to/passfile'."
}

locals {
  # this is the structure I use -- modify it as you see fit
  host_vars_path = "${path.root}/ansible/inventory/host_vars/${var.host_name}"
}

resource "terraform_data" "host_vault" {
  input = {
    payload = jsonencode(var.vault_contents)
    path    = "${local.host_vars_path}/vault.yaml"
  }

  # create/replace vault file
  provisioner "local-exec" {
    command = "echo '${self.output.payload}' > ${self.output.path}"
  }

  # encrypt vault file
  provisioner "local-exec" {
    command = "ansible-vault encrypt --vault-id ${var.ansible_vault} ${self.output.path}"
  }

  provisioner "local-exec" {
    when    = destroy
    command = "rm -f ${self.output.path}"
  }
}
```

Here's how that looks in action as a module to put my Git server on Tailscale (I don't expose it to the internet). The `tailscale_tailnet_key` resource lets you generate a one-time-use token that pre-authorizes whatever machine uses it to perform its first `tailscale up`. I've also included the relevant section of the hosts file.

```hcl
resource "tailscale_tailnet_key" "git_host" {
  preauthorized = true
  description   = "DigitalOcean Git Host"
}

module "git_host_vault" {
  source = "./host_vault"

  ansible_vault  = local.ansible_vault
  host_name      = module.git_host.fqdn
  vault_contents = {
    tailscale_authkey = tailscale_tailnet_key.git_host.key
  }
}

resource "local_file" "ansible_hosts" {
  filename = "${path.root}/ansible/inventory/hosts.yaml"
  content  = yamlencode({
    tailnet: {
      hosts: {
        "${module.git_host.fqdn}": {
          # this key has been configured using the DigitalOcean provider
          "ansible_ssh_private_key_file": abspath(var.dev_private_key_file),
        },
      },
      vars: {
        # some people have a bootstrap tier in Ansible to setup
        #   a user with passwordless root (per the Ansible docs)
        # but I use a provisioner or cloud-init user_data 
        # (preferably the latter but things get weird in practice)
        "ansible_user": "admin",
      }
    }
  })
}
```

This playbook worked for me first try. Ansible really has matured so well. The role I've chosen has Molecule tests so you can confidently deploy it against any supported platform. I want to learn from it when I build my little Tarsnap role (nobody else has!) as Colin Percival has written nice instructions for setting up the package repos on multiple platforms.

```yaml
# Ansible playbook
- name: Tailscale
  hosts: tailnet
  roles:
    # make sure to install with ansible-galaxy
    - role: artis3n.tailscale
  # the tailscale_authkey is implicitly included from host_vars
```
