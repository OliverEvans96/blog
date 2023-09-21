# Commit signing in emacs

I've been using emacs for a while and been interested in `magit`, but I haven't been able create any commits due to GPG errors.

## The setup

I have git configured to auto-sign commits by setting `commit.gpgSign = true`. And I have `gpg-agent` configured to use `pinentry-curses` to read my password since I normally commit from the terminal.

However, when I've tried to create a commit from `magit`, I was met with the following error:
```
gpg: signing failed: Inappropriate ioctl for device
```

After several failed attempts, I'm happy to report that I've finally figured out the issue and a solution.

## What didn't work

I initially followed the advice in [this vxlabs blog post](https://vxlabs.com/2021/03/21/gnupg-pinentry-via-the-emacs-minibuffer/)/[reddit thread](https://www.reddit.com/r/emacs/comments/m9u3yl/gnupg_pinentry_via_the_emacs_minibuffer/).
It seemed promising, but it didn't fix my issue.

I also found [this reddit thread](https://www.reddit.com/r/emacs/comments/qjrm7z/gpgagent_pinentry_and_loopback_how_to_make_it_work/), which didn't have a full solution either.

I also saw some references to [pinentry.el](https://elpa.gnu.org/packages/pinentry.html), which is a package that allows _all_ pinentry requests to be forwarded to emacs, which is not what I want. I still want to be able to sign commits from the terminal without going through emacs.

I also saw some suggestions to add `allow-emacs-pinentry` to `gpg-agent.conf`, but I didn't find it necessary for my final solution.
I think it's probably relevant when using `pinentry.el`.

I also saw some suggestions to set the environment variable `GPG_TTY=$(tty)`.
I didn't try this explicitly, but I don't think it would have accomplished what I was after.

I also saw some suggestions just to use `pinentry-gtk`, which I didn't want to do - I prefer using `pinentry-curses` from the terminal.

## The problem

The vxlabs blog post described how to configure `epa`/`epg` to use the `loopback` pinentry mode to allow emacs to read the GPG password.
It took quite a bit of reading through `magit` source code and tinkering with `edebug` to realize why their solution didn't work for me.
Finally, I realized that `magit` doesn't actually use `epg` to sign the commit.
It just shells out to the `git` binary, which in turn communicates with `gpg-agent`, so setting `epg-pinentry-mode` had no effect.

## The solution

After I managed to get it working, I realized that `magthe0` from the second reddit thread linked above was on the right track: just unlock `gpg-agent` from emacs right before commiting.

See, the suggestion in the vxlabs blog post _does_ actually work.
If I manually call `epa-sign-file`, it can read my password from a minibuffer correctly.
And by doing so, `gpg-agent` will remember my password for a few minutes.
So by automatically signing a string before commiting from magit, `gpg-agent` will already be happy when the `git` binary tries to authenticate.
We can do this automatically via elisp's [advice-add](https://www.gnu.org/software/emacs/manual/html_node/elisp/Advising-Functions.html)

So my final solution looks like this:

Add the following to my emacs config:
```elisp
;; pinentry config for gpg signing from within emacs
;; See https://vxlabs.com/2021/03/21/gnupg-pinentry-via-the-emacs-minibuffer/
(setq epg-pinentry-mode 'loopback)

(defun unlock-gpg-key (&rest _args)
  "Unlock the GPG key by attempting to sign something. Ignore all arguments."
  (let
      ((context (epg-make-context epa-protocol))
       (msg "test"))
    (epg-sign-string context msg)))

;; magit doesn't actually respect epg-pinentry-mode. It just calls the git binary,
;; which attempts to call pinentry-curses itself. This fails because it has no tty.
;; Therefore, we first attempt an epg signature to unlock the key from within emacs.
;; Then, gpg-agent will let magit sign the commit from the git binary :)
(advice-add 'magit-commit-create :before #'unlock-gpg-key)
```

And add the following to `~/.gnupg/gpg-agent.conf`:
```
allow-loopback-pinentry
```

Happy committing :)

## See also
- https://www.gnupg.org/documentation/manuals/gnupg/Agent-OPTION.html
- https://www.reddit.com/r/emacs/comments/pgas3e/magit_not_showing_gpg_password_prompt_when/
