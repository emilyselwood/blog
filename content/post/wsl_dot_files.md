---
title: "Windows Subsystem for Linux Dot files"
date: 2018-08-09T08:25:13+01:00
author: "Emily Selwood"
draft: false
slug: "wsl_dot_files"
tags: ["Windows Subsystem for Linux", "wsl", "dotfiles"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

I've been using the Windows Subsystem for Linux (WSL) for a while at work. It's been something of a radical improvement over cygwin for most of my use cases. A couple of weeks ago my installation got borked by the work Antivirus (AV) system. So I've created an automated script to set up my environment just how I currently like it.

Before we start lets get one thing out the way. Windows is a perfectly fine development environment. I won't be tolerating any comments of just use a mac or just install linux. This is not always possible with a corporate machine. If this is of no help to you stop reading and go look at something else.

# Assumptions

You have Ubuntu for windows already installed but not yet configured

# Process

Step one was to create a directory to keep my initial environment in. Later we will turn it into a git repo as this sort of thing should not be kept on your machine. The times you need it most are when your machine has died and you need to build a new one.

```bash
mkdir -p /mnt/c/git/dotfiles/bin
cd /mnt/c/git/dotfiles
```

Next is to create the skeleton of the script that will do all the work for us in future. In the bin folder create a script called `setup.sh`

```bash
#!/bin/sh

# This needs to be run with sudo

apt update
apt upgrade -y

```

This updates all the existing packages.

Next we need to install the new things that we want. Add the following underneath:

```bash
apt install -y $(cat ../pkglist.txt)
```

This installs everything listed in a file one level up called `pkglist.txt` lets go and create that now. The following is mine:

```text
build-essential
git
ssh
vim
tmux
make
wget
curl
zip
python3
python3-pip
xdg-utils
dos2unix
jq
html-xml-utils
```

Next we will create a directory in our home folder called bin to house all the executable things we need for our user:

```bash
mkdir -p ~/bin
chown wselwood: ~/bin
```

We need to change the owner of the bin folder as we are currently running as root 

Now my work environment is very mixed. I work in a mixture of almost even parts Scala/Java, Go, and Python. I install python 3 the pkglist.txt already so that is easy enough. Next we will tackle java

## Java

There are plenty of guides about installing java on a ubuntu system out there. [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-java-with-apt-get-on-ubuntu-16-04) have a good one that I usually follow. It boils down to adding a line to the `setup.sh` file and one entry in the `pgklist.txt`

The `setup.sh` should now look like this

```bash
#!/bin/sh

# This needs to be run with sudo
add-apt-repository ppa:webupd8team/java
apt update
apt upgrade -y

apt install -y $(cat ../pkglist.txt)
```

Then add `oracle-java8-installer` to the `pgklist.txt` file.

Almost done. One last little trick to make our Java life easier. If we sym-link our WSL `~/.m2` folder to our windows `.m2` folder we can avoid having two sets of libraries and very strange problems if we do `mvn install` at any point. Add the following to the bottom of your `setup.sh`

```bash
ln -s /mnt/c/Users/Wil.Selwood/.m2/ ~/.m2/
```

Next on to go

## Go

Unfortunately at the time of writing the version of Go in the ubuntu package manager is way out of date so we need to go and get the latest version our selves. For this we are going to use a few tricks to make sure we always download the latest version.

First off installing go its self. Unfortunately there isn't a handy URL I could find that always points to the latest stable build. There is a link on the [download](https://golang.org/dl) page but it changes with each version. So we are going to use the `html-xml-utils` package of tools to do some nasty (but better than pure regex) command line foo to get hold of the url from that page. Using chrome inspect the download link we are after. It should have a class of `downloadBox` So we can use that to find the one we need

1. Get hold of the page. We will use curl (cat url from my understanding) pass it a URL and it returns the text content to stdout.
1. Normalise the html into valid XML
1. Extract the links with a class of downloadBox
1. Find the link that says its for linux-amd64
1. Extract the link from that line.

The following line does this and stores the resulting text in a variable for us to download later:

```sh
go_url=$(curl --silent "https://golang.org/dl/" | hxnormalize -x | hxselect -s '\n' -i 'a.downloadBox' | grep linux-amd64 | grep -Po 'http[^\"]+')
```

Thats a bit long and unpleasant but it does the job. Now that we have the download url we can download, extract, and link to our path

```sh
wget $go_url -O ~/go-linux.tar.gz
tar xzf ~/go-linux.tar.gz -C ~/
ln -s ~/go/bin/go ~/bin/go
ln -s ~/go/bin/godoc ~/bin/godoc
ln -s ~/go/bin/gofmt ~/bin/gofmt
```

That should get us go installed. I've a few projects that use dep for package management so we need to install that too. This is a little bit easier as dep has its releases in github which provides an api to download them so we can use the `jq` command line tool to extract the url we need from the json response.

1. Send the query to the github API
1. Extract the asset that has a name ending with linux-amd64 and pull out the browser_download_url

```bash
dep_url=$(curl --silent "https://api.github.com/repos/golang/dep/releases/latest" | jq -r '.assets[] | select(.name | endswith("linux-amd64")).browser_download_url')
```

Now we just need to download that url and make it executable

```bash
wget $dep_url -O ~/bin/dep
chmod +x ~/bin/dep
```

There we go. That should be go set up and ready to work.

Now on to some quality of life changes

# Quality of life

## vim and git default configuration

I have a copy of my `.vimrc` and `.gitconfig` files stored in the dotfiles directory so we need to copy those over to our home directory. The following mess of characters copies everything from the directory above that starts with a dot to our home directory. Note there is no -r flag on the cp command so it will not copy the .git folder that exists in our project.

```bash
cp ../.* ~/
```

My configuration for vim and git are not too complex. I am colour blind and find the default colour schemes of both programs hard to read. So I change them in this config.

## bash

I use [mrzool's Sensible bash](https://github.com/mrzool/bash-sensible) as a starting point here. Along with a `.bashrc` file that I have had for so long I'm not sure where it came from.

Drop the `sensible.bash` file into the bin directory and the entry to source it to your `.bashrc` file. There are instructions in the git repo.

I also add an entry to make `~/bin` and `~/.local/bin` be part of the path.

Next I like to enable `xdg-open` to start a web browser. I find this really useful to include at the end of long build processes to open a browser to the server I have just launched so that I get pulled back from what ever I went off to do while the build ran.

The first step is to create a sym-link that points to your windows browser executable. Chrome in my case.

```bash
ln -s /mnt/c/Program\ Files\ \(x86\)/Google/Chrome/Application/chrome.exe /usr/bin/chrome
```

Then in your `.bashrc` file add `export BROSWER=chrome` and you should now be able to type `xdg-open https://google.com` and it will pop open a new chrome tab for you.

The last thing I have added to my `.bashrc` file is a thing that starts my ssh agent the first time I open a shell. This actually works way better under WSL than it did under cygwin as it lasts until you logout rather than as long as a single shell is left open. You may not want to do this depending on how paranoid you are about security. However it does mean that I don't have to type in my ssh key password several hundred times a day.

```sh
export SSH_AUTH_SOCK=$HOME/.ssh-socket

ssh-add -l ? /dev/null 2>&1
if [ $? = 2 ]; then
  rm -rf $SSH_AUTH_SOCK
  ssh-agent -a $SSH_AUTH_SOCK >| /tmp/.ssh-script
  source /tmp/.ssh-script
  echo $SSH_AGENT_PID >| ~/.ssh-agent-pid
  rm /tmp/.ssh-script

  ssh-add
fi
```

And there we will leave the changes. Now all that we need to do is commit this to a remote repo somewhere. You can find mine on [github](https://github.com/wselwood/dotfiles) There are a few tweaks in there I don't mention here as they are only todo with the way I my disk is layed out.

I hope you have found this useful and have some ideas for your own environment. If you have any questions or comments please let me know on [twitter](https://twitter.com/wilselwood) I'd love to see your suggestions and little things you have found that makes your life a bit easier.
