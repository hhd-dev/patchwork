# Linux kernel AI agent instructions

Template for AI coding assistants working on the Linux kernel.
See Documentation/process/coding-assistants.rst for setup instructions.

Customize this file for your agent (see Assisted-by: below) and workflow.

Then, remove this preamble and place the file in the root of your kernel tree.

## Working with the kernel

 - Only use ASCII characters in source files.
 - Do not use double spaces after periods.
 - Before adding a year to copyright, fetch the current year with `date`.
 - Use the current user for module author/maintainer/copyright fields.
   If uncertain, fetch with `git config user.name` and `git config user.email`.
 - For sysfs docs, use `date` and `make kernelversion` to get the correct
   information. Target the next kernel version (e.g. 7.0-rcN means 7.1) and
   estimated merge month. If either is uncertain (e.g. rc is high),
   ASK THE USER.

### Building

Use `make M=<module-dir> <CONFIG_FLAGS>` for out-of-tree module builds.
Always verify the module compiles after changes.

## Commit messages and changelogs

Before sign-offs, include an Assisted-by tag if you made substantial changes
to the commit message or commit contents:

  Assisted-by: AGENT_NAME:MODEL_VERSION

See Documentation/process/coding-assistants.rst for the full tag format.

Add a Signed-off-by for the user. If you are uncertain about the sign-off
identity, run `git config user.name` and `git config user.email`.

### Single patch vs. series

 1) Single patch: USE NO COVER LETTER. Add the changelog as a trailer
    to the commit message itself.
 2) Patch series: USE A COVER LETTER (branch description). Subject on the
    first line, then the description. After the trailer, append the changelog.

### Changelog format

Begin with the trailer, then:
```
---

Changes for <n>:
  - Make change 1
  - Make change 2
...

Changes for 1:
...

V<n-1>: lore link
...
V1: lore link
```

If you do not have a version link, ASK THE USER. Search is unreliable, lore
has bot protection. When asking, include a commit message or the description
subject and a link to https://lore.kernel.org so they can fetch it.

## Editing a series

Orient yourself first: `git log --oneline <base>..HEAD`

### Code changes

For single patches:
`git commit --amend --no-edit`

For series:
`git commit --fixup=<commit> && GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <base>`

For `<commit>` and `<base>` you have two options.
 1) Use the commit hash and base branch commit
 2) If you know the target offset, use `HEAD~N` for the commit and `HEAD~M` for
    the base, where M = N + 2 to account for one fixup commit, or more for more.

### Patch re-ordering and flexible rebasing

To inspect the rebase todo without executing it, save and abort:
`GIT_SEQUENCE_EDITOR='sh -c "grep -v ^# \$1 > /tmp/todo.tmp; false" _' git rebase -i <base> 2>/dev/null || true`

Edit `/tmp/todo.tmp` (reorder, drop, or modify lines), then apply:
`GIT_SEQUENCE_EDITOR='cp /tmp/todo.tmp' git rebase -i <base>`

### Commit messages and branch descriptions

To edit without an interactive editor, use a temp file:

Commit message:
 1) `git log -1 --format=%B <commit> > /tmp/commitmsg.tmp`
 2) Edit `/tmp/commitmsg.tmp`
 3) For HEAD: `git commit --amend -F /tmp/commitmsg.tmp`
    For older commits:
    `GIT_SEQUENCE_EDITOR="sed -i '1s/pick/edit/'" git rebase -i <commit>~1`
    then `git commit --amend -F /tmp/commitmsg.tmp && git rebase --continue`

Branch description (cover letter source):
 1) `git config branch.<name>.description > /tmp/coverletter.tmp`
 2) Edit `/tmp/coverletter.tmp`
 3) `git config branch.<name>.description "$(cat /tmp/coverletter.tmp)"`

This avoids eating context and introducing errors from full rewrites.

### Rebasing

If the user asks for a different base:
`git rebase --onto <new-base> <old-base>`
Fix any conflicts. If appropriate, regenerate patches and re-run checkpatch.

### Splitting patches

First, read the patch and identify logical breakpoints. Ask the user to
verify. Then:
 1) Create branch `<name>-split` starting at `<commit>~1`
 2) Write the commits one by one with appropriate messages
 3) Verify `git diff <name> <name>-split` is empty and checkpatch is clean
 4) Present results and offer to reset:
    `git branch -f <name> <name>-split && git branch -d <name>-split`
    Only do the reset if the user explicitly confirms.

## Generating patches

Generate with `git format-patch`. Two forms:

- With cover letter:
`rm -rf patches/<name> && git format-patch --cover-letter --base <base> --cover-from-description=subject <ver> -o patches/<name> <base> && ./scripts/checkpatch.pl --strict patches/<name>/* || true`
- Without cover letter:
`rm -rf patches/<name> && git format-patch --base <base> <ver> -o patches/<name> <base> && ./scripts/checkpatch.pl --strict patches/<name>/* || true`

`<name>` is the branch name. If the user is on `master` or detached HEAD,
STOP and ask them to create a named branch.

`<base>` is the base branch/commit. You can use `HEAD~N` if you know the
number of commits.

`<ver>` is `-vN` for normal series, `--subject-prefix="RFC vN"` for RFC.
N is the version number starting at 1. ALWAYS INCLUDE IT. ASK THE USER
if the series is RFC and the version N if unsure.

After generating, list the patch file paths so the user can review them.

If you have started generating patches, for follow-up changes, begin fixing
the checkpatch output before presenting the files to the user. Ignore
CHECK-level messages unless they point to a real issue.

## Sending to kernel / git send-email
If the user asks for a command to send the generated patches to the kernel,
first, find recipients: `./scripts/get_maintainer.pl patches/<name>/*`, then
split them into two groups, the emails that should be included and those that
should not. For the included emails, sort them by to, cc, then kernel mailing
lists last.

Begin with a preamble that includes two sections, one for the included
people/emails and one for the excluded. For each person/email, include a
justification about why or why not they are included
(why to? why cc? why excluded?).

Then, provide the final command to send the emails with the following syntax:
```
git send-email --confirm=always --thread --no-chain-reply-to \
 --to=... --cc=... \
 patches/<name>/
```

NEVER SEND EMAIL YOURSELF. Only supply the command so the user can run it.

## Reviewing patches

Patches must be re-generated before reviewing. Read each patch file, not
the raw commits. Nits apply only to added code (e.g., it is fine to use
existing global vars). Keep the thought process to yourself and only
present issues to the user.

Here are some common issues to check for (also avoid these when writing code):
 - Does each commit message accurately describe its patch?
 - Does the cover letter reflect what the series actually does?
 - Is the changelog filled in for every version?
 - Are Signed-off-by, Assisted-by, and other tags present and correct?
 - Is the patch order logical (dependencies before dependents,
   excl. documentation which goes first)?

 - Declarations: Reverse christmas tree (order local variable declarations
   longest line to shortest), all declarations at start of function.
   Exception: __free(kfree) vars which get allocated after an early exit,
   prefer those to manually freeing the pointer.
 - Use `dev_err()` etc instead of `pr_*()` when a `device` is available.
 - No file-scope variables except in `module_init`/`module_exit` paths.
 - Use `devm_*` functions which handle teardowns.
 - Alphabetical order on imports, struct pad alignment, etc.
