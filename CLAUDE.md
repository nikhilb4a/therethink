# Working on the Re:think website (nikhilb4a/therethink)

You're helping maintain the Re:think website. The person asking you for changes may not be technical. Your job is to make **small, targeted, safe changes**, show them what the change looks like before it goes live, and only merge once they've approved it. When in doubt, do less and ask.

---

## 1. How the site works

- **Plain static HTML.** No framework, no build step, no package.json. Every page is one self-contained `.html` file with its CSS in a `<style>` block and its JavaScript in a `<script>` block at the bottom.
- **Deploys automatically.** Netlify watches the `main` branch. Anything merged into `main` is live on the public site about 1–2 minutes later. **Merging = publishing.**
- **Live addresses:**
  - `www.therethink.com` (and `therethink.com`): this repo
  - `lab.therethink.com`: old address; permanently redirects to the same page on www (handled in `_redirects`)
  - `me.therethink.com`: the **personal coaching site on Squarespace**. It is NOT in this repo and can't be edited from here. Changes to it are made in Squarespace's own editor.

### Files

| File | What it is |
|---|---|
| `index.html` | Landing page with two panels: "individual" → me.therethink.com, "business" → the Lab |
| `rethink-lab.html` | Re:think Lab home page |
| `lab-who-we-serve.html` | Who We Work With |
| `lab-approach.html` | Our Philosophy |
| `lab-how-we-work.html` | How We Work |
| `lab-team.html` | Team (bios and photos) |
| `lab-contact.html` | Contact |
| `_redirects` | Netlify redirect rules (old lab links, old Squarespace links). **Don't edit unless explicitly asked.** |
| `CLAUDE.md` | These instructions. Hidden from the public site by a rule at the top of `_redirects`. |
| `*.png`, `*.jpeg` | Logos, icons, team photos, banner |

### Things that are repeated across pages (change them everywhere)

- The **top nav**, **mobile hamburger menu**, and **footer** are copied into each of the 6 lab pages (`rethink-lab.html` + the five `lab-*.html`). If you change a nav link, menu item, or footer, change it in **all six**, keeping the current-page highlight (`class="active"`) on the right item in each.
- Each page has its own copy of shared styles (colors in `:root`, fonts, nav styles). A "site-wide" style change means editing every page.

### Mobile layout

- Breakpoints: `1100px`, `900px`, `768px`, `480px`.
- At **≤768px** the desktop nav links hide and the **hamburger button** (`#nav-hamburger`) opens a full-screen menu (`#mobile-menu`).
- Known gotcha: the `nav` must have a higher `z-index` (300) than `.mobile-menu` (250), or the close "X" gets covered by the menu.

---

## 2. Ground rules

1. **Only change what was asked.** No drive-by refactors, reformatting, renaming, or "improvements". If you notice another problem, mention it and ask; don't fix it in the same change.
2. **One request = one branch = one PR.** Keep the diff small enough to read in a minute.
3. **Never push to `main` directly. Never force-push. Never rewrite history.**
4. **Never delete files, images or pages** unless explicitly asked, and confirm first.
5. **Text content:** use the exact wording you were given. Don't invent claims, credentials, statistics, client names, testimonials or quotes. If wording is missing or unclear, ask.
6. **Don't touch without an explicit request:** `_redirects`, `<link rel="canonical">`, the `application/ld+json` block, `<meta>` tags, the domain links (`me.therethink.com`, `www.therethink.com`), the email address, or the Kit sign-up form script on `rethink-lab.html`.
7. **Images:** keep file names stable if the page already references them (or update every reference). Keep new images reasonably sized for the web (aim for under ~500 KB; resize big photos).
8. **Out of scope here; explain, don't guess:** DNS/domain settings (managed at NameBright), Netlify settings, Squarespace (me.therethink.com), and email (Microsoft 365). Never suggest changing DNS MX, `autodiscover` or TXT records; they run the email.
9. **Ask before anything you can't easily undo**, and always before merging.

---

## 3. Workflow for every change

### Step 1: Understand
- Restate the request in one or two plain sentences ("I'll change the heading on the Team page from X to Y on desktop and mobile").
- If it's ambiguous (which page? what exact text? which photo?), ask **before** editing. Find the exact spot in the code first so your question is specific.

### Step 2: Branch and edit
```bash
cd <repo>
git fetch origin main && git checkout -b <short-descriptive-branch> origin/main
# make the edit(s)
git diff --stat   # confirm ONLY the files you meant to touch changed
git diff          # read every changed line
```

### Step 3: Render and look at it (required, never skip)
Render the changed page(s) **before and after**, at **desktop (1280px)** and **phone (390px)** widths, and actually look at the screenshots. Check:
- the change appears exactly as intended, and nothing else moved, overlapped or broke
- text isn't cut off or overflowing on phone width
- if nav/menu was touched: the hamburger opens and closes and the X is visible
- **no JavaScript errors** in the console

Chromium + Playwright are usually preinstalled (don't run `playwright install`). Script to adapt:

```python
# render.py: usage: python3 render.py <repo_dir> <out_dir> page1.html [page2.html ...]
import sys
from playwright.sync_api import sync_playwright
repo, out, pages = sys.argv[1], sys.argv[2], sys.argv[3:]
with sync_playwright() as p:
    b = p.chromium.launch()
    for w, h, tag in [(1280, 800, "desktop"), (390, 844, "phone")]:
        pg = b.new_page(viewport={"width": w, "height": h}, device_scale_factor=2)
        errors = []
        pg.on("pageerror", lambda e: errors.append(str(e)))
        for f in pages:
            pg.goto(f"file://{repo}/{f}"); pg.wait_for_timeout(2200)  # landing page has a ~2s intro animation
            pg.screenshot(path=f"{out}/{f}-{tag}.png", full_page=True)
            if tag == "phone" and pg.query_selector("#nav-hamburger"):
                pg.click("#nav-hamburger"); pg.wait_for_timeout(500)
                pg.screenshot(path=f"{out}/{f}-{tag}-menu.png")
                pg.click("#nav-hamburger"); pg.wait_for_timeout(400)
        print(tag, "JS errors:", errors or "none")
        pg.close()
    b.close()
```
For "before" shots, render a copy of `main` (e.g. `git worktree add /tmp/before origin/main`, or `git archive origin/main | tar -x -C /tmp/before`).

Full-page screenshots are tall. Crop to the part that changed when showing them. A side-by-side before/after image is the clearest way to show a non-technical person what will change.

### Step 4: Show and get approval
- Send the before/after screenshots to the person and describe the change in plain language.
- Ask: "Does this look right? Shall I publish it?" **Do not merge until they say yes.**

### Step 5: Commit, push, open PR
```bash
git add <specific files>   # not `git add -A` unless you've checked git status
git commit -m "<plain description of the change>"
git fetch origin main && git rebase origin/main   # stay current; resolve conflicts carefully or ask
git push -u origin <branch>
```
Open the PR with the REST API (`gh pr create` may fail because GraphQL can be blocked in this environment):
```bash
gh api repos/nikhilb4a/therethink/pulls -f title="<title>" -f head=<branch> -f base=main -F body=@pr_body.md --jq .html_url
```
PR description: what changed and why, in plain words; which pages; what you checked (desktop/phone render, no JS errors).

### Step 6: Pre-merge safety check
All of these must be true:
- [ ] The person approved the screenshots
- [ ] `git diff origin/main...HEAD --stat` shows only the intended files
- [ ] PR is mergeable with no conflicts: `gh api repos/nikhilb4a/therethink/pulls/<n> --jq '{mergeable, state}'` → `mergeable: true`
- [ ] Pages render with no JS errors at desktop and phone width
- [ ] No ground rule from section 2 was broken

Then merge:
```bash
SHA=$(gh api repos/nikhilb4a/therethink/pulls/<n> --jq .head.sha)
gh api -X PUT repos/nikhilb4a/therethink/pulls/<n>/merge -f merge_method=merge -f sha=$SHA
```

### Step 7: Verify it's live
- Wait ~2 minutes for Netlify to deploy.
- Fetch the live page (e.g. `https://www.therethink.com/lab-team.html?v=<anything>`; the query string avoids stale caches) and confirm the new content is there. The shell may not be able to reach the site; use the web-fetch tool instead.
- Tell the person it's live, with the **PR link**. Suggest they refresh (or use a private window) if they still see the old version.

### If something goes wrong after merging
Don't patch on top in a hurry. Revert the PR (open a revert PR that undoes the merge commit: `git revert -m 1 <merge_sha>` on a new branch → PR → merge), confirm the site is back to normal, then fix properly in a fresh PR.

---

## 4. Communicating with a non-technical person

- Plain language. Say "the Team page" and "the menu on phones", not file names or CSS terms, unless they ask.
- Lead with what will visibly change on the site, show screenshots, and say when it'll be live.
- Never leave them with a command to run or a setting to change without step-by-step instructions, and say what it does.
- If a request can't be done from this repo (e.g. editing me.therethink.com, which is Squarespace), say so plainly and point them to where it's done.
- If you're unsure what they want, ask one specific question rather than guessing.
