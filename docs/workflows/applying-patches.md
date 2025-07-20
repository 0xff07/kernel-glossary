---
topics: tooling
tags: 
    - "tooling"
---

# Applying Patches

This is specifically for applying patches from a `.patch` or a `.mbox`, which is a less ideal way especially if patches has landed in some git trees.

Easiest way is to use `b4`:

```sh
b4 shazam ${LINK_FROM_LORE_KERNEL_ORG}
```

For example:

```
b4 shazam 20250613-for-upstream-not-instantiate-spd5118-v2-1-cf456ed9b587@canonical.com
```

Another way would be downloading the `.mbox` from a mailing list thread and use `git am`.

## References

### Videos

### Links

1. [git-am - Apply a series of patches from a mailbox](https://git-scm.com/docs/git-am)
2. [How to apply patches from the Linux Kernel Mailing List](https://blog.reds.ch/?p=1814)
3. [Applying Patches To The Linux Kernel (obsolete)](https://www.kernel.org/doc/html/latest/process/applying-patches.html)
4. [am,shazam: retrieving and applying patches](https://b4.docs.kernel.org/en/latest/maintainer/am-shazam.html) from B4 end-used doc.