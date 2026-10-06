# destiny `soul` — install the Soul Stack agent

Brings a host to the state an onboarded Soul must be in: `/etc/soul` and
`/var/lib/soul-stack` laid out, the Keeper CA pinned, `soul.yml` and the systemd unit
rendered, a checksum-verified binary in `/usr/local/bin/soul`, the one-shot bootstrap
token spent on a SoulSeed, and `soul.service` running and enabled at boot.

Public on purpose: installing the agent is what everyone deploying the platform has to
do, not something one site does. The normative description of the *result* lives in
[`keeper/internal/soulinstall`](https://github.com/souls-guild/soul-stack/blob/main/keeper/internal/soulinstall) in the engine
repository — that package describes an outcome and has no production caller; this destiny
is the procedure.

> ⚠ **Nothing checks the two against each other yet.** The guards that did — 12 Go tests
> plus 3 render-only L0 cases — lived in the engine tree and read this destiny from
> `examples/destiny/soul`. Moving the artifact into its own repository took their subject
> away, and how an out-of-tree destiny gets checked is a decision still open (a
> compatibility corpus of pinned external artifacts is the shape being considered). Until
> that lands, every invariant below is held by its comment alone. The tables name what must
> hold, not what is under test.

## ★ The artifact install and the packaged one overwrite each other, on purpose

Two install models, writing the same three files. Whichever ran last wins — that is
accepted, and it is harmless because the two are kept the same where it matters.

| | this destiny | the distribution package |
|---|---|---|
| unit | `/etc/systemd/system/soul.service` | **same path**, [`deploy/systemd/soul.service`](https://github.com/souls-guild/soul-stack/blob/main/deploy/systemd/soul.service) |
| `soul.yml` | rendered, minimal | same path, operator-installed |
| `soul.env` | rendered (for the packaged unit's sake) | same path + key |
| binary | fetched to `/usr/local/bin/soul` | `/usr/bin/soul`, owned by dpkg |
| account | whatever the unit names (root on a fresh VM) | `soul-stack:soul-stack` |
| onboarding | `soul init` in-plan, token in `env:` | manual, [deb-onboarding.md](https://github.com/souls-guild/soul-stack/blob/main/docs/operations/deb-onboarding.md) |

What keeps the overlap safe:

- **the unit bodies agree directive for directive** — the same supervision (`Type=notify`,
  watchdog, `StartLimit*` in `[Unit]`) and the same soft hardening profile, bar one directive
  (below). Keep them in step **by hand**: the check that compared the
  two files is pending (see the note above);
- **this destiny also renders `/etc/soul/soul.env`**, which its own unit never reads. The
  packaged unit's `EnvironmentFile=` carries no `-` prefix, so systemd treats the file as
  mandatory: without it, a packaged unit landing on a host this destiny installed would
  refuse to start;
- **ownership is not managed here at all** — no step sets `owner:`/`group:`. A `chown
  root:root` beside the 0700 modes would lock the packaged (non-root) agent out of its own
  configuration and its own SoulSeed, report green, and kill the host at its next reboot.
  The shell form this replaces chowned nothing either. Do not "improve" this by adding
  `owner:`.

Four differences survive deliberately — the **account**, the **binary path**,
**`EnvironmentFile=`** (this unit names the config directly; `soul.env` is rendered for the
packaged unit, above) and **`RestrictSUIDSGID`**. This unit drops the last one: the agent's children inherit it, dpkg
included, so a package whose postinst sets a setgid bit fails to install (`redis-tools`:
`chmod 2750 /var/log/redis` → "Operation not permitted"). The packaged unit still carries it.
The one crossing that still needs a second apply is a packaged unit landing on a fresh-VM
install — it names an account that host does not have.

## Using it

```yaml
- name: Install the agent
  apply:
    destiny: soul
    input:
      keeper_host:              "${ vars.keeper_endpoint_host }"
      keeper_bootstrap_port:    "${ vars.keeper_bootstrap_port }"
      keeper_event_stream_port: "${ vars.keeper_event_stream_port }"
      keeper_ca:                "${ vault(vars.keeper_ca_path + '#ca') }"
      binary_url:               "${ vars.soul_binary_url }"
      binary_sha256:            "${ 'sha256:' + vars.soul_binary_sha256 }"
      allow_private:            true      # internal artifact mirror on RFC1918
```

Applied the ordinary way — `apply:` against a host already in the registry — it converges
and reports **unchanged** on a repeat: no step writes a value that differs run to run, and
the redeem is guarded on the seed certificate.

`bootstrap_token` is omitted above because a registered host already holds an identity.
A machine with no agent yet is reached by `core.ssh.apply` instead: the Keeper delivers its
own agent over SSH to `/var/lib/soul-stack/bin/soul` (from `keeper.yml::push.soul_binary_path`),
applies this destiny with it, and hands each host its own token through `input_from:`, which
names a field of the host's entry:

```yaml
- name: Install the agent on the new VMs
  module: core.ssh.apply
  params:
    hosts: "${ register.mint.hosts }"   # from core.bootstrap.issued, reissue: true
    ssh_provider: teleport
    destiny: soul
    input:
      keeper_host:              "${ vars.keeper_endpoint_host }"
      keeper_bootstrap_port:    "${ vars.keeper_bootstrap_port }"
      keeper_event_stream_port: "${ vars.keeper_event_stream_port }"
      keeper_ca:                "${ vault(vars.keeper_ca_path + '#ca') }"
      binary_url:               "${ vars.soul_binary_url }"
      binary_sha256:            "${ 'sha256:' + vars.soul_binary_sha256 }"
    input_from:
      bootstrap_token: bootstrap_token
```

The delivered agent only executes this destiny. The unit runs the binary this destiny
fetched to `/usr/local/bin/soul`, and nothing here touches the delivered copy, which stays on
the host.

## Seven invariants, and where each one is held

None of them follows from the shape of a destiny, and each produces a host that installs
**green** and then never onboards — the failure surfaces at the onboarding barrier,
minutes later and under someone else's cause.

| | Invariant | Held by |
|---|---|---|
| 1 | The token never reaches argv — argv is visible in `ps`, `audit.log` and journald **on the host** | `env: SOUL_BOOTSTRAP_TOKEN` on the redeem step — never `args:` |
| 2 | The token is **spent** by `soul init`, behind a guard on the seed certificate (it is single-use) | `creates: ${ vars.seed_cert }` on the redeem step |
| 3 | `enabled: true`, not a bare start — else the unit does not survive a reboot | `core.service.running` with `enabled: true`, and no restart step anywhere |
| 4 | The binary is renamed into place, never written over (ETXTBSY) | `core.url.fetched` writes a temp beside the target and renames |
| 5 | The checksum is verified **before** chmod and **before** the rename | `core.url.fetched` verifies before chmod and before rename; `binary_sha256` is `required: true` with no default |
| 6 | Two **different** ports in `soul.yml`, and `paths.seed` is the directory (2)'s guard reads | two separately named inputs; `vars.seed_cert` derived from `vars.seed_dir` |
| 7 | Directory modes named explicitly (`install -d -m` gave created parents 0755 regardless) | `tasks/layout.yml`, one explicit `mode:` per directory |

## Three deliberate departures

- **Nothing restarts the agent.** Applied the ordinary way the plan is executed *by*
  `soul.service`, so a restart step would kill the run that issued it. All three artifacts
  are affected, not just the config: a re-rendered `soul.yml`, a re-rendered unit **and a
  replaced binary** take effect at the next restart. The binary is the one that surprises —
  `core.url.fetched` renames the new file into place and reports `changed`, while the live
  process keeps running its old inode for as long as it lives. **So this destiny cannot
  upgrade a running agent on its own:** it puts the bytes in place, and the caller restarts
  the unit afterwards, outside this run.
- **No systemd-hardening block**, against
  [production-conventions §3](https://github.com/souls-guild/soul-stack/blob/main/docs/destiny/production-conventions.md). This
  process *is* the configuration-management agent: `ProtectSystem=strict` or an empty
  `CapabilityBoundingSet` would break it wholesale, and a sandbox wide enough to keep it
  working would protect nothing. Its confinement is Keeper-side RBAC.
- **Three directory modes are stricter than the blueprint's** (`/etc/soul` 0700 vs 0755,
  `/etc/soul/tls` 0700 vs 0600, the seed directory 0700 vs 0755). The stricter set is the
  one a live install ran with. The guard compares paths against the blueprint and
  deliberately not modes.

⚠ One blueprint capability has no equivalent here: `soul_binary_ca: keeper` (pin the
binary download to the Keeper CA). `core.url.fetched` verifies against the system trust
store or not at all — a mirror behind a private CA needs that CA in the image's trust
store. The bootstrap channel is unaffected; it is always pinned to `/etc/soul/tls/keeper-ca.pem`.

## Tests

`_trial/` carries three render-only L0 cases — the whole task plan, the `allow_private`
branch an internal mirror needs, and the refusal of a `keeper_ca` that is not a
certificate. They run under `soul-trial`, which ships with the engine:

```
soul-trial run <path to this checkout>/_trial
```

There is no CI here yet: see the note at the top.
