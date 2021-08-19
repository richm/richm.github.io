How to find play failures in Ansible logs
=========================================

At the end of a play, Ansible outputs a line like this:
```
PLAY RECAP *********************************************************************
myself                     : ok=550  changed=7    unreachable=0    failed=0    skipped=405  rescued=0    ignored=0
```
However, if you have a lot of `block/rescue` in your playbooks e.g. if you are
doing testing for errors in your playbooks, then you may have a lot of failures
in your logs:
```
PLAY RECAP *********************************************************************
myself                     : ok=531  changed=20   unreachable=0    failed=3    skipped=294  rescued=3    ignored=0
```
This is ok, as long as the number of failures matches the number of rescued.
How to use `grep` to scan through a bunch of log files?  Use `grep -P` to use
PCRE:
```
grep -P 'failed=([1-9][0-9]*) .* rescued=(?!\1)' *.log
```
That is - show me all lines in which there is a failure, but where the `failed=`
value does *not* match the `rescued=` value.  Regular `grep` or `grep -E` does
not support the negative lookahead assertion `?!`.
```
> grep -P 'failed=([1-9][0-9]*) .* rescued=(?!\1)' *.log
ANSIBLE-1.log:myself                     : ok=38   changed=10   unreachable=0    failed=1    skipped=17   rescued=0    ignored=0
ANSIBLE-2.log:myself                     : ok=42   changed=10   unreachable=0    failed=1    skipped=20   rescued=0    ignored=0
ANSIBLE-3.log:myself                     : ok=42   changed=10   unreachable=0    failed=1    skipped=20   rescued=2    ignored=0
```
This still gives you cases where we rescued more times than we failed, which
aren't really failed plays.  Ideally you could write some sort of regex to list
the lines where the `failed=` value is *greater* than the `rescued=` value, but
I think that's beyond regex capability - you'll need to use some sort of
scripting language for that.
