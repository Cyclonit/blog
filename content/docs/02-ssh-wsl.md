---
date: '2025-03-01T09:14:22+01:00'
draft: false
title: '02 ssh and the WSL'
ShowToc: true
---

I am used to using Windows' graphical user interface for decades, but have been using Linux on servers and some testing devices for almost as long. To me, Linux always had superior CLI experience. When Microsoft introduced the Windows Subsystem for Linux (or WSL for short) a few years back, I jumped on it immediately. WSL brings all of the benefits a Linux CLI has, integrating it (almost) seamlessly into the Windows experience.

When using both Windows and WSL in parallel, configuring SSH can be somewhat annoying. Both have completely distinct ssh binaries and respective configurations. Thus, if you want ssh to work in both environments, you need to synchronize the configuration somehow. This also applies to those of us who use ssh authentication for git. The git integrations in IDEs like VS Code, Visual Studio and Intellij IDEA use Windows' own ssh. But if you prefer using a WSL in their integrated terminals, you find yourself using WSL's ssh instead.

Thus I decided to write this short article to explain how I set up my own WSL and solved this problem, allowing me to use one ssh config and set of ssh keys for both Windows and WSL.

## Problem

WSL runs in a dedicated linux file system. Each version you install creates a disk image file somewhere in your `%LOCALAPPDATA` directory. For the Ubuntu WSL I use, that path becomes `%LOCALAPPDATA%\Packages\CanonicalGroupLimited.Ubuntu22.04LTS\LocalState\ext4.vhdx`. Anything you do in the WSL happens on this disk image by default. You can access the linux filesystem from windows by opening the path `\\wsl$\<distribution>` in your explorer.

However, the reverse is also true. WSL mounts your Windows partitions automatically. Within the WSL, they show up as `/mnt/c` and so on. There is little keeping you from using the files directly. But when it comes to using ssh keys stored on your Windows partition there is one problem: File permissions. When looking at files on your Windows partitions from inside the WSL, you will not find any reasonable Linux file permissions.

```sh
$ ls -la /mnt/c/Users/Cyclonit/.ssh
total 8
drwxrwxrwx 1 cyclonit cyclonit 4096 Feb 24 21:29 .
drwxrwxrwx 1 cyclonit cyclonit 4096 Feb 17 14:00 ..
-rwxrwxrwx 1 cyclonit cyclonit  122 Feb 10 09:36 allowed_signers
-rwxrwxrwx 1 cyclonit cyclonit  419 Feb 10 08:57 github_ssh.id_ed25519
-rwxrwxrwx 1 cyclonit cyclonit  106 Feb 10 08:57 github_ssh.id_ed25519.pub
-rwxrwxrwx 1 cyclonit cyclonit  184 Mar  1 08:04 known_hosts
```

By default, Windows uses your Windows' users permissions do determine file permission in the WSL. Given that your current user has access to all of its files, the WSL assigns them the full range of permissions. In almost all situations, this will not be a problem.

Except, if you want to use the same set of ssh keys in both Windows and WSL. Your WSL's ssh *can* access your Windows keys, but it *refuses* to do so, because it considers the key's file permissions to be insecure. This is a perfectly reasonable defense mechanism, as it is supposed to alert you to the possibility of others being able to tamper with your keys, config or known hosts file.

## WSL automount metadata

Thankfully WSL does have a simple solution: automount metadata.

Within your WSL, edit the file `/etc/wsl.conf` and add the following lines:

```txt
[automount]
options="metadata"
```

This allows WSL to apply more restrictive permissions to individual files and directories than they have using the default behaviour. You won't be able to relax file permissions this way, but that does not matter to us right now.

Next, you'll have to restart the WSL instance for this change to take effect. On Windows, open a PowerShell and use this command. This will terminate all WSL instances. The next time you open a WSL, it will have the metadata flag enabled.

```ps1
> wsl --shutdown
```

Start a new WSL and test that everything works correctly:

```sh
$ chmod 600 /mnt/c/Users/Cyclonit/.ssh/github_ssh.id_ed25519
$ ls -la /mnt/c/Users/Cyclonit/.ssh
total 8
drwxrwxrwx 1 cyclonit cyclonit 4096 Feb 24 21:29 .
drwxrwxrwx 1 cyclonit cyclonit 4096 Feb 17 14:00 ..
-rwxrwxrwx 1 cyclonit cyclonit  122 Feb 10 09:36 allowed_signers
-rw------- 1 cyclonit cyclonit  419 Feb 10 08:57 github_ssh.id_ed25519
-rwxrwxrwx 1 cyclonit cyclonit  106 Feb 10 08:57 github_ssh.id_ed25519.pub
-rwxrwxrwx 1 cyclonit cyclonit  184 Mar  1 08:04 known_hosts
```

## Configuring ssh

One minor problem remains: ssh's own config file. Because Windows and WSL use different paths to access the key files, we need to use some tricks. For example, my GitHub ssh key is called `C:\Users\Cyclonit\.ssh\github_ssh.id_ed25519` in Windows, but `/mnt/c/Users/Cyclonit/.ssh/github_ssh.id_ed25519` in the WSL.

I define an environment variable `SSH_HOME` in both Windows and the WSL, containing the respective path. Windows' ssh config will become the "main" config, using this variable in each key's path. Lastly, the WSL will use Windows' ssh config using ssh's `Include` statement.

### Set environment variables

In Windows, create a new user environment variable called `SSH_HOME` and set its value to `C:\Users\<your username>\.ssh\`.

```ps1
> setx SSH_HOME C:\Users\Cyclonit\.ssh\ /m
```

In the WSL, add the same environment variable `SSH_HOME`, but this time set the value to `/mnt/c/Users/<your username>/.ssh/`.

```sh
$ echo 'SSH_HOME="/mnt/c/Users/<your username>/.ssh/"' >> ~/.bashrc 
```

The new environment variables will be available in new PowerShell and bash terminals. Thus, close all current terminals and create new ones prior to proceeding with the next steps.

### Update ssh config

In your Windows' ssh config, replace all references to private keys to use the environment variable. There **must not** be a `\` between the environment variable and your key's name, because the correct slash is included in the environment variable. For example, an entry for Github should look like this.

```txt
Host github.com
  User git
  IdentityFile ${SSH_HOME}<your private key's filename>
```

In your WSL's ssh config, add the following line. This will make your WSL's ssh use the Windows config file.

```txt
Include /mnt/c/Users/<your username>/.ssh/config
```

## References

- <https://learn.microsoft.com/en-us/windows/wsl/file-permissions>
- <https://man7.org/linux/man-pages/man5/ssh_config.5.html>
