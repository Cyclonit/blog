---
date: '2025-03-26T20:33:22+01:00'
draft: false
title: '03 git signing'
ShowToc: true
---

When starting with git for your first time, you've likely been prompted to set the configuration options `user.email` and `user.name`. The values you entered will be added to the author field of every commit you make. Github, Gitlab and the all time favorite `git blame` use this field to determine who is responsible for a change. But what prevents you or anybody else from using another persons email address and name? Nothing. A colleague might use this flaw to play a prank on you, but more nefarious actors could actually use it to shift the blame for injected malware and more.

Git signing aims to solve this problem. It is the process of cryptographically signing every commit and tag you create. As long as you keep your private key safe, you'll be able to confidently claim that only those things that have been signed using it were authored by yourself. But is this actually true? Not really.

While adding signatures does in fact bind commits and tags to a private key, it doesn't bind them to your identity. Only those who are both in possession of your public key and know that it is yours can verify the signed contents origin. This is the same problem that prevented GPG signatures for emails from becoming mainstream. Sharing your public key with everyone whom it may concern in a manner that is safe is certainly difficult.

This is where Git forges like Github and Gitlab come into play. You can let them do the heavy lifting for you. After uploading your signing key's public key to your profile, they will automatically verify the signatures on your commits for you. Then, whenever someone looks at your commits on either website, they will show a "Verified" status next to them.

![a verified commit in GitHub](github-verified.png)
![a verified commit in GitLab](gitlab-verified.png)

I consider signing your contributions to be a good practice that is easy to follow. On one hand, it gives you plausible deniability if someone else tries blaming changes on you, on the other hand others can trust that changes with your signature actually originate from you. Due to the increasing threat level on software supply chains and regulations tightening up, I expect more organizations to require contributions to be signed in the near future.

## SSH vs GPG vs S/MIME

I chose to use SSH for signing my work. There are two other options and I'd like to explain why I did not go for one of  them.

### GPG - GNU Privacy Guard

[GPG](https://en.wikipedia.org/wiki/GNU_Privacy_Guard) is the original when it comes to cryptographically signing things. It has a strong following and does its job well. As far as I know, its biggest use case is signing and encrypting emails. GPG is based on a web of trust, where users share their public keys with each other prior to sharing actual data.

When it comes to signing, there is not much of a difference to using SSH. An author uses their private key to generate signatures and others can use the author's public key to verify it. However, the GPG tooling is more complex than that of SSH. Instead of working with plain files, the user manages a keyring, similar to Java's truststores.

Ultimately, GPG doesn't bring anything extra to the table that would be relevant for signing commits.

### S/MIME - Secure/Multipurpose Internet Mail Extensions

Other than the name might suggest, [S/MIME](https://en.wikipedia.org/wiki/S/MIME) is not limited to being used for emails. While both SSH and GPG are limited to using a private/public key pair, S/MIME introduces certificates into the mix. Every user must have its public key signed by a certificate authority that is trusted by everyone they want to interact with. This can either be a public or an organization level CA, depending on the intended audience.

Adding certificates creates a proper relationship between a public key and thus the generated signatures and the certificate holder. This is certainly a nice to have, but being linked to a CA introduces cost.

When it comes to git, the signature verification is offloaded to the Git forge (GitHub, GitLab and the like). Thus the added complexity simply is not needed.

## Create a SSH key pair

You will need a SSH key pair. If you already have one you want to use, you can skip this step. Otherwise run the following command to create a new pair. Regarding key type and length, we use the default settings defined in ssh-keygen. As of time of writing, it will use the elliptic curve algorithm Ed25519 with 256 bits.

ssh-keygen will ask you where to save the generated private key, defaulting to `~/.ssh/id_rsa`. The corresponding public key will be placed in the same directory and have the same name, but ending in `.pub`. Take note of the paths, as they will be used later on.

If you want maximum security and prevent others from using this key if they happen to get a copy, you should add a passphrase. In case you do, you will be asked for the passphrase whenever you use the private key in the future. If you don't want to use a passphrase leave the prompt empty.

```sh
$ ssh-keygen
Enter file in which to save the key (~/.ssh/id_rsa): <optional alternative path>
Enter passphrase (empty for no passphrase): <optional passphrase>
Enter same passphrase again: <optional passphrase>
```

## Upload your Public Key

Both GitLab and GitHub work the same way, in that you must upload your public key, for them being able to verify your signature.

### GitHub

1. Login to GitHub.
2. Go to <https://github.com/settings/keys>.
3. To the right of **SSH Keys** click on the button `New SSH key`.
4. Use a title of your choosing.
5. Under `Key type` select `Signing Key`.
6. Under `Key` paste the content of your ***public key***. It should start with something like `ssh-ed25519`.
7. Finish by clicking `Add SSH Key`.

### GitLab

1. Login to GitLab.
2. Go to <https://gitlab.com/-/user_settings/ssh_keys> or the respective URL in your self hosted instance.
3. At the top of the list of keys click on `Add new key`.
4. Under `Key` paste the content of your ***public key***.
5. Use a title of your choosing.
6. Under `Usage type` select `Signing` or `Authentication & Signing` if you want to use the key for both purposes.
7. Finish by clicking `Add key`.

## Configure Git

We will configure Git next. I assume, that you want these settings to apply globally. If you want to restrict their effect to a certain repository, you must execute all commands from within that repository and ommit the flag `--global`.

### Identity Configuration

The options `user.name` and `user.email` are used as your identity within Git.

```sh,ps1
git config --global user.name "<your name>"
git config --global user.email "<your email>"
```

### Git Signing Configuration

These commands will configure commit and tag signing in git.

```sh,ps1
git config --global user.signingkey <path of your private key>
git config --global commit.gpgSign true
git config --global tag.gpgSign true
git config --global gpg.format ssh
```

- `user.signingKey`  
This is the **private key** corresponding to the private key used for signing commits and tags.
- `commit.gpgSign`  
Setting this option to `true` makes Git sign *all* commits you create.
- `tag.gpgSign`  
Setting this option to `true` makes Git sign *all* tags you create.
- `gpg.format`  
This tells Git that you use a ssh key to sign your commits. The naming is a remnant of Git only supporting GPG keys for
signing in the past.

## Testing Locally

We are going to test the configuration by creating an empty signed commit and having Git verify the signature for us. For this purpose, we must create the so called `allowedSignersFile`. It contains a list of identities and their respective public keys. You may choose any location on your machine for this file, we recommend to put it next to your key pairs (default `~/.ssh/allowed_signers`).

You are not required to maintain this file in the future, if you do not want to verify commit signatures locally later.

### Configure `allowedSignersFile`

We need to tell Git where it can find `allowedSignersFile`. This is done using the option `gpg.ssh.allowedSignersFile`.

```sh
git config --global gpg.ssh.allowedSignersFile <path of your allowedSignersFile>
```

### Create `allowedSignersFile`

The `allowedSignersFile`'s format is as follows:

```txt
"<identity>" namespace=\"git\" <key type> <public key value>
```

Assuming you have configured Git according the commands in the previous sections, you can create the entry for your own identity using the following command.

```sh
echo "\"$(git config --global user.name)\" namespaces=\"git\" $(awk '{print $1, $2}' '<path of your public key>')" >> $(git config --global gpg.ssh.allowedSignersFile)
```

(!) When creating files via the commandline, Powershell prefers the encoding `UTF-16 LE` and uses `CLRF` as line endings. Thus we must set the encoding and write a proper `LF` as part of the command.

```ps1
"""$(git config --global user.name)"" namespaces=""git"" $(awk '{print $1, $2}' '<path of your public key>')`n" | Out-File $(git config --global gpg.ssh.allowedSignersFile) -Encoding utf8 -NoNewLine
```

### Signing a Commit

For testing, we will create and sign an empty commit. If everything is configured correctly, the second command `git show` will contain a line confirming that the commit was signed with a good signature. The last command `git reset` will undo the empty commit, such that you have a clean git history going forward.

Navigate to any git repository on your machine and run these commands.

```sh
$ git commit -m "chore: test commit signing" --allow-empty
$ git show --show-signature
commit <commit hash> (HEAD -> main, origin/main)
Good "git" signature for <your name> with <key type> key SHA256:<key hash>
Author: <your name> <your email>
...
$ git reset HEAD~
```

## References

- <https://en.wikipedia.org/wiki/GNU_Privacy_Guard>
- <https://en.wikipedia.org/wiki/S/MIME>
- <https://git-scm.com/book/ms/v2/Git-Tools-Signing-Your-Work>
- <https://stackoverflow.com/questions/73489997/whats-the-difference-between-signing-commits-with-ssh-versus-gpg>
- <https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification>
- <https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key>
- <https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits>
