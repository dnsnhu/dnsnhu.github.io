---
layout: default
title: David Noh | Computer Science ePortfolio
---

# David Noh

**Computer Science, B.S. | Southern New Hampshire University**

This portfolio documents a single application taken apart and rebuilt at three
levels: its architecture, its algorithms, and its database. Each section covers what
the original code did, what was wrong with it, and what changed.

[GitHub](https://github.com/dnsnhu · [Email](mailto:david.noh@snhu.edu)

---

## Professional Self-Assessment

*Coming in Module Seven.*

This section will introduce the portfolio as a whole and cover collaboration in a
team environment, communication with stakeholders, data structures and algorithms,
software engineering and databases, and security.

---

## The Artifact

**Weight Tracking Application** — Android, Java, SQLite
Originally submitted as Project Three in CS 360: Mobile Architecture and Programming.

The original application works, and that is roughly all that can be said for it.
Activities call the database directly from click listeners on the UI thread. The
weights table has no user column, so every account registered on a device shares
one undifferentiated list of entries. Dates are stored as locale-dependent display
strings that cannot be sorted or range-queried. Passwords sit in a plaintext column.
The app stores data it never computes anything from.

Each of those is a different kind of problem, which is why one artifact carries all
three enhancement categories.

[Original source](./artifact/original/) | [Enhanced source](./artifact/enhanced/)

---

## Code Review

*Coming in Module Two.*

A walkthrough of the existing application, an analysis of its structure, logic,
efficiency, security, testing, and documentation, and the reasoning behind each
planned enhancement.

[Watch the code review](#)

---

## Enhancement One: Software Design and Engineering

*Coming in Module Three.*

Restructuring the application into an MVVM architecture with a repository layer and
ViewModels, moving all I/O off the UI thread, centralizing validation, and replacing
plaintext password storage with salted PBKDF2 hashing and a constant-time comparison.

[Read the narrative](./narratives/enhancement-one/) | [View the changes](#)

---

## Enhancement Two: Algorithms and Data Structures

*Coming in Module Four.*

Adding an analytics layer computing a moving average series, windowed minimum and
maximum, streak detection, and a goal projection. Each has an obvious quadratic
implementation and a linear one. The running sum computes the moving average in O(n)
regardless of window size, and a monotonic deque gives amortized O(1) per element for
the windowed extremes.

[Read the narrative](./narratives/enhancement-two/) | [View the changes](#)

---

## Enhancement Three: Databases

*Coming in Module Five.*

Replacing the hand-written SQLiteOpenHelper layer with Room, redesigning the schema
around an owning user entity with a foreign key and cascading delete, storing
timestamps as sortable integers, adding a composite index on the application's primary
access path, moving aggregates into SQL, and replacing the destructive upgrade path
with a tested migration.

[Read the narrative](./narratives/enhancement-three/) | [View the changes](#)

---

## Course Outcomes

| Outcome | Where it is demonstrated |
|---|---|
| Collaborative environments | Code review, in-code documentation |
| Professional communication | Code review, narratives, this site |
| Algorithmic design and trade-offs | Enhancement Two |
| Well-founded techniques and tools | Enhancements One and Three |
| Security mindset | Enhancements One and Three |

---

<sub>CS 499 Computer Science Capstone, Southern New Hampshire University</sub>
