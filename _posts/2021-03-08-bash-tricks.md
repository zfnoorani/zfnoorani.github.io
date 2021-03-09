---
layout: single
classes: wide
title:  "Some Bash Scripting Tricks"
date:   2021-03-08 12:00:00 -0600
categories: bash tricks
---

Bash is arcane and ugly.
It's tough to write maintainable, reliable, and clean scripts for complex use cases.
Here are some of the coolest tools and tricks I've picked up through work or mentorship to help do that.

I'll also have some information on integrating these tools with [Kakoune](https://kakoune.org/), my text editor of choice.
It's fantastic, but I'll save the gushing for another time.

### Shellcheck
[Shellcheck](https://github.com/koalaman/shellcheck) is a `bash` and `sh` linter.
It's aggressive, but it's almost always on point.
Make it a part of your workflow and CI pipeline, and never look back.

[shellcheck.kak](https://github.com/whereswaldon/shellcheck.kak) works great for Kakoune integration.

### shfmt
[shfmt](https://github.com/mvdan/sh) is a shell formatter.
It's in Go, so you can use `GO111MODULE=on go get mvdan.cc/sh/v3/cmd/shfmt` to install it to your `GOPATH`.

Set your editor to format on save, and formatting scripts is no longer a concern.
For Kakoune:
```
hook global WinSetOption filetype=sh %{
    set window formatcmd 'shfmt'
    hook buffer BufWritePre .* %{format}
}
```

### Strict Mode
[Bash strict mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/) boils down to throwing this at the top of scripts:
```bash
set -euo pipefail
IFS=$'\n\t'
```

The most important is `set -e`, which tells Bash to immediately exit the script on any failure, rather than going runaway.

I highly encourage reading the linked article front to back.
Aaron Maxwell does a better job of explaining how it works, and what common pitfalls you might encounter are.
It's clear he's a shell wizard.

### Trap
Maxwell, master of dark shell arts, also provides a discussion about [Bash exit traps](http://redsymbol.net/articles/bash-exit-traps/).
This technique is an analogue to `defer` from Go, or `try`-`catch` in C++-like languages.
You can execute cleanup or recovery code whenever a script exits.

My favorite use is for sending a Slack notification for a CI script that indicates success or failure (to workaround our CI tool's native Slack functionality being broken):
```bash
set -euo pipefail

notify() {
	status=$?
	if [ $status -eq 0 ]; then
		send-notification "yay :tada:"
	else
		send-notification "there is doom and gloom when things go boom"
	fi
}
trap notify EXIT

sudo some commands that could
sudo fail that you need to know about
sudo so you may respond
```

What Maxwell lacks in a site that has `https` enabled, he more than makes up for in awesome shell tricks.

### set -x
If you need to see exactly what's going on in a script, throw `set -x` at the top of it.
Your script will print out every command being executed.

You can even do this in your interactive shell, but this clogs your terminal since the commands from `.bashrc` will print out.
Use `set +x` to undo it.

### Summary

While I'm just an apprentice of arcane shell magic, these resources have helped me create beauty out of the mess that is Bash.
Hopefully they can be of help to you too.
