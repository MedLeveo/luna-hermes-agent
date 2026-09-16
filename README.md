# Luna — a pregnancy companion you text

Tell it once how far along you are and it never asks again. It raises the exams
the week calls for, explains the ones you photograph, keeps the questions you
meant to ask, briefs you the day before your appointment — and when you describe
something that should not wait, it does not reassure you. It tells you to go.

A [Plow](https://plow.co) agent: a container that reaches its owner through
Plow Chat and nothing else.

## Why

An obstetrician sees you for fifteen minutes a month. The other twenty-nine days
you are alone with a search engine at 3am, and the results reach you before
anyone explains them — a portal posts your labs and you read them at 11pm, days
before your visit.

Luna is not a second opinion. It is the continuity between visits: it holds the
week, it holds the paperwork, and it holds the list of things you keep
forgetting to ask. Everything clinical goes back to the doctor.

## What it does

- **Knows the week, always.** "I'm 22 weeks" is enough. Every date it states is
  computed, never estimated.
- **Raises what the week calls for.** Anatomy scan, glucose test, Tdap, group B
  strep — and it stops raising anything you tell it you have had.
- **Writes to you every morning.** Where you are, how far there is to go, at
  most one thing that matters today, and a real question about how you feel.
- **Briefs you before the appointment.** The day before: how far along you'll
  be, what that visit usually covers, the questions you collected, and the
  symptoms you mentioned in between — with when they started.
- **Reads what you photograph.** What the test is and what it is for, in plain
  language, filed where you can find it again. What it *means* stays your
  doctor's to say.
- **Remembers the deadlines nobody owns.** Leave dates and FMLA notice,
  choosing a pediatrician, hospital pre-registration, adding the baby to your
  insurance. Not medical, not on anyone's checklist, expensive to miss.
- **Finds you somewhere to go,** and hands the appointment back: an `sms:` link
  that opens Messages with the message already written, so you send it yourself.
- **Answers in your language,** whichever one you wrote in.

Nothing here asks you to connect an account. No OAuth, no portal login, no app
to install — you text, and that is the entire interface. A capability that would
need you to authorise something does not belong here.

## What it will not do

It does not diagnose, prescribe, adjust medication, or tell you a result is
normal. Asked something clinical it has exactly three answers: *this is common
at this stage* (and mention it at your next visit), *I have added this to your
questions list*, or *this needs care now*. There is no fourth.

One rule sits above every other, in [`image/seed/SOUL.md`](image/seed/SOUL.md):
bleeding, leaking fluid, severe pain, fever, reduced fetal movement, visual
changes, sudden swelling — it stops, and it says go. No reassurance, no
clarifying question first.

> Erring toward reassurance kills people. Erring toward "get it checked" costs
> someone an afternoon. Always choose the afternoon.

## Running it

You need Docker and Python 3. You do **not** need to be pregnant to try it:
tell it "I'm 22 weeks" and everything works from there — the morning message,
the exam it raises next, reading a photographed lab, the emergency rule.

A Mac running [Plow Latch](https://github.com/plow-pbc/latch) is optional, and
only for the one thing that needs a browser: finding somewhere to go.

Get the credential CLI:

```sh
git clone https://github.com/plow-pbc/plow-agents.git
export PATH="$PWD/plow-agents/bin:$PATH"
```

Then this agent:

```sh
git clone https://github.com/MedLeveo/luna-hermes-agent.git
cd luna-hermes-agent
```

Log in — it prints an activation phrase to text from the handset that owns the
account, and that handset is the identity. Add `--new-line` only if the account
has no assistant line yet. Pick a line marked `free`:

```sh
plow-agents login
plow-agents lines
plow-agents mint ln_xxx
```

`mint` writes `./plow-credentials`, and it has to run **before** the first
`up`: Docker creates a missing bind source as a directory, and the agent then
finds a directory where its credential should be.

**Set your timezone.** The container is UTC otherwise, and the morning message
is scheduled for 08:00 — which would reach you at four in the morning. One line
in `.env`, beside `compose.yml`:

```sh
echo 'TZ=America/New_York' > .env
```

Then:

```sh
docker compose up --build -d
```

The first build takes a few minutes. Watch `docker compose logs -f agent`
until `plow-init: configured ... as cht_` appears, then text the number that
line answers on and say hello.

No API key: inference comes from Plow, and the credential `mint` writes is the
only one the container ever sees.

When you are done, retire the agent and remove its memory:

```sh
plow-agents revoke
docker compose down -v
```

`docker compose down` on its own keeps the conversation and her record. `down
-v` is also what an edited `SOUL.md` needs, since the identity is read when a
session is created and not afterwards.

## Layout

This repository is a variant of
[`plow-pbc/plow-hermes-agent`](https://github.com/plow-pbc/plow-hermes-agent):
the boot layer is vendored from it, and what makes this agent itself is the
seed.

| path | what it is |
|---|---|
| `compose.yml` | how it is run: the credential mount, the home volume, the timezone |
| `image/seed/SOUL.md` | who the agent is, and the rules it may not break |
| `image/seed/skills/health/prenatal/` | the skill: when to run what, and the scripts |
| `image/seed/config.yaml` | the reference config plus this agent's overrides |
| `image/crons/crons.json` | the one scheduled job, as data |
| `image/s6-overlay/s6-rc.d/luna-crons/` | converges that job onto the scheduler at boot |
| `image/cont-init.d/00-luna-home` | restores the home on a host whose volume does not inherit image content |
| `image/cont-init.d/02-luna-credentials` | writes the credential from the environment on a host with no bind mount |
| `image/cont-init.d/02-luna-own` | gives the dotenv back to root, for a host where root is not privileged |
| `Dockerfile`, the rest of `image/` | vendored boot layer |

## The dates are code, not prompt

A model that estimates a due date will eventually estimate it wrong, and every
reminder after that is wrong with it.
[`prenatal.py`](image/seed/skills/health/prenatal/scripts/prenatal.py) owns the
arithmetic — gestational age, Naegele's rule, which milestones the current week
falls in, what is overdue and what she has already had — and `SOUL.md` forbids
doing it any other way. It refuses a future date, and one over 300 days old,
rather than recording a number that would be wrong for the whole pregnancy.

The same split runs through the rest: the model decides what to say, and a
script owns anything where being subtly wrong is invisible — filing a document
so two never collide, URL-encoding a link she will tap without reading.

## Running it somewhere without a bind mount

`plow-init` reads its credential only from `/var/lib/plow/credentials`, and
drops the process environment as a source on purpose: an environment variable
must not be able to outrank the credential the image was given. A PaaS has no
bind mount, so `image/cont-init.d/02-luna-credentials` writes that file from
`PLOW_API_BASE` and `PLOW_AGENT_TOKEN` — and **never** overwrites one that is
already there, which keeps a mounted credential authoritative.

The capability set is the sharpest difference. A PaaS drops
`CAP_DAC_OVERRIDE`, so root obeys file modes like anyone else — and the boot
order then matters in a way it never does on a normal host. The base image's
`01-hermes-setup` chowns the whole home to uid 10000; `plow-init` then runs as
root and writes `$HERMES_HOME/.env`. It hardens the directory, `skills/` and
`SOUL.md` first, but not the dotenv, so the dotenv is still uid 10000 mode
0640 — and root, being neither its owner nor in its group, gets EACCES.
`02-luna-own` runs after that hook and hands the dotenv back to `root:hermes
0640`. On a host where root keeps the capability the same write just succeeds,
which is why the reference image needs none of this.

The volume is the other difference. Compose mounts a Docker *named* volume at
the home, and Docker populates an empty one from the image — seed, ownership and
modes included. A PaaS mounts raw block storage: it arrives empty and
root-owned, shadowing the seed and leaving a home the agent cannot write, which
crash-loops the container on `plow-init`. `00-luna-home` seeds the gaps from a
baked copy kept outside the home and reapplies the ownership the Dockerfile
sets.

That is how this agent actually runs: the container is hosted, the Mac it
reaches is still a Mac, and the two meet over the Plow relay.

**Never run two agents on one line.** Both answer the same texts and the owner
cannot tell which replied; `plow-agents mint` refuses a line that is not free
for exactly that reason.

## License

MIT — see [`LICENSE`](LICENSE). [`NOTICE`](NOTICE) records what is vendored
from Plow PBC under Apache-2.0, and what is fetched at build time rather than
copied in.

## Not medical advice

Luna is a logistics and memory tool. It is not a medical device, it does not
practise medicine, and it does not replace prenatal care. The milestone weeks it
uses are routine low-risk guidance; the obstetrician's own plan always wins.
