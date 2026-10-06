# Associate Data Practitioner Study Hub

My own notes and practice questions for the Google Cloud Associate Data Practitioner exam. Free and unofficial. 2026 edition.

I put this together while I was learning data work on Google Cloud. It is how I understand the topics, written in plain words, plus practice questions I made to test myself. If it helps you, you are welcome to all of it.

## Start here

**[Open the ADP Study Hub](https://claude.ai/artifact/BKoxgMe73TZksDEshXkXGY)**

It is one web page. It works on a phone or a computer, with no account and nothing to install.

| In the hub | What you get |
|---|---|
| Handbook | The four domains of the public exam guide in plain words, with flowcharts, decision trees and process maps |
| Practice 1 to 4 | Four sets of 60 practice questions, each with a 2 hour clock |
| Your result | A score at the end, split by domain |
| Review | A short reason under every option, so you see why the right one is right and why the others are wrong |
| Quick reference | "If the question says X, think Y", tool choosers and look-alike names |

## A two week plan

This is the plan I would follow. Change it to fit your own pace.

| Day | What to do |
|---|---|
| 1 | Read [Start here](handbook/00-start-here.md). Try Practice set 1 cold, just to see where you stand |
| 2 to 3 | Read [Domain 1](handbook/01-data-preparation-and-ingestion.md). Load a file into BigQuery yourself and query it. My [hands-on notes](hands-on/README.md) show what I built |
| 4 to 5 | Read [Domain 2](handbook/02-analysis-and-presentation.md). Write SQL in BigQuery every day |
| 6 | Read [Domain 3](handbook/03-pipeline-orchestration.md) and [Domain 4](handbook/04-data-management.md) |
| 7 | Do Practice set 2 with the clock on. Review every question you missed or guessed |
| 8 | Go back to the handbook sections your misses point to |
| 9 | Do Practice set 3. It is the hardest set, so a lower score is normal |
| 10 | Review set 3. Read [Traps I fell for](notes/traps-i-fell-for.md) |
| 11 | Do Practice set 4 |
| 12 | Review set 4. Repeat the set where you scored lowest |
| 13 | Read the [quick reference sheets](handbook/05-quick-reference.md) and the 2026 product renames |
| 14 | Rest |

Three things made the difference for me:

1. **Review every miss.** The learning is in the reasons, not in the score. Read why each wrong option fails.
2. **Count a guess as a miss.** If you were not sure, read the reason anyway.
3. **Get your hands on the tools.** Reading about BigQuery is not the same as running a query in it.

## What is in this repo

| Folder | What it holds |
|---|---|
| [handbook](handbook/README.md) | The handbook as markdown, one file per chapter, with the diagrams. Easy to read here on GitHub |
| [notes](notes) | [Questions I asked while studying](notes/plain-language-qa.md), with plain answers, and [Traps I fell for](notes/traps-i-fell-for.md) |
| [hands-on](hands-on/README.md) | What I learned working hands-on with Google Cloud: what I built, what it taught me and what tripped me up |
| [learning-notes](learning-notes/README.md) | My full working notes, long and unpolished, with every question I asked and my self-check questions |
| [maps](maps) | Two interactive maps I drew for myself: how data moves on Google Cloud, and the families of machine learning models. Download a file and open it in a browser |
| [offline](offline) | The whole hub as a single HTML file. Download it and open it in any browser |

## Official resources

Google's own pages are the source of truth. Use them first.

- [Official certification page](https://cloud.google.com/learn/certification/data-practitioner): format, price and how to register
- [Official exam guide (PDF)](https://services.google.com/fh/files/misc/v1.0_associate_data_practitioner_exam_guide_english.pdf): the public list of topics
- [Google Cloud documentation](https://docs.cloud.google.com/docs): every product named in these notes

## Good to know

- This is unofficial, personal study material. It is not affiliated with or endorsed by Google.
- Everything here is original: my own notes and my own practice questions, put together from the public exam guide, the public Google Cloud documentation and my own hands-on practice.
- Nothing here comes from the actual exam. The questions were finished before I took it and have not been changed from memory of it.
- These are my notes, so they can be wrong in places. Where they disagree with the Google Cloud docs, trust the docs.
- Google renamed several products in 2024 to 2026. The hub uses the new name and puts the old one in brackets, for example Managed Service for Apache Spark (used to be Dataproc).
- Found a mistake? Please open an issue and I will fix it.

## Good luck

Read, practice, and review your misses.

Created by [Reza Seifollahi](https://www.linkedin.com/in/mrezasf/), 2026.
