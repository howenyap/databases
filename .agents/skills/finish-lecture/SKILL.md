---
name: finish-lecture
description: Record a lecture the user just finished watching in this notes repo. Covers the quiz, notes summary, three quiz files, links to slides, notes and cited papers, guest speaker page, syllabus update, and push. Use when the user says they finished, watched, or completed a lecture, or asks to quiz them on one.
---

# Finish a lecture

Each finished lecture gets one folder, for example `cmu-15-445/14-query-execution-2/`:

```
README.md          Resources, Papers and Summary sections
quiz/
  quiz.html        interactive quiz; the only place questions are written
  quiz.md          generated: questions only
  answer.md        generated: questions with answers
speaker.md         only if there's a guest speaker: who they are, their company, links
```

Use `cmu-15-445/14-query-execution-2/` as the reference for every file. Match its format rather than inventing a new one.

**Never commit PDFs.** The repo is public, and the slides, course notes and papers are copyrighted. Link to them instead.

## Local scripts

`scripts/` is gitignored, so it exists only on the user's machine. If a script is missing, write it again in `scripts/` from this description:

- `scripts/vtt-to-text.py <file.vtt>`: turns YouTube auto-captions into ~30-second paragraphs prefixed with `[m:ss]`. Rolling captions repeat each line, so keep only lines containing `<c>` word tags, strip the tags, and drop consecutive duplicates.
- `scripts/quiz-md.mjs <lecture-dir>...`: reads the `Q` array from `<lecture-dir>/quiz/quiz.html` and writes `quiz/quiz.md` (questions with timestamps and lettered options) and `quiz/answer.md` (the same, plus `**Answer:**` with the letters and the explanation). Each file links to the other and to `quiz.html`. It throws if a question lacks `q`, `ts` or `e`, or if an answer index is out of range.

## 1. Get the transcript

The syllabus row in `cmu-15-445/README.md` has the video, slides and notes URLs.

- YouTube's subtitle endpoint returns `HTTP 429` to yt-dlp unless you're signed in. Don't retry in a loop. Ask the user to paste the caption `.vtt` file, or ask before using `--cookies-from-browser`.
- Convert the captions: `python3 -I scripts/vtt-to-text.py <file.vtt> > /tmp/<lecture>/transcript.txt`.
- The captions may be machine-translated and garbled (for example "streams" for threads, "cars" for machines). Write from what the lecture means, not from its wording. Download the slides to `/tmp` to resolve unclear terms.
- Keep the transcript and any downloads out of the repo.

## 2. Quiz the user

- If the user gives questions, use them as written. Otherwise write about 10 from the transcript, using multiple choice, one or two select-all, and one or two written answers.
- Ask with `ask_question`, **one call per question** (no batch), with `includeOther: false` on choice questions.
- After each answer, say whether it was right and explain the correct answer in 1-3 sentences. Grade written answers on the key idea, not the wording.
- Finish with the score and the topics to review.
- If `ask_question` errors, report the exact error and stop. Don't fall back to plain-text questions.

## 3. Write the notes (`README.md`)

The file has the title (`# #NN: Topic`) and three sections, in this order:

1. **`## Resources`:** a bullet list.
   - Video, using the video id only, without the playlist parameters.
   - Slides and Notes, linked to the course site. The last lecture has no notes PDF.
   - Readings.
   - Quiz: `[interactive](quiz/quiz.html) (open it in a browser) · [questions](quiz/quiz.md) · [answers](quiz/answer.md)`.
   - Guest speaker: `[speaker.md](speaker.md)`, only if there's a guest.
2. **`## Papers`:** see step 5.
3. **`## Summary`:** starts with "The timestamps link to the part of the video they cover."
   - Topic sections are `### 0.`, `### 1.` and so on, with `####` subsections. Each heading gets a timestamp link (`https://www.youtube.com/watch?v=<id>&t=<seconds>s`) taken from the transcript paragraph where the topic starts. Spot-check a few.
   - Bullets are short, nested only where the content is a list, and include the concrete examples, numbers and system names the lecture uses.
   - It ends with `### Takeaways`. If there's a guest talk, add `### Guest talk: ...` after it, opening with a link to `speaker.md`.

## 4. Save the quiz

1. Copy the newest lecture's `quiz/quiz.html` into the new lecture's `quiz/` folder. Change only the title in `<div class="meta">` and the `Q` array.
2. Put the questions from step 2 in `Q`, each with a `ts` timestamp and an `e` explanation:
   - `kind: "one"` (the default) uses `o` (options) and `a` (an index).
   - `kind: "many"` uses `o` and `a` (an array of indexes).
   - `kind: "open"` uses `e` as the expected answer.
3. Run `node scripts/quiz-md.mjs cmu-15-445/<lecture-dir>`. Never edit `quiz.md` or `answer.md` by hand.

The page defaults to light mode, with a dark toggle saved in `localStorage`. Keep that when copying.

## 5. Find the papers

The slides cite papers as images, so their text has no citations. Find papers in the transcript instead:
- papers named aloud, including in guest talks
- the original papers behind the ideas the lecture teaches (for example the Volcano paper behind the iterator model)

For each paper:

1. **Find the DOI** with the Crossref API: `https://api.crossref.org/works?query.bibliographic=<title author year>&rows=2`. dblp returns `429` after a few requests, so don't use it for lookups.
2. **Find a free PDF from a legitimate host**: the author's homepage, then the conference site (vldb.org, cidrdb.org), then the publisher's open-access page. `https://api.semanticscholar.org/graph/v1/paper/DOI:<doi>?fields=openAccessPdf` tells you whether one exists. Don't link third-party copies such as other universities' reading lists, and never link Sci-Hub or other piracy mirrors.
3. **Check the link:** download it to `/tmp` and read the first page with pypdf (run Python with `-I`) to confirm the title. ACM (`dl.acm.org`) returns `403` and IEEE Xplore returns `418` to curl, so link those without checking.
4. **Write one bullet:** `Author, *Title*, Venue Year. Why it's here. [PDF](url) · [doi](url)`. Say whether it was mentioned by name or sits behind an idea the lecture covers. If there's no free copy, link the DOI only.

## 6. Write `speaker.md` (only if there's a guest speaker)

Skip this step when the lecture has no guest. Use `cmu-15-445/14-query-execution-2/speaker.md` as the template.

1. **Identify the speaker.** Get the company from the slides (a "DB Flash Talk" slide names it) and the transcript. Garbled captions often drop the speaker's name. Confirm it from a primary source, such as the company's blog author list (its RSS feed lists authors), an "about" page, or a CMU event page. Never guess a name. If you can't confirm one, write "name not confirmed" and say so in the report.
2. **Write the file** with these sections:
   - an intro line linking back to the guest-talk section of `README.md`, with the talk's start timestamp
   - **The speaker:** role, background, and where each fact comes from
   - **The company:** what it builds, its products, open-source projects and funding, if a primary source states them
   - **Main ideas from the talk:** the points worth remembering, without repeating the README section in full
   - **Links:** the company site, blog, launch post, GitHub, and job page. Check each one returns `200` with curl.
3. **Don't repeat a company's marketing numbers as fact.** If you cite one, open the source it points to and check the number is there.

## 7. Update the syllabus and push

1. In `cmu-15-445/README.md`, set the lecture's **Completed** cell to today's date (`YYYY-MM-DD`).
2. Check that every relative link resolves to a file. Run this from the lecture folder:
   ```
   grep -ohE '\]\(([a-z]+/)?[a-z0-9-]+\.(md|html)(#[a-z0-9-]+)?\)' README.md speaker.md quiz/*.md 2>/dev/null |
     sed -E 's/^\]\(//; s/(#.*)?\)$//' | sort -u |
     while read f; do test -f "$f" -o -f "quiz/$f" && echo "ok $f" || echo "MISSING $f"; done
   ```
3. Stage only the lecture folder and the syllabus. Run `git status --short` and confirm no PDF or `scripts/` file is staged.
4. Commit with a message like `Add CMU 15-445 lecture NN notes and quiz`.
5. **Push.** The user wants every finished lecture synced to GitHub.

## Report back

Tell the user:
- which files were added
- the quiz score
- which papers have a free PDF and which are DOI-only
- the commit hash
- anything you couldn't get, such as a blocked transcript or an unconfirmed speaker name
