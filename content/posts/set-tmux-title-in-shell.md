---
layout: post
title: "set tmux title in shell"
date: 2016-01-01 00:00:00 +0800
comments: true
tags: [ tmux, shell ] 
---
change window title dynamically in tmux
<!--more-->
according to [this](http://superuser.com/questions/249293/rename-tmux-window-name-to-prompt-command-ps1-or-remote-ssh-hostname) thread, i set my tmux windows title to the dir name of $PWD, thus i can tell the dirname of each windows :

{{< highlight bash >}}

export PS1=$'\ek${PWD##*/}\e\\\\\${USER}@${HOSTNAME} $ '

{{< /highlight >}}

![](/images/tmux1.png)  

everything went well, until I exit tmux :

![](/images/tmux2.png)  

turn out if you're out of tmux , there's nothing can absorb the \033k...\033\\ escape sequences, so it's printed out 


usually you don't exit tmux ,but it seems really weird and imperfect for a normal shell.

luckily i found the magical two cursor command : sc (save_cursor) and rc (restore_cursor)

you can use it to wrap you 'tmux shell command ':

{{< highlight bash >}}
export PS1=$'$(tput sc)\ek${PWD##*/}\e\\\\\$(tput rc)$(tput el)${USER}@${HOSTNAME} $ '
{{< /highlight >}}

now the shell prompt is the same inside and outside tmux .

a colored prompt (tested in bash):

{{< highlight bash >}}
export PS1=$'$(tput sc)\ek${PWD##*/}\e\\\\\$(tput rc)$(tput el)[\[\e[0;32m\]\u@\h\[\e[1;34m\]\w\[\e[0m\]]\$ '
{{< /highlight >}}
