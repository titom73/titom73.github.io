---
title: Ansible VAULT with 1password
date: 2022-03-29
categories: [Ansible]
tags: [ansible, shell, security]     # TAG names should always be lowercase
---
Ansible provides a tool to [protect sensitive data](https://docs.ansible.com/ansible/latest/user_guide/vault.html) in inventories and makes them unusable without encryption password.

Documentation for ansible-vault is pretty nice and very helpful, but at some point you will have to think about how to protect your vault password and make it easy to use in your project. Idea in this blog post is to leverage 1password CLI tool to get this password.

![](/assets/img/ansible-vault-1password/1password-login.png)

## Install 1password CLI tool

Here we will assume installation will be done on MacOS, if you are running a different OS, please check [1password support page](https://1password.com/downloads/command-line/) for step-by-step installation process.

```bash
# Install 1password op CLI tool
brew install --cask 1password/tap/1password-cli

# Install JQ for JSON parsing
brew install jq
```

Then, configure your `op` tool to login to your realm:

```bash
$ op signin <your-realm>.1password.com
```

## Create script to get VAULT passphrase

To do that, we will need your 1password vault ID where your vault password is sotred (`$1PASSWORD_VAULT_ID`) and the 1password entry where your VAULT passphrase is saved (`$1PASSWORD_VAULT_ITEM`).

__Get 1Password VAULT ID__

```bash
 op vault ls
ID                            NAME
xxxxx                         Personal
yyyyy                         Works
...
```

Let's consider VAULT entry is saved in Works.

__Get your 1Password Vault entry__

Just run the following command, assuming your 1password entry is 'Ansible Vault':

```bash
op item get "Ansible Vault" --format json
```
And the result will be something like this:

```json
{
  "id": "id1010101010",
  "title": "Ansible Vault",
  "version": 5,
  "vault": {
    "id": "zzzzzz"
  },
  "category": "PASSWORD",
  "last_edited_by": "xxx",
  "created_at": "2020-09-01T11:19:19Z",
  "updated_at": "2022-03-28T16:46:59Z",
  "sections": [
    {
      "id": "add more"
    },
    {
      "id": "linked items",
      "label": "Related Items"
    }
  ],
  "fields": [
    {
      "id": "password",
      "type": "CONCEALED",
      "purpose": "PASSWORD",
      "label": "password",
      "value": "my-super-safe-password",
      "entropy": 1,
      "password_details": {
        "entropy": 141,
        "generated": true,
        "strength": "FANTASTIC"
      }
    },
    {
      "id": "notesPlain",
      "type": "STRING",
      "purpose": "NOTES",
      "label": "notesPlain",
      "value": ""
    }
  ]
}
```

So now, idea is to use `jq` to get only password field:

```bash
# JQ filtering for 1Password version 8
jq '.fields[] | select(.id=="password").value'
```

Finally, we have following command:

```bash
$ op item get --format json "Ansible Vault" |jq '.fields[] | select(.id=="password").value' | tr -d '"'
my-super-safe-password
```

__Create your vault__

Now we will put everything in a small bash script to use with ansible

```bash
# vim ~/bin/op-vault

#!/bin/bash

# Arista Vault
VAULT_ID="yyyyyy"
VAULT_ANSIBLE_NAME="Ansible Vault"

op item get --format json --vault=$VAULT_ID "$VAULT_ANSIBLE_NAME" |jq '.fields[] | select(.id=="password").value' | tr -d '"'
```

## Use your vault in ansible

So now you can easily encrypt/decrypt using `ansible-vault` with following command:

```bash
ansible-vault encrypt_string 'ansible' \
    --name 'ansible_ssh_pass' \
    --vault-password-file=~/bin/op-vault

ansible_ssh_pass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          38373166646364376632366166383138663238346164386161646536646266633330623966323730
          3535633165623632303331393361656663633236366264660a343562636435346630656264353432
          36636263613838313836663031326636663461343634333333333663313332343839303131363061
          6537653439353630300a626135313461343136626634633764303839653938616264646437623131
          3738
Encryption successful

```

> Note: Don't forget to accept connection to 1password

You can also use it during playbook execution:

```bash
ansible-playbook --vault-password-file=~/bin/op-vault ...
```

And because it can be painful to type this option all the time, `alias` comes to the rescue:

```bash
$  alias ansible-playbook-vault='ansible-playbook --vault-password-file=~/bin/op-vault'
```

Here you are, it is now easy to protect your sensitive data in your inventories and reduce the risk to loose your passphrase