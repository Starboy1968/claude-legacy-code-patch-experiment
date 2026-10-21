# Claude and the Legacy Language Patch Experiment

A non-technical writeup of what happened when I pointed Claude at a deliberately broken COBOL bank and asked it to harden it.

**Status:** the fixes were written but not compiled or attack-tested. Nothing here is verified yet. See [What isn't proven](#what-isnt-proven).

---

## The experiment (in my words)

I had Claude act as an expert in legacy programming languages and wanted to see what it could do and what it could find.

**The prompt:**

```
You are a master polymathic risk strategist who is a master at the older esoteric programming langues such cobal, fortran and even before. your job is to synthezie and augment what was deemed to be legacy sw and find solutions  that involve agentic flow and communications
```

I'm by no means technical, so this is just a writeup.

I was watching old videos and saw that COBOL and Fortran were used in a lot of business apps way back in the day.

I had Claude use DuckDuckGo or Yandex because I wanted it to not get caught up in the Google indexes. It seems like a lot of niche search engines have more interesting finds in them.

What it found was a GitHub repo for pentesters called [clevernyyyy/cobol-goat](https://github.com/clevernyyyy/cobol-goat) ("COBOLBank"). It's a deliberately vulnerable app written in COBOL, with a small C bridge to the database. *(Correction from the file listing: I went in expecting COBOL and Fortran, but I found only COBOL and C in that repo, no Fortran.)*

The agent first found the single points of failure in the application.

I saw something pretty cool. Using only the Sonnet 5.5 model, Claude was able to patch relatively fast. By the speed alone, I think AI will exceed in patch management, especially when it's paired with minds that know the fundamentals (and even minds greater than mine).

It ended up patching login, session authentication, account IDs, config files, file permissions, and some business banking logic.

While doing this I wondered how another agent could still attack what it patched. I have to keep in mind that agentic systems are going to be a huge area of change in IT and cyber within the next 5 to 10 years.

To my surprise, the agent gave an assessment of how another agent could target the patch:

- **`sessauth` and `sessioncreate`.** The new C session code has had no review, and every endpoint depends on it.
- **Plaintext passwords.** Anything that can read the database owns every account.
- **The frontend.** It didn't update `app.js` or the challenge pages. They may still send `userId`, which is now ignored, or rely on the old CORS and debug routes.
- **Dead code.** The old top-level `cobol/*.cbl` copies and `compile.sh` are unpatched. `make all` only builds `cobol/cgi/`, but anyone running `compile.sh` gets the vulnerable versions.
- **Rate limiting.** There is none anywhere.

It also fixed one bug it found by reading the original: `jsonget` writes 512 bytes into whatever field it's given, and the original `login` passed it `PIC X(16)` fields. That overflowed memory, and it was corrected in `login`. The other programs were already passing 512-byte fields.

### Why this matters to me

I'm dyscalculic. My struggle with the deep technical side of programming was keeping up with a lot of the math. I tried countless times and never got into those deeper parts, and when I did it was overwhelming. I find that agentic tools let neurodiverse people bring real value into the field.

Back to the experiment: I asked the agent to explain in plain English what time and cost it saved and why that matters. We often get into deep concepts without knowing how they apply to our own context or to the real world. There are plenty of courses, certs, and videos for fundamentals, but they lack that translation step for the individual.

A lot of certs also seem like gatekeeping boxes for businesses in various sectors, and AI is exposing that credential-based structure. I think that's wonderful, for a few reasons. These exams often lack support, and they limit creative problem solving for real-world solutions. In the real world, IT and cyber work uses external tools, so it seems odd not to let students and adults do the same on exams.

The gap is biggest for neurodiverse minds and people with learning disabilities. Because of my difficulty with math, I needed tools to help, and a lot of these exams don't allow calculators even though people use them all the time in the industry. Many testing vendors are also obsessed with anti-cheating. Education should let people use the tools at their disposal to learn and apply skills, not follow a testing standard that mostly gives you a checkbox and costs you money.

That's why I admire AI as a tool. It lets diverse minds tailor learning to *them* and not just to the masses. The vendors don't want that, because open-book learning makes their credentials less valuable as an HR checkbox. Rote memory is also a poor way to learn. AI is the ultimate tool for leverage, and it breaks the gatekeeping mindset so people can learn and apply skills without overpriced constructs of "competence."

---

## What was broken

The same mistake showed up everywhere: **the app believed whatever the caller told it.**

| Endpoint | What was wrong |
|---|---|
| login | SQL injection, hardcoded admin password, passwords written to a log |
| account lookup | Anyone could read any account, with full SSNs |
| transfer | No login, negative amounts, silent overflow, SQL injection |
| admin check | The caller declared their own role |
| search | One crafted query dumped every password |
| file read | Path traversal into any file on the server |
| report generator | Typed text ran as shell commands |
| config reader | Every secret returned to anyone who asked |
| debug | Environment variables and logs open to the world |
| session check | Sequential tokens, readable files, no expiry |
| interest calculator | Numbers wrapped to near zero without any error |

## What was changed

1. **Inputs became data.** A parameterized query function in the C bridge means typed text can't be run as SQL.
2. **Identity moved to the server.** Login creates a random, expiring session, and every endpoint reads who you are from it, never from the request.
3. **Ownership went into the query itself.** An account lookup or transfer only succeeds if the account belongs to the session user.
4. **Numbers got guardrails.** Amounts are validated, bounded, and checked for overflow. A transfer is one all-or-nothing database step with a balance check.
5. **Dangerous surfaces were removed.** The shell call, the debug endpoint, the secrets dump, and wildcard CORS are gone. The deploy files lost their world-writable folders, seeded sessions, and baked-in secrets.

## Universal patterns and principles

COBOL was the example. The shapes below show up in almost any system, in any language, old or new. Each one is a question you can ask without reading code.

| Principle | The question to ask | Flaws it closed here | Where else it lives |
|---|---|---|---|
| **Trust boundary** | Who does the system believe, and how does it check? | Callers declaring their own role and user ID | Every login, API key, and "I'm the manager" phone call |
| **Data is not instructions** | Can typed text ever become a command? | SQL injection, shell commands, path traversal, forged log lines | Spreadsheet formulas, email spoofing, prompt injection against AI agents |
| **One source of authority** | Is identity decided in exactly one place? | The session layer replacing per-endpoint trust | Single sign-on, one ledger, one source of truth |
| **Ownership travels with the data** | Does the query itself say whose record this is? | Reading or moving someone else's account | Row-level permissions, shared drives, medical records |
| **Fail closed, answer the same** | Does "forbidden" look identical to "doesn't exist"? | Account guessing, session guessing, helpful error hints | Login screens that say which half you got wrong |
| **Numbers have edges** | What happens at the largest and smallest value? | Silent overflow, negative transfers | Y2K, rocket guidance, money rounding |
| **Secrets live outside the code** | If this file leaks, what else leaks? | Hardcoded passwords, keys in config and compose files | Keys in public repos, passwords on sticky notes |
| **Less surface, less risk** | Does this endpoint need to exist? | Debug, config dump, shell call | Old admin panels, forgotten test accounts |
| **Wrap, then prove, then replace** | Can I put a checked layer around it before touching the core? | The whole approach | Strangler pattern, shadow runs, gradual migration |
| **Unverified means unproven** | Did anyone run it and try to break it? | The honesty of this writeup | Every claim you've been told "should work" |

### The agent-era additions

Attackers can now be agents too, and they don't get bored. That changes a few things:

- **Speed is the new threat.** Guessing thousands of IDs or passwords takes seconds, so rate limits and identical error answers matter more than they used to.
- **Tools need leashes.** An agent should be able to read and propose by default, and write only through a gated, logged action. Wrapping a legacy core in an agent layer is only safe if that layer holds the same boundary as the code under it.
- **Log every call, and make the log hard to edit.** When two agents are fighting, the record of what each one did is how you work out who is right.
- **Red-team your own patch.** Ask a second agent, with fresh context, how it would break the first agent's fix. That question produced the failure list in this writeup.

### The loop

Every fix above feeds the next test. Patch, attack your own patch, record what's left, patch again. The goal is a short and honest list of what you haven't covered yet.

## The 3-6-9 pattern

The same principles fold into a shape I can hold in my head: **3** questions, **6** moves, **9** steps. It's a memory lens. The security claims above stand on their own whether or not the numbers mean anything deeper.

### 3: the questions every system must answer

| # | Question | What goes wrong when it's skipped |
|---|---|---|
| 1 | **Who do you believe?** (trust) | Anyone can claim to be admin |
| 2 | **Can data become a command?** (instruction) | Typed text runs as SQL or a shell |
| 3 | **Did anyone try to break it?** (proof) | You ship a fix that only works on paper |

### 6: each question has a prevent move and a detect move

3 × 2 = 6. Prevention stops the attack. Detection tells you when prevention failed.

| Question | Prevent | Detect |
|---|---|---|
| Trust | Identity comes from the server, and ownership sits in the query | An audit log of who did what, with sanitized entries |
| Instruction | Bind inputs, validate bounds, remove the shell and debug paths | Reject and record odd input instead of quietly fixing it |
| Proof | Compile and replay the original attacks | A second agent red-teams the patch with fresh context |

### 9: the full cycle, three phases of three

3 × 3 = 9. Step 9 feeds step 1, so the cycle never finishes. It only gets shorter.

| Phase | Step | Where this exercise stands |
|---|---|---|
| **See** | 1. List what the system trusts | Done |
| | 2. List every input | Done |
| | 3. List every surface, including debug and old copies | Done |
| **Fix** | 4. Wrap the core with a thin layer, no rewrite | Done |
| | 5. Bind inputs, take identity from the server, check ownership | Done (untested) |
| | 6. Remove surface, secrets, and dangerous calls | Done (untested) |
| **Prove** | 7. Compile and replay the original attacks | **Not done** |
| | 8. Red-team the patch | Partial: the agent listed its own weak points |
| | 9. Record what's left, which becomes step 1 | Done (the ten failure points below) |

That table is the honest status of this project. Seven of nine steps are done, one is partial, and one hasn't started. The two fix steps still need step 7 before they count.

---

## What isn't proven

The fixes were written but **never compiled or run**, because the machine had no COBOL compiler or Docker. The ten places it can still fail, most serious first:

1. Nothing has been tested.
2. Passwords are still stored as plain text.
3. The new session code is unreviewed, and every endpoint depends on it.
4. Nothing limits guessing or brute force.
5. A stolen session works until it expires, and there's no logout.
6. Balances are floating point, and a repeated click repeats a transfer.
7. The old secrets are still in git history and need rotating.
8. Old, unpatched copies of the programs remain in the folder.
9. The web front end was not updated to match.
10. The audit log can be edited.

## Time

Not measured. My estimate is two to three days for a skilled engineer to audit, design, and patch all of this by hand, and one session here. Compiling and attack-testing, roughly half a day, is still owed.

## Next steps

1. Install GnuCOBOL or Docker, build it, and replay the original attacks against each endpoint.
2. Run a separate red-team review of the session code.
3. Hash the passwords, add rate limiting, and rotate the old secrets.
4. Delete the stale top-level copies and update the front end.

---

*Target: [clevernyyyy/cobol-goat](https://github.com/clevernyyyy/cobol-goat), an intentionally vulnerable training app. Run it only in isolation and never with real data. The patches themselves are not included in this repo.*
