# ReScan

A [Claude Code](https://claude.ai/code) skill that reviews an untrusted repository **before you
install it**.

You found something on GitHub. The README says `git clone && npm install && npm run build`. That
install step runs code from the repo on your machine, with your permissions, before you have read a
single line of it. If the repo is hostile, the install *is* the attack.

ReScan makes Claude clone the repo read-only and audit it first.

```
You:    is this safe? https://github.com/someone/some-tool
        git clone ... && cd some-tool && pnpm install && pnpm build

Claude: [clones read-only, never installs]
        ...
        ## Bottom line
        Safe with caveats — no malware, but it is an undisclosed fork of
        <upstream>, three weeks stale, and its default config disables
        the sandbox. Details below.
```

## Install

One line — clone straight into your skills directory:

```sh
git clone https://github.com/anizum1/ReScan_ClaudeSkill.git ~/.claude/skills/rescan
```

For a single project instead, clone to `.claude/skills/rescan` inside that project.

Start a new Claude Code session and it will pick the skill up. Confirm with `/skills`.

**Requirements:** `git`, `bash`, `python3`. Nothing to build, no dependencies, works offline apart
from the clone itself.

## Use

Just ask — it triggers on its own when you paste install instructions or ask whether something is
safe:

- *"is this legit? https://github.com/x/y"*
- *"check this repo for anything malicious before I install it"*
- *"should I run this?"* (with a link)
- *"audit this npm package"*

You can also invoke it directly with `/rescan`.

## What it checks

It works outward from blast radius, so the most dangerous things get looked at first and hardest.

**1. Provenance.** Who wrote it, when, and whether it is what it claims to be. A five-thousand-file
"Initial commit" is somebody else's project with the history deleted — ReScan goes and finds the
original.

**2. Install-time execution.** Every script that runs when you install: npm/pnpm/yarn lifecycle
hooks, Python `setup.py` and local build backends, Cargo `build.rs`, gem extensions, Makefiles, git
hooks. These get read in full, because this is the only surface a drive-by compromise can use.

**3. Dependency sourcing.** Lockfile entries pointing at git URLs or raw tarballs instead of the
registry, redirected registries, and patch files that inject code into your dependencies at install
time.

**4. The usual suspects.** Credential paths (`~/.ssh`, `~/.aws`, browser cookie stores, wallets),
`curl | sh`, outbound hosts, obfuscation, base64 blobs, checked-in binaries.

**5. Upstream diff — the part that actually works.** Nobody can read 5,000 files. But most
suspicious repos are forks, and if you can find the original, `diff` collapses the job to the few
dozen files that actually changed. ReScan locates the fork point by comparing git tree hashes, which
is *proof* of byte-identity rather than an impression from skimming. On a real audit this turned
"review 5,670 files" into "review 67", and established that the sandbox and shell packages were
byte-identical to upstream.

**6. Reachability.** Scary-looking code is not automatically a finding. Before calling anything
dangerous, ReScan traces whether it actually runs in the path you would execute — a malicious-looking
default that a shipped config overrides is dormant, and gets reported as dormant.

## What it will NOT do for you

Read this part. It matters more than the list above.

- **It does not audit your dependencies.** The repo's own code is reviewed; the hundreds of packages
  it pulls from npm/PyPI are not, because reading those would mean installing them, which is the
  thing being avoided. Most real-world supply-chain attacks live in that gap.
- **It cannot read compiled binaries.** Checked-in `.node`, `.so`, `.dll`, `.exe`, `.wasm` files get
  flagged as unreadable, not cleared.
- **A clean report is not a guarantee.** It means nothing was found in the places that were looked
  at. A competent attacker writes code that survives review.
- **It is not a substitute for not running untrusted code.** The safest thing remains a VM or a
  container. ReScan lowers your risk; it does not remove it.

If a report says "safe" and you install something that owns your laptop, the skill was wrong and its
limits were real. Treat the output as a well-informed second opinion, not a verdict.

## Why it is built this way

Two design decisions do most of the work.

**Never install to inspect.** Obvious in hindsight, routinely violated in practice — including by
people who then write the post-mortem.

**Bound the problem honestly.** An audit that claims to have reviewed everything has reviewed
nothing. ReScan is explicit about scope, reports the negatives (things checked that came back clean,
which is what makes a verdict credible), and states what it could not reach. An audit that implies
more coverage than it has is the failure mode that gets someone owned.

## Contents

```
SKILL.md                                 the workflow Claude follows
references/install-execution-surface.md  what executes at install time, per ecosystem
scripts/install_hooks.py                 enumerate install-time execution
scripts/danger_sweep.sh                  credential / network / obfuscation sweep
scripts/fork_diff.sh                     fork-vs-upstream classification
```

The scripts are standalone — run them by hand without Claude if you like:

```sh
python3 scripts/install_hooks.py ./some-cloned-repo
bash scripts/danger_sweep.sh ./some-cloned-repo
```

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, improve it.

Bug reports and additional ecosystem coverage welcome, particularly install-time execution vectors
not yet covered in `references/install-execution-surface.md`.
