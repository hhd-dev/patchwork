# Linux Kernel Additional AI agent skills
The file ./AGENTS.md contains a generic framework for kernel developers to base
their workflow on. However, different developers have different development
and reviewing needs. It cannot satisfy everyone and doing so would cause context
rot. This file provides additional subsystem and workflow specific skills that
can be used to customize the framework in ./AGENTS.md.

This document is formatted as follows:

```
## Skill name
> Single line description of skill
> - Contact: John Doe <jdoe@example.com>,
>            Alice Smith <asmith@example.com>
> - Date: 2026-01-01
>
> Multiline description of skill...
>
> Example prompt:
> > ...

The skill contents.
```

## Reviewer's Friend
> Review section addendum for subsection maintainers to sanity check patches
> - Contact: Antheas Kapenekakis <lkml@antheas.dev>
> - Date: 2026-04-01
>
> This skill provides additional instructions for agents to review a specific
> series for common issues without modifying the current git tree. The agent
> reviews the current series with its replies, runs checkpatch, and only
> reports unaddressed issues. For new issues, the agent addresses them inline
> and opens your editor in diff mode.
>
> Replace `<editor>` with your editor.
>
> Example prompt:
> > Review the series at https://lore.kernel.org/... for potential issues.

If the user provides you with the URL of a series (<url>) and asks you to review
it, run the following command to fetch the current series and its replies:

```
./scripts/agent/lore-replies --untruncated <url> | tee /tmp/series /tmp/comments | \
  awk '/^====/{l1=$0; if ((getline l2)>0 && (getline l3)>0 && l3~/^====/) {print ""; print ""; print ""; next} print l1; print l2; print l3; next} /^>/{print ""; next} /^----/{print ""; next} {print}' | \
  ./scripts/checkpatch.pl --strict --ignore=BAD_SIGN_OFF,NO_AUTHOR_SIGN_OFF /dev/stdin > /tmp/checkpatch || true
```

Read /tmp/comments and /tmp/checkpatch in full. Find potential unaddressed
issues in /tmp/comments and unfixed /tmp/checkpatch warnings (ignore Duplicate
signature warnings). After you reason about the issues, edit /tmp/comments and
comment about the issues inline.

Use the following format for inline comments:
```
------------------------------------------------------------------------
> Agent:
>   Here X is not checked properly
------------------------------------------------------------------------
```

Or reply to an existing comment:
```
------------------------------------------------------------------------
> johnd:
>   Here X is not checked properly
>   > Agent:
>   >   I think it is checked properly because of Y
------------------------------------------------------------------------
```

Finally, run `<editor> -d /tmp/series /tmp/comments` to open the editor in diff
mode for the user. Then, present a summary of the new issues you found, checkpatch
warnings, and issues that have already been addressed by other reviewers and by
whom.