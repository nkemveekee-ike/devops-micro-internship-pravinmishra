# Assignment 6 — Building an AI-Assisted Git Safety Net (PR Ready Check)

Part of the DevOps Micro Internship (DMI) Cohort 3 with Agentic AI

---

## Purpose

In Week 2 you built Claude Code hooks that block a dangerous action *before* it happens (`PreToolUse`), and a restricted skill that could look but not touch (`allowed-tools` without `Write`). In this assignment you will discover that Git has the exact same idea, decades older: a **pre-commit hook** that blocks a commit before it's created.

You will build both halves of a real "PR Ready" workflow:

1. A **Git hook that follows fixed rules** — scans staged changes for hardcoded secrets and oversized files and refuses the commit. No AI involved, no guessing, just a rule that gives the same answer every time.
2. A **restricted Claude Code skill** (`/pr-ready`) that reads your staged diff and drafts a Pull Request title, description, and a short list of things worth a second look — the kind of judgment a fixed rule can't make (mixed changes, missing context, unclear intent). The skill never commits, pushes, or opens the PR. You do that yourself, using its draft as a starting point.

This mirrors the Agentic Loop from Week 3's Linux triage assignment: **Gather → Analyze → Human Act → Verify**. The hook and the skill both gather and analyze; only you act.

---

# Task 0 — Confirm Your Fork and Create a Feature Branch

## Goal

Confirm you are working in your own fork, then create a dedicated branch for this assignment.

### Evidence

#### Screenshot 1 — Output of git remote -v and git branch showing the new branch

![screenshots](screenshots/A6-T0-S0.png)

---

### Notes

**1. Why create a dedicated branch instead of doing this work on main?**

I chose to work on a separate branch because this task involved intentionally adding a fake secret and a debug line. Even though they weren't real, I didn't want those changes anywhere near main. That way, if I made a mistake or forgot a step, my main branch would stay clean. It also reinforced what I learned earlier about using feature branches for changes like this and only merging them into main through a Pull Request after everything has been checked.

---

# Task 1 — Stage a Change With Realistic Risk

## Goal

On your own fork of this repository (the one you've been submitting your DMI work in since onboarding), create a new branch and stage a change that a real reviewer should catch: a hardcoded-looking secret and a leftover debug statement.

### Evidence

#### Screenshot 1 — Output of  `git status` showing the staged file on feature/ai-pr-ready

![screenshots](screenshots/A6-T1-S1.png)

---

### Notes

**1. Why does this assignment use an obviously fake key instead of a real one?**

I used a fake AWS key because the goal of this task was to test whether the pre-commit hook and AI skill could detect a secret—not to create a real security risk. If I had used an actual AWS key and accidentally pushed it to a public repository or included it in a Pull Request, I'd be doing the exact thing this assignment is teaching me to avoid. Using a clearly labeled fake key let me test the workflow with confidence, knowing that even if I made a mistake, I wasn't exposing any real credentials.

---

# Task 2 — Write a Real Git Pre-Commit Hook

## Goal

Create a tracked, shareable pre-commit hook that blocks a commit containing secret-like patterns or files over 1MB.

### Evidence

#### Screenshot 2 — `hooks/pre-commit` open in VS Code showing the full script

![screenshots](screenshots/A6-T2-S1.png)
---

#### Screenshot 3 — Output of `git config core.hooksPath` confirming it points to `hooks`

![screenshots](screenshots/A6-T2-S2.png)

---

### Notes

**1. Why is `hooks/pre-commit` tracked in the repo instead of living only in `.git/hooks/`?**

Because .git/hooks/ never actually gets pushed or pulled  it's local to whoever's machine it's sitting on. If I'd just dropped the hook straight into .git/hooks/ without tracking it, it would only ever protect me, on my computer. Anyone else who clones the repo wouldn't have it at all, and they'd have no idea it was even supposed to exist unless someone told them. By putting it in hooks/ and running git config core.hooksPath hooks, the script itself becomes part of the repo's history  so when a teammate clones it, the hook comes with the code, and one config command turns it on for them too. It's the difference between a rule that only I follow versus a rule the whole team gets automatically.

---

**2. Compare this to `PreToolUse` from Week 2 Assignment 6. What does each one intercept, and what do they have in common?**

One thing I noticed is that both PreToolUse and the Git pre-commit hook act as safety checkpoints, but they protect different parts of the workflow.

PreToolUse runs before Claude Code executes a tool, such as a Bash command or editing a file. It checks the requested action first and can allow it, block it, or ask for approval before anything actually happens. In other words, it helps control what the AI is allowed to do.

The Git pre-commit hook, on the other hand, runs automatically when I execute git commit. It doesn't matter whether the changes were made by me, a script, or another tool—it simply checks everything I've staged before allowing the commit to go through. If it detects something that breaks the defined rules, like a secret or another restricted pattern, it blocks the commit.

Working with both helped me understand that they serve the same purpose in different places. PreToolUse protects the AI's actions before they happen, while the pre-commit hook protects the repository before changes become part of its history. Having both gives me an extra layer of confidence because one helps prevent risky actions during development, and the other acts as a final safeguard before the changes are permanently committed.

---

# Task 3 — Prove the Hook Blocks the Risky Commit

## Goal

Attempt to commit the staged file from Task 1 and show the hook rejecting it.

### Evidence

#### Screenshot 4 — Terminal showing `git commit` rejected with the hook's "BLOCKED" message naming the exact file

![screenshots](screenshots/A6-T3-S1.png)

---

### Notes

**1. Which line in `hooks/pre-commit` matched your fake key, and why did it match?**

if git diff --cached -- "$file" | grep -qE 'AKIA[0-9A-Z]{16}|-----BEGIN (RSA|OPENSSH|PRIVATE) KEY-----'; then


When I looked at the pattern, I understood why my fake key was detected. I had intentionally used AKIAABCDEFGHIJKLMNOP, which starts with AKIA and is followed by 16 uppercase letters, just like the format the hook was checking for. Even though the key wasn't real, it still matched the expected pattern, so the hook treated it as a potential secret and blocked the commit. That helped me see that the hook isn't trying to determine whether a key is valid it simply looks for patterns that resemble real secrets to prevent them from being committed by mistake.

---

**2. Could this hook have caught a poorly-named variable that stores a secret without the `AKIA` prefix? What does that tell you about the limits of a fixed rule like this?**

No, it couldn't have. I realized the hook only catches patterns it has been explicitly written to detect, such as the AKIA prefix. If a secret didn't match one of those patterns, the hook would likely let it pass. That showed me the limitation of fixed-rule checks they're fast and consistent, but they can't identify anything outside their predefined rules, which is why combining them with the AI skill provides better protection.

---

# Task 4 — Build the `/pr-ready` Skill

## Goal

Create a manually invoked Claude Code skill that reads your staged changes and produces a PR-readiness report and a draft PR description — without writing, committing, or pushing anything itself.

### Evidence

#### Screenshot 5 — `SKILL.md` frontmatter showing `allowed-tools: Bash, Read, Grep` (no `Write`) and `disable-model-invocation: true`

![screenshots](screenshots/A6-T4-S2.png)

---

#### Screenshot 6 — `/pr-ready` output while the risky file is still staged, showing it flagged the secret and/or debug statement

![screenshots](screenshots/A6-T4-S3.png)

---

### Notes

**1. Why does `/pr-ready` have `Bash` and `Read` but not `Write`?**

 /pr-ready has Bash and Read because it needs them to inspect my staged changes by running commands like git diff -- cached and git status, and to read files when necessary. It doesn't have Write because its role is only to analyze my changes and suggest a PR title and description it isn't supposed to modify my files. Keeping Write out also acts as a safety measure, ensuring that only I can make changes, commit them, and submit the Pull Request.

---

**2. The pre-commit hook and `/pr-ready` both looked at the same staged diff. Did they flag the same things? What did one catch that the other didn't?**

Both the pre-commit hook and /pr-ready flagged the hardcoded AWS key because it matched the expected pattern. However, only /pr-ready pointed out the debug echo statement since the hook wasn't designed to detect that kind of issue. This showed me that the hook enforces specific rules, while /pr-ready can also recognize risky changes based on the overall context, making the two tools work well together.

---

# Task 5 — Fix the Issues and Re-Verify

## Goal

Remove the secret and debug statement, then prove both gates now pass clean.

### Evidence

#### Screenshot 7 — `git commit` succeeding after the fix (no BLOCKED message)

![screenshots](screenshots/A6-T5-S1.png)

---

#### Screenshot 8 — Second `/pr-ready` run showing a clean risk report and a drafted PR title + description

![screenshots](screenshots/A6-S8.png)

---

### Notes

**1. What exactly did you change to satisfy the pre-commit hook?**


I edited scripts/notify.sh to remove the two lines the hook and /pr-ready both flagged:

- I deleted the hardcoded credential line: `AWS_ACCESS_KEY_ID=AKIAABCDEFGHIJKLMNOP`
- I deleted the debug echo that printed it: `echo "DEBUG: token is $AWS_ACCESS_KEY_ID"`

What was left was just the shebang and a harmless status message:

#!/bin/bash
# demo only — fake credential for this assignment, never a real key
echo "Notification script ready."

With the AKIA-shaped string gone, the hook's grep -qE pattern had nothing left to match, so re-staging and committing went through cleanly with no BLOCKED message.

---

# Task 6 — Push and Open a Pull Request Using the AI Draft

## Goal

Push your branch and open a real Pull Request, using `/pr-ready`'s drafted title and description as your starting point — read it critically and edit before you use it.

**Important:** Open this Pull Request with base repository set to **your own fork** — not the shared upstream `pravinmishraaws/devops-micro-internship-pravinmishra` repository. This assignment's hook and skill files are your own practice work, not a change meant for the shared class repo.

### Evidence

#### Screenshot 9 — Your Pull Request showing the base repository is your own fork, plus the title and description, with the `/pr-ready` draft visible for comparison (paste it in the PR conversation or your notes below)

![screenshots](screenshots/A6-PULL.png)

---

#### PR Link

https://github.com/nkemveekee-ike/devops-micro-internship-pravinmishra/pull/1

---

### Notes

**1. What, if anything, did you edit in the AI's drafted PR description before using it? Why?**

I reviewed the AI's draft before using it and made a few edits to improve the wording and make the description more complete. I wanted the Pull Request to clearly explain what I changed and why, rather than relying on the AI's first draft. Reviewing and editing it also helped ensure the description accurately matched the changes in my PR.

---

**2. If you had blindly copy-pasted the AI's draft without reading it, what could go wrong?**

If I had copy pasted the AI's draft without reading it, the Pull Request description might not have accurately reflected my changes or provided enough context for reviewers. That could lead to confusion, misunderstandings, or an incomplete review. This assignment taught me that AI should provide a helpful starting point, but it's still my responsibility to review, edit, and make sure the final PR description accurately matches my work.

---

**3. Why does this PR need to target your own fork instead of the shared upstream repository?**

This PR needed to target my own fork because these files were created for my internship assignment, not as a contribution to the shared upstream repository. Opening the PR against my fork kept my work in my own repository, where I have permission to manage and merge my changes. If I had targeted the upstream repository instead, I would have been submitting personal assignment files to a shared project, which isn't what this assignment was intended for.

---

# Task 7 — Map the Workflow to the Agentic Loop

## Goal

Explain this assignment's workflow using the same Gather → Analyze → Human Act → Verify structure from Week 3.

### Notes

**1. Which step(s) represent Gather?**

I would put Gather at the very start of both checks, before either one makes a judgment call. The pre-commit hook opens with `git diff --cached` and `git status` to see exactly what's staged, and `/pr-ready` does the same two commands as its first move. Neither one is deciding anything yet they're just collecting the raw facts about my staged change.

---

**2. Which step(s) represent Analyze?**

The hook's `grep -qE` check against the AKIA/PEM-key pattern is technically analysis, but it's about as narrow as analysis gets it's just asking "does this string match this exact shape," nothing more. The real Analyze step, to me, is `/pr-ready`'s report. That's where actual judgment happened: it looked at my diff and decided whether something was a secret, a leftover debug line, or an unrelated change I'd mixed in.

---

**3. Which step is Human Act, and why must a human — not Claude — run `git commit`, `git push`, and open the PR?**

Human Act was everything I did with my own hands: editing `scripts/notify.sh` to strip out the key and the debug echo, running `git add`, `git commit`, and `git push` myself, and reading through the AI's draft PR description critically before rewriting it and submitting it. This has to stay human because the skill's `allowed-tools` deliberately leaves out `Write`, and its own instructions say it can't run `git commit`, `git push`, or `gh pr create`. Once a change actually touches the shared repo, it's irreversible in a way that matters — I wanted that decision, and the accountability for it, to be mine.

---

**4. Which step is Verify?**

Verify shows up twice for me. In Task 3, I confirmed the hook actually blocked my risky commit git log showed nothing new got created. In Task 5, I confirmed the opposite: the commit succeeded cleanly after my fix, and a second `/pr-ready` run came back clean too

---

**5. In one or two sentences: why do you need *both* the fixed-rule pre-commit hook and the AI skill? Isn't one enough?**

Honestly, neither one alone would've cut it the hook is fast and consistent but only catches exactly what its regex is written for, so a secret without the AKIA prefix would slip right past it, while /pr-ready can reason about context but isn't reliable enough to trust as a hard gate on its own. Using both means the certain stuff gets caught automatically every time, and the fuzzier judgment calls still get a second set of eyes before I make the final call.

---

# Task 8 — LinkedIn Post

## Goal

Publish a LinkedIn post summarizing what you built and what you learned about combining fixed-rule safety checks with AI-assisted review.

### Evidence

#### LinkedIn Post URL

https://www.linkedin.com/posts/ekweozor_dmibypravinmishra-git-github-share-7485979206628179969-0bC6/?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEFzwtYB-RXnYG13TMOIwtIDL3APbwSz4XI

---

## Key Learnings

Add 3-5 bullet points on what you learned this week.

1.  I got a lot more comfortable using Git and GitHub for real collaboration this week instead of just working locally like I had before.
2. Branching finally made sense in a hands-on way. Creating a feature branch, keeping the default branch clean, and only merging after everything checked out helped me understand why teams don't work directly on main.
3. I learned how a pre-commit hook can catch mistakes like hardcoded secrets before they ever reach the repository, no matter who is committing the code.
4. I built my first Claude Code skill that reviews staged changes and drafts a Pull Request without editing, committing, or pushing any code itself.
5. I also reinforced the Agentic Loop by applying Gather → Analyze → Human Act → Verify to a real Git workflow, from creating a feature branch and opening a Pull Request to using a fixed-rule safety check backed up by AI review.

---

# Submission Instructions

- Ensure `hooks/pre-commit` and `.claude/skills/pr-ready/SKILL.md` are committed to your GitHub repository
- Add all required screenshots to your submission
- All written answers must be in your own words
- Do not use a real secret or credential anywhere in your submission — the fake key in Task 1 is intentional and must stay clearly fake
- Open your Pull Request against your own fork, not the shared upstream repository
- Push your final changes to your forked repository
- Include your PR link and LinkedIn post URL

---

## GitHub Repository URL

Paste your forked repository URL here:

https://github.com/nkemveekee-ike/devops-micro-internship-pravinmishra

---

# Completion Checklist

- [X] Branch `feature/ai-pr-ready` created with a staged file containing a fake secret and a debug statement
- [X] `hooks/pre-commit` created and tracked in the repo (not only in `.git/hooks/`)
- [X] `core.hooksPath` configured to point at `hooks/`
- [X] Pre-commit hook shown blocking the risky commit
- [X] `.claude/skills/pr-ready/SKILL.md` created with correct `allowed-tools` (no `Write`) and `disable-model-invocation: true`
- [X] `/pr-ready` run against the risky diff and shown flagging issues
- [X] Risky file fixed; `git commit` succeeds cleanly
- [X] `/pr-ready` re-run showing a clean report and drafted PR title/description
- [X] Pull Request opened using the AI draft as a starting point, with your own fork as the base repository (not upstream), PR link included
- [X] Agentic Loop mapping (Task 7) completed in your own words
- [X] LinkedIn post published and URL submitted
- [X] All required screenshots added
- [X] GitHub repository URL provided

---

## 📌 About DMI & CloudAdvisory

DevOps Micro Internship (DMI) is a project-based DevOps program run by Pravin Mishra (The CloudAdvisory) focused on real-world execution, systems thinking, and career readiness.

It helps learners build strong DevOps foundations with hands-on experience.

---

## 📌 Resources

- 🌐 DMI Official Website: https://pravinmishra.com/dmi  
- 🎓 DevOps for Beginners (Udemy): https://www.udemy.com/course/devops-for-beginners-docker-k8s-cloud-cicd-4-projects/  
- 🎓 Agentic AI DevOps with Claude Code: https://www.udemy.com/course/ultimate-agentic-ai-devops-with-claude-code/  
- 🎓 DevOps with Claude Code: Terraform, EKS, ArgoCD & Helm: https://www.udemy.com/course/devops-with-claude-code-terraform-eks-argocd-helm/  
- ▶️ YouTube Playlist: https://www.youtube.com/playlist?list=PLFeSNDtI4Cho  
- 🔗 Pravin Mishra (LinkedIn): https://www.linkedin.com/in/pravin-mishra-aws-trainer/  
- 🏢 CloudAdvisory (LinkedIn): https://www.linkedin.com/company/thecloudadvisory/

---

*This submission is part of DevOps Micro Internship (DMI) Cohort 3 — Agentic AI Track.*
