---
title: "Build Software? No Laptop? Have Android? No Problem!"
datePublished: Thu Nov 27 2025 10:20:21 GMT+0000 (Coordinated Universal Time)
cuid: cmiha8qyh000f02l7htrb80fk
slug: build-software-no-laptop-have-android-no-problem
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1764238802877/b92e656c-a518-4c23-80d7-18606c590d84.jpeg
tags: postgresql, linux, go, neovim, cli, android, tmux, termux, pgweb

---

## Initial Steps

Download F-Droid here: [https://f-droid.org/en/](https://f-droid.org/en/)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764233860355/aa18dfa2-1b3b-4cf3-84c8-1775ac5333d3.png align="center")

Search and install Termux and Termux API.

## Setup Termux for Development

The first commands to run after opening termux are:

```bash
pkg update && pkg upgrade
```

This updates all the packages in the repository.

### Android Integration

To integrate with android features:

```bash
termux-setup-storage # allow termux to access your file storage
```

You should be able to access all files in your android device in the path: `/storage/shared/`.

```bash
pkg install termux-api
```

This allows syncing termux with android. I install this to sync my system clipboard with termux's clipboard. That's the only reason I use it. But I'm sure it can be used for more other hacky things, but that's not the focus of this guide. Remember that the termux-api we installed in F-Droid is required for this package to work as expected.

## Installing Linux on Android

We will install `proot-distro` to install our favourite distro. I use arch btw, so that's what I'm installing, but there are other options:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764234540029/25e64fca-661b-4cbb-b560-0b521a7a4c88.png align="center")

> You might want to buy a wireless keyboard and mouse. It makes life easier.

```bash
pkg install proot-distro
```

To install arch:

```bash
pd install archlinux
```

After installation, login into your archlinux shell:

> Note: There is no GUI, only CLI

```bash
pd login archlinux
```

## Setting Up Archlinux

Update your repositories:

```bash
pacman -Syu
```

Install any software you'll need for development. For myself, I have neovim as my editor and I primarily work with go and postgresql.

```bash
pacman -S go neovim nodejs npm cargo python python-pip
```

I added in cargo, nodejs, python, python-pip, and npm, because they are needed for the neovim extensions in my config. You can view my neovim config here: [https://gitlab.com/personal-os/nvim](https://gitlab.com/personal-os/nvim).

In addition to neovim, I have other tools I use for productive developer experience, you can explore them if you want to:

```bash
pacman -S git make unzip gcc tmux ripgrep fd wget curl tar fzf
```

## Customization

For out-of-the-box auto complete, I use fish and then, starship for a nice shell prompt.

```bash
pacman -S fish starship
```

For version control, I use [Jujutsu](https://docs.jj-vcs.dev/latest/). It approaches version control differently from git in a way that is simpler for developers and far more intuitive in my experience.

```bash
pacman -S jujutsu
```

My updated `fish.config`:

```bash
function fish_greeting 
end

starship init fish | source

jj util completion fish | source # don't add this line if you are not using jj-vcs
```

We will be install some go applications, don't forget to update your `$PATH`

```bash
fish_add_path go/bin/
```

At this point we can update our git config:

```bash
git config --global user.name "Your name"
git config --global user.email "youremail@email.com"
git config --global init.defaultBranch main
```

To setup ssh for working in your remote repos, github provides an excellent guide here:

* [Generating an SSH key and adding it to the key agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent).
    
* [Add a New SSH Key to Github Account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account).
    

Next, we get our neovim config:

```bash
git clone https://gitlab.com/personal-os/nvim.git ~/.config/nvim
```

We also need to get our tmux config:

```bash
wget https://gitlab.com/personal-os/dotfiles-v2/-/raw/main/tmux/tmux.conf?ref_type=heads -O ~/.tmux.conf
```

We are good to go.

## Creating and Using Postgres

To use a database in postgres, we need to open a new shell on termux. Long press the edge of your screen, it will bring up something like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764237140917/52dd6e87-f560-45f0-995e-e871b4d4ec15.png align="center")

Select new session. This will open a new termux session.

We will install postgres in there.

```bash
pkg install postgresql
```

When you are done, you need to initialiaze the database:

```bash
initdb -D $PREFIX/var/lib/postgres/data
```

Create a postgres user:

```bash
createuser --interactive
```

Create a postgres database:

```bash
createdb local # local can be any name of your choice
```

Then we need to start the database:

```bash
pg_ctl -D $PREFIX/var/lib/postgresql start
```

In case you want to stop it, you run a similar command:

```bash
pg_ctl -D $PREFIX/var/lib/postgresql stop
```

As a lazy programmer that I am, I placed this into a bash script so I don't think about it.

We now login into the database:

```bash
psql local
```

We need to make some updates:

* Add password for our user.
    
* Make the user the owner of local.
    

```pgsql
ALTER USER user_name WITH ENCRYPTED PASSWORD 'your_password';
ALTER DATABASE local OWNER TO user_name;
```

We can exit the postgres shell by typing `exit` or the classic, `CTRL-D`.

I agree that managing a database without a GUI can be a pain sometimes, so we will setup a GUI for our database. Now, the advantage of termux is that though we cannot run native linux GUI application, we can always run web servers. We are going to install **pgweb,** which is a web UI for managing postgres.

Installing pgweb: (you can choose to install on termux shell or login into your archlinux shell)

* We need Go installed. We did that previously so we are good.
    
* We need to add the `GO BIN` directory to our `$PATH`. We did already as well.
    

```bash
go install github.com/sosedoff/pgweb@latest
```

When it is done, run the command:

```bash
pgweb
```

Navigate to `http://localhost:8081`. That is it. You will see this beautiful interface that you can even install as a PWA:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764238002310/a3fe9180-e8da-416a-875d-2f9376a7028b.png align="center")

As you can see, I'm working on a side project 😂.

This is how my tmux + neovim looks like:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1764238078506/04a8c030-a961-4c02-ae12-85306bf883f0.png align="center")

Alright, I'm going to finish the project. If you have any questions, you can ask in the comments and I'll respond. Anything linux? I'm your guy.

## Bonus

Now you might be wondering: what about docker? Docker needs kernel access to work and my android device isn't rooted so docker will fail. But… We can install a virtual machine and run it there. However, your device needs to have more RAM, at least 8GB for a smooth workflow.

You can follow oofnikj's guide here: [https://gist.github.com/oofnikj/e79aef095cd08756f7f26ed244355d62](https://gist.github.com/oofnikj/e79aef095cd08756f7f26ed244355d62). That is what I used and it worked seamlessly. And yeah, put that startup command in a bash script 😂.