---
title: "kubectl alias done right"
date: 2023-04-23T21:06:00-07:00
draft: false
---

It is not uncommon to see people having an alias for `kubectl` to save few
keystrokes. It is admittedly useful, but I find it is not easy to do it
correctly.

Let's start from the basic form:

```bash
# ${HOME}/.zshrc
alias k=kubectl
```

This is what I get when I have my laptop configured by the company automation
tool. If you are satisfied with it, then you probably can close this page.

For me, however, this barely resolves the actual issue: switching contexts back
and forth is painful.

```console
$ k --context=<ctx1> ...
$ ...
$ k --context=<ctx2> ...
```

To the rescue, I wanted to leverage the tmux window name so that a context
switch becomes a window switch:

```bash
if [[ ${TMUX} ]]; then
  wname=$(tmux display-message -p '#W')
  kubectl config get-contexts ${wname} &>/dev/null
  if [[ $? -eq 0 ]]; then
    alias k="kubectl --context=${wname}"
  fi
fi
```

This becomes much easier for me because there is no more mental effort to check
the current context I'm operating on. But wait, won't you be able to solve this
by enabling `kube-ps1` zsh plugin which shows the current context name.

There are two issues with `kube-ps1`. For one, it is still painful to switch
contexts even though you get a little bit of visual feedback. However, the more
tricky issue is, you may still end up with using an incorrect context name. It
is not unusual to work on multiple panels, and if you switched context from one
panel, and then switched to another panel, you may forget that the prompt you
see is no longer correct.

Let's come back to this again:

```bash
alias k="kubectl --context=${wname}"
```
It is great that now I have an alias per tmux window, but there is a problem in
it - It doesn't work with kubectl plugin because the plugin name is supposedly
to follow `kubectl` immediately.

The solution to this situation is that we can move `--context=${wname}` to the
end:

```bash
function k() {
  kubectl $@ --context=${wname}
}
```

Here, instead of using an alias, I defined a function `k` which I can make sure
`--context=${wname}` is appended at the end so that kubectl plugins can still
work.

As a side note, I observed that people just create another alias for kubectl
plugins like `alias kshell=kubectl shell`.

I have been happy with this `k` function for a long time, until recently it
failed for me in this case:

```console
$ k exec -it <pod> -c <container> -- bash
```

This doesn't work and fails in a pretty bad way because it ends up with
executing something like below:

```console
$ kubectl exec -it <pod> -c <container> -- bash --context=${wname}
```

This command doesn't specify a context name at all, and the `--context=${wname}`
parameter actually becomes the parameter of `bash`.

The flaw is that we can't just put `--context=${wname}` at the end because there
are cases where it doesn't make sense. The right place should be right before
`--` instead.

```bash
function k() {
  # TODO: I'm pretty sure there is a better way to do this!
  KUBECTL_ARGS_1=()
  KUBECTL_ARGS_2=()
  KUBECTL_HAS_ARGS_2=false
  for p in ${@}; do
    if [[ $p == "--" && ${KUBECTL_ARGS_2} != true ]]; then
      KUBECTL_HAS_ARGS_2=true
      continue
    fi
    if [[ ${KUBECTL_HAS_ARGS_2} = true ]]; then
      KUBECTL_ARGS_2+=(${p})
    else
      KUBECTL_ARGS_1+=(${p})
    fi
  done
  if [[ ${KUBECTL_HAS_ARGS_2} = true ]]; then
    kubectl ${KUBECTL_ARGS_1} --context=${wname} -- ${KUBECTL_ARGS_2}
  else
    kubectl ${KUBECTL_ARGS_1} --context=${wname}
  fi
}
```

Maybe there are more cases where it may fail, but so far this is the version
which works for me in a lot of situations.

By sharing this little story, I think it is clear that a simple thing may not
actually be simple as you thought, and there are always complexities with it.
