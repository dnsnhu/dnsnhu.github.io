---
layout: default
title: "Code Review"
description: "What the code review of the original CS 360 application found, by category."
permalink: /code-review/
---


# Code Review

[Watch the recording](https://youtu.be/Ki56DSV0wM0) · [Back to the portfolio]({{ site.baseurl }}/)

One correction to the recording. In the video I say the original had about six comments. The real count is fifteen one-line comments across the seven classes, with no Javadoc anywhere. I recounted them in the source while writing the Enhancement One narrative, and the written figure is the accurate one. The finding itself stands either way, because fifteen labels in 546 lines is still almost no documentation.

The review walks the original application, applies a checklist to it, and sets out the planned enhancements. Everything below is a finding from that review, and each one is answered by one of the three enhancements.

## Software design and engineering

- No layering. Four classes opened the database and ran their own SQL.
- Duplication. `saveWeight` and `updateWeight` were the same five lines apart from insert against update.
- Dead code. `DatabaseHelper.updateWeight` was never called, because the activity kept a private copy.
- Almost no documentation. Fifteen one-line comments in 546 lines, no Javadoc.
- No defensive programming. `Float.parseFloat` ran on raw text, so any non-numeric entry crashed the activity.
- Plaintext passwords, compared inside a SQL WHERE clause.
- No session. Sign-in gated the interface and not the data.
- A live bug: two files read the same preference with different defaults, so a fresh install could send a text to a user who never opted in.

## Algorithms and data structures

- The application computed nothing. No average, no trend, no extremes, no sorting.
- The main query had no WHERE, no ORDER BY, and no LIMIT, so the whole table loaded every time.
- Deleting one row re-queried everything and rebuilt the adapter.
- Nothing sorted. Rows looked chronological by accident.
- The weight was stored as a String, so it changed type three times on a round trip and any calculation would have had to parse the series first.

## Databases

- No relationships. Three tables and not one referenced another.
- The date column held `Date.toString` output, which is locale dependent, does not sort chronologically, and cannot appear in a range.
- No indexes beyond the implicit primary keys.
- `onUpgrade` dropped all three tables, so any schema change would erase the user's history.
- Write failures were invisible, because the return value of every insert was discarded.
