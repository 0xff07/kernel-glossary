# kmemo

This is an experimental, LLM-generated version of [0xff07/kernel-glossary](https://github.com/0xff07/kernel-glossary).

> WARNING: due to its LLM-generated nature, DO NOT attempt to upstream any docs from this project, unless you are already a renowned expert in that area.

This agent skill creates ad-hoc cross references for topics in various kernel subsystems. Whenever I'm reading certain specs and wondering "how does Linux kernel handle this?", I use this as a fast scout before digging into it. This repo also stores LLM interesting LLM artifacts created in this process (e.g. fancy ascii diagrams).

> WARNING: due to its LLM-generated nature, DO NOT attempt to upstream any docs from this project, unless you are already a renowned expert in that area.

A human reader should treat this agent skill as a detector. It relies on a human reader's judgement to decide what is valid.

> WARNING: due to its LLM-generated nature, DO NOT attempt to upstream any docs from this project, unless you are already a renowned expert in that area.

The output of this skill varies greatly between LLM models. Use it with care. 

> WARNING: due to this nature, never attempt to upstream any docs from this project, unless you are already a renowned expert in that area.

**AI CAN MAKE MISTAKES, SO VERIFY.**

## Setup

### Prerequisites

Set up [`facebookexperimental/semcode`](https://github.com/facebookexperimental/semcode). You need both the binaries and MCP.

After installation, create your first semcode DB by indexing a range of commits. For example, to index all commits from v6.17 to v7.0:

```
semcode-index --git 'v6.17..v7.0' --db-threads $(nproc)
```

Optionally you may want to index mailing list archive with `semcode-index --lore`, for example:

```
semcode-index --lore lkml
```

That way, you can reference the mailing list archive locally from LLM. See [`lore.md`](https://github.com/facebookexperimental/semcode/blob/main/docs/lore.md) for more information.

### Use with Claude Code

To use it with Claude Code, simply clone it into the `.claude/skills/`:

```
# In kernel root
git clone https://github.com/0xff07/kernel-glossary.git -b kmemo ./.claude/skills/kmemo
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
  kmemo · ~68 description tokens
```

You can now start using it!

