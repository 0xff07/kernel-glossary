# Kernel Glossary

Welcome to the [0xff07/kernel-glossary](https://github.com/0xff07/kernel-glossary). You can find the webpage version of this repository on:

[https://0xff07.github.io/kernel-glossary](https://0xff07.github.io/kernel-glossary).

## About

This is a place where I dump breadcrumbs of kernel subsystems, mostly conference videos. At first, I made this mostly because they make really good stories about the kernel, and it happens that I enjoy both learning Linux and listening to stories. Later there's another more practical purpose: to learn subsystems faster when the necessity arises. A more stressful way, but I always feel refreshed when I see a living subsystem face to face. It's like the magical feeling of touching snow for the first time.

This is still a work in progress, but I doubt that it'll ever be finished. It's like drawing a map for a place where the landscape changes weekly. That being said, exploration is part of the fun. My hope is that by creating this, I'm also sharing the joy of exploration with people who might also be interested in the kernel. So here we are!

## Browsing the glossary

Expand the navigation bar, or check out the [TAGS](TAGS.md) page.

All Markdown files are tagged and can be found on the TAGS page. However, not all Markdown files appear in the navigation bar. This is usually because there are currently too few entries for that subsystem, making the index too fragmented—or more likely—simply because of my laziness.

## Use as an agent skill (highly experimental)

This glossary can grow itself by being used as an agent skill, also called `kernel-glossary`.

When this `kernel-glossary` is used as an agent skill, you can instruct it to take note about a subsystem of interest, making it a personal journal for exploring the Linux Kernel. Do remind that doing so may take very large amount of tokens!

### Example usage

Here's an example:

```
> /kernel-glossary take some notes about the USB4 sideband registers
```

When taking notes, it reads the `Documentation/` directory and speculates through potential kernel code, producing 1) brief summary of given topics 2) related macros and functions with the links to the the `torvalds/linux` GitHub kernel mirror (up to the exact line numbers), and 3) sections and chapters number from relevant specification, if the underlying LLM has that knowledge.

> *When doing this example, Claude Opus 4.6 did make a few mistakes on certain details.* **AI can make mistakes. Check important info.**

### Use with Claude Code

To use it with Claude Code, simply clone it into the `.claude/skills/`:

```
# In kernel root
git clone https://github.com/0xff07/kernel-glossary.git ./.claude/skills/kernel-glossary
```

Launch Claude Code in the root directory of kernel source code:

```
# In kernel root
claude
```

Confirm the skill is loaded by running `/skills`

```
# In Claude Code
> /skills
```

If this is loaded, Claude will output something like this:

```
  Skills
  1 skill

  Project skills (.claude/skills)
  kernel-glossary · ~68 description tokens
```

You can now start using it!
