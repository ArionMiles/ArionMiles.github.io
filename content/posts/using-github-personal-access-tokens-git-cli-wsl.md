---
title: Using GitHub Personal Access Tokens with Git CLI on WSL
date: 2021-06-24
type: "post"
description: "Accessing GitHub repos on WSL with Personal Access Tokens"
tags:
  - Linux
  - GitHub
---

I've been using my GitHub username-password for authenticating with GitHub through Git CLI for a long time now. [GitHub announced in July last year](https://github.blog/2020-07-30-token-authentication-requirements-for-api-and-git-operations/) that they'll be moving to using token-based authentication for all authenticated Git operations (cloning private repos, pushing, etc.) [Beginning from Aug 13, 2021](https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/), any authenticated operation will require token based auth (personal access tokens, OAuth, etc.). To this end, GitHub has been emailing their users every time they carry out an authenticated Git operation using passwords (once per day). I've been receiving these deprecation notices for a year now and I've been lazy about it. I finally got off my ass and setup Git CLI to use Personal Access Tokens (PATs).

## Old setup

I'd been using `credential.helper=store` to save my GitHub username-password. This helped me avoid having to enter my credentials everytime I ran clone/push.

The way it works is, you run

```
$ git config --global credential.helper store
```

and the next time you do a pull/push/clone, you'll be prompted for credentials, which will be stored and used on subsequent operations. I learnt today that it actually stores this information in plaintext under `~/.git-credentials`.

![Surprised Pikachu](/images/surprised-pikachu.png "Surprised Pikachu")

Yikes! Serves me right for blindly copying commands off StackOverflow.

## Fixing it up

I had already adapted to using PATs. I've been using them to work with repos on a VPS. They're very nice, since you get granular permissions and each machine can have its own token, so in case one token is compromised, you can simply revoke it, create a new one and be on your way.

I wanted my token stored on my machine and I wanted it to be stored safely. While looking up solutions I learnt that Mac and Windows have dedicated credential stores which store secrets safely and integrate well with git. I couldn't find such native solution for Linux, it was either third-party solutions, hand-rolled solutions (which I'd never consider when it comes to security) or plain old text.

I looked up to see if there was a credential helper available for WSL. [I found this great documentation](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-git#git-credential-manager-setup) which describes using the Windows Credential Manager from within WSL to access secrets.

The way it works is, it uses the Git Credential Manager (GCM) on the Windows installation of Git (you need to have [Git for Windows](https://git-scm.com/download/win) installed) which in turn communicates with Windows Credential Manager to fetch the secret.

First thing I did was remove the `~/.git-credentials` file.

Running below command sets WSL distro's credential helper to the Windows GCM:

```
$ git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/libexec/git-core/git-credential-manager-core.exe"
```

and the next time you clone/push/pull or do any operation requiring authentication, you'll have a login prompt open.

Now, it can work in 2 ways -

1.  If you click the "Login with GitHub" button, it'll authenticate via OAuth and save the oauth token to Windows Credential Manager
2.  If you close the prompt, it'll ask for username and password. You can enter your GitHub username and in place of password, enter the PAT you created from the GitHub settings. This username-token pair will then be saved by Windows Credential Manager.

I chose to go with the latter option for the reasons mentioned previously.

## Conclusion

Setting up Git CLI to use GitHub PATs and having them stored securely locally was fairly straightforward. I regret putting off doing it for a year. Another alternative I looked into was using SSH, and there's great [documentation from GitHub](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh) for it. I chose not to go with it in this case because it'd require me to change my remote URLs from HTTPS to SSH, and I'm too lazy to bother with it. However, I'll be trying it next time when I have to setup a new machine.
