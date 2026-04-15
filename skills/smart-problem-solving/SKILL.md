---
name: smart-problem-solving
title: Smart Problem Solving
description: How to diagnose and solve technical problems efficiently, distilled from ESR's "How To Ask Questions The Smart Way". Use before asking anyone for help.
---

# Smart Problem Solving

## Before You Ask Anyone

Do your homework. Go through this checklist in order:

1. **Search** — Google the exact error message; search the project's issues/forum archives.
2. **Read** — Skim the manual, README, and FAQ. Most problems are documented.
3. **Experiment** — Change one variable at a time and observe. Isolate the trigger.
4. **Read the source** — If it's open source, grep the codebase for the error string.
5. **Ask a skilled friend** — Only after the steps above.

> Hasty questions get hasty answers or silence. The more effort you show, the more likely experts will help.

---

## How to Frame the Problem

### Describe symptoms, not your guesses
- **Bad:** "I think the database connection pool is leaking."
- **Good:** "After 20 minutes of uptime, memory usage climbs from 200 MB to 2 GB and requests start timing out. Restarting the process resets it."

### Describe the goal, not the step you're stuck on
- **Bad:** "How do I set `foo.bar = 3`?"
- **Good:** "I need to disable retries for this specific endpoint. I tried setting `foo.bar = 3` but it didn't work. Is there a better way?"

### Be precise and informative
Include these facts up front:
- Environment (OS, version, language/runtime version)
- Exact error messages / logs
- What you already tried and the results
- Minimal reproducible steps or code snippet
- Any recent config/code changes

### Volume is not precision
Trim huge code dumps. Create a **minimal test case** that exhibits the bug with the least code possible. This often leads you to the fix yourself.

### Don't rush to claim it's a bug
Assume you are doing something wrong until proven otherwise. Extraordinary claims require extraordinary evidence.

---

## When You Need External Help

### Choose the right channel
- Search Stack Overflow / project issues first.
- Post to the correct forum/list; don't cross-post.
- Use a meaningful subject: `object - deviation` (e.g. "Vite 5 build fails with `EMFILE` on macOS 14").

### Be explicit about what you want
- **Weak:** "Can someone explain X?"
- **Strong:** "I read the docs on X but don't understand how it handles Y. Can you point me to an example or section I missed?"

### Courtesy & follow-up
- Say thanks.
- After it's solved, post a brief follow-up with the fix. This helps the next person searching.
- If the docs were unclear, send a documentation patch.

---

## How to Interpret Answers

- **RTFM / STFW / "Google is your friend"** → The answer is easy to find. Go look it up yourself; you'll learn more.
- **If you don't understand the answer** → Research the terms they used before demanding clarification. Then ask a follow-up showing what you learned.
- **Rudeness** → Much of it is just direct, low-fluff communication. Don't take it personally. Stay focused on the problem.

---

## Classic Pitfalls to Avoid

| Pitfall | Why it hurts |
|---------|--------------|
| "Urgent!" flags | Experts delete these as rude noise. |
| Yes/no appendages ("Can anyone help?") | Superfluous and annoying. |
| Dumping homework | You won't learn; hackers spot it instantly. |
| Private replies | Breaks the public knowledge record. |
| Grovelling | "I'm a pathetic newbie..." is distracting. Just state facts. |
| All-caps or l33t speak | Signals sloppy thinking; you'll be ignored. |

---

## The Core Mindset

> Hackers are volunteers. Time is scarce; expertise is abundant. The less time you ask someone to spend, the more likely a busy expert will answer. Demonstrate alertness, observation, and a willingness to be an active partner in solving the problem.
