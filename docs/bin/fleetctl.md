# fleetctl

Run work on machines you do not all administer, through a private inventory,
under one admission policy. One file of Python 3 and the standard library
driving `ssh`, `scp` and `rsync` — no daemon, nothing installed on the remote.

`fleetctl help` is the manual: `overview verbs roles reach config secrets jobs
examples`. This page is the tour.

## Why

The fleet grows in variety faster than in size — a Raspberry Pi whose only job
is to be jumped through, a bare-metal GPU box behind it, a Slurm site reached
through a login node. So most of fleetctl's value is in what it refuses.
Running `python train.py` on a login node is not a mistake it lets you make by
accident.

## The pieces

- **Target** — one SSH-reachable surface. A Slurm compute node is *not* a
  target; node types are queue presets on the site's protocol.
- **Pool** — an ordered set of targets. Selection takes the first enabled one.
- **Protocol** — a site: whether compute is scheduled there, and its queues.
- **Profile** — how work runs: interpreter, and submit/status/logs/cancel.
- **Project binding** — a local directory mapped to a target and a remote root,
  so a bound directory needs no `--target`.
- **Secret** — connection material, kept out of the inventory and out of git.

| path | holds |
| --- | --- |
| `~/.config/fleet/` | `targets.d/` `pools.d/` `protocols.d/` `profiles.d/` `secrets/` `config.toml` `projects.toml` |
| `~/.local/state/fleet/` | private `known_hosts`, staged scripts |
| `~/.cache/fleet/mux/` | SSH control sockets |

`$FLEET_CONFIG_HOME`, `$FLEET_STATE_HOME` and `$FLEET_CACHE_HOME` move all
three; `--config-home`, `--state-home` and `--cache-home` do it for one run.

An unknown key in any of those files is fatal, and the error names the file, the
key and a likely correction — because the default that fills in for a
misspelling is always the permissive one.

## Roles decide what may be done

Every target declares a `role`. A role is a taint, not a label: it repels work,
and `--admin` is the toleration — permission, never a recommendation.

```
role         ssh   exec  script sync  submit job
bridge       yes   admin no     admin no     no
login        yes   admin admin  yes   yes    yes
compute      yes   yes   yes    yes   yes    yes
workstation  yes   yes   yes    yes   yes    no
storage      yes   admin no     yes   no     no

yes = allowed    admin = needs --admin    no = refused
```

`fleetctl help roles` prints that table, generated from the same constant the
admission check reads, so the two cannot drift apart.

There is deliberately no role meaning "scheduler node where direct execution is
fine", so the state this tool exists to prevent cannot be spelled in the config
at all. A refusal names the alternative — *run compute through `fleetctl
submit`* — and no flag lifts it. `fleetctl explain <target>` reports every cell
for one host, where a refusal is the answer rather than an error.

Role and protocol must agree, checked at load: `role = "login"` on a protocol
that runs work directly is refused, and so is any other role on a
scheduler-backed one. A target with no `role` gets the conservative reading of
its protocol, and `doctor` names it.

## Reach: direct first, the bridge as fallback

How a host is reached is inventory, not a credential.

```toml
# targets.d/rtx1.toml
reach = ["direct", "via:numpi"]
```

Ordered, and the order is the point: direct is the route taken, the bridge is
the declared fallback. Every `via:` must name an enabled target whose `role` is
`bridge`; self-bridges and cycles are refused at load, once per fleet rather
than once per dependent.

A hop is a full fleetctl connection to the bridge — its own secret, and the same
private `known_hosts`, `IdentitiesOnly`, timeouts and control socket the
destination gets — so it inherits nothing from `~/.ssh/config`, and it chains.
Using a bridge is transport, not execution, so it takes no admission check;
`exec` on that same bridge is still refused. A password-authenticated bridge is
refused as a hop, because sshpass hands one password to the whole process tree.

`--route direct` or `--route via:<bridge>` pins a *declared* route; fleetctl
will not invent one. `fleetctl list --format topology` prints the graph.

Current limit: the first declared route is the one used — nothing degrades
automatically yet — and both routes to one host share a control socket, because
OpenSSH's `%C` does not hash the ProxyCommand.

## Start

```bash
fleetctl init
fleetctl import-ssh <alias> --name <target> --role workstation
fleetctl doctor            # must be clean before anything else
fleetctl doctor --probe    # also checks remote python3, which exec needs
```

`migrate-config` rewrites an older inventory in place: it fills in `role`, makes
`protocol` explicit, and lifts a secret's `proxy_jump` into `reach` with
`direct` in front of it.

## Daily

```bash
fleetctl smoke <target>                            # transport, then workdir
fleetctl exec <target> -- python3 -c 'print(1)'    # after `--` is verbatim
fleetctl script ./setup.sh --target <target> -- --flag value
fleetctl sync push .                               # from a bound directory
fleetctl submit train.sh --target <target> --queue <preset>
fleetctl job status <id> --target <target>
```

`--dry-run` prints the fully resolved plan and exits 0, claiming nothing about
remote state — except for `sync`, where it runs `rsync --dry-run
--itemize-changes` and returns rsync's own code.

`sync --delete` refuses a home or root directory outright, and refuses the
target's own workdir, or any pull, without `--force`.

`submit` wraps your script in a scheduler preamble built from the queue preset;
`--native-batch` sends your file unchanged instead. Nothing is submitted unless
a job id could be reported back.

## Secrets

`host`, `user`, `port`, `auth`, `identity_file`, `password` and the SSH tuning
knobs live in a secret, never in a target file — putting one in a target is a
refusal with that explanation. Three backends: a private TOML file, a `pass`
entry, or an inline table. Resolution is lazy.

```bash
fleetctl show <target> --secret               # host and user both redacted
fleetctl show <target> --secret --sensitive   # ask for the real values
fleetctl seed-pass --pass-prefix infra/fleet
fleetctl deploy-config --pass-prefix infra/fleet --doctor
```

`deploy-config` renders a fresh tree beside the live one, validates it, and only
then swaps — keeping one previous generation, and never clearing `secrets/`
unless every target is pass-backed and it can refill them. `./setup.sh` runs it
when the `infra/fleet` pass prefix exists locally.

Files are written `0600` and directories `0700`; `doctor` treats a group- or
world-readable secret as a problem, not a warning. Prefer keys — and above all
on the bridge, which every other host is reached through.

## Exit status

`0` success · `1` `doctor` found problems · `2` any refusal, and any usage error
· `130` interrupted. Anything else is the remote command's own.

## Shell integration

`fssh` in Zsh merges classic SSH host discovery with `fleetctl list --format
ssh-hosts`; picking a `fleet:*` entry routes through `fleetctl ssh`.
