---
layout: default
title: "David Noh | Computer Science ePortfolio"
description: "Computer science ePortfolio for the Southern New Hampshire University capstone, showing one Android application rebuilt across software design, algorithms, and the data layer."
permalink: /
---


# David Noh

Computer Science, Southern New Hampshire University

[GitHub](https://github.com/dnsnhu) · [Email](mailto:david.noh@snhu.edu)

---

## Professional Self-Assessment

Early in the Southern New Hampshire computer science program I could usually get something working, and I could often tell when a design was going wrong without being able to say why. Tacit judgment is hard to defend in a code review and impossible to put on a resume. What the program gave me is the vocabulary and the formal grounding to make that reasoning explicit, and this portfolio is where it shows most clearly.

The capstone turned out to be the part of the degree that pulled everything together, because it asked me to do something harder than writing new code. It asked me to go back to code I wrote earlier, say plainly what was wrong with it, and fix it in a way I could defend. Reading your own old work honestly is uncomfortable, and it is also the single most useful habit the program gave me.

**Collaborating in a team environment.** Most of what I learned here came from writing for other developers rather than for a grader. The code review that opens this portfolio is the clearest example. I recorded a walkthrough of an application I wrote, named its weaknesses without softening them, and explained what I intended to do about each one. Writing it for a peer rather than an instructor changed the content: a peer does not want to be told the code works, they want to know where it breaks and what I plan to do. That same discipline showed up in CS 405, where the Green Pace secure development policy I wrote had to be specific enough for another engineer to apply, so each of the ten standards was mapped to a SEI CERT rule rather than left as general advice. Comments in code are part of this too. The enhancements in this portfolio document why a decision was made where a maintainer would otherwise be tempted to undo it, and stay quiet everywhere else, because a comment on every line trains people to stop reading them.

**Communicating with stakeholders.** CS 319 taught me more about this than any other course, mostly by making me wrong first. Designing a flip-phone interface for Kiva borrowers, I produced a color-coded layout that looked clean and was unusable on the hardware those borrowers actually carry, which is a USSD menu on a monochrome screen. Redoing it in black and green monospace with dashed separators was not a visual preference, it was accepting a constraint I had initially designed around. The CareGuardian client communication report in the same course had a related lesson: the audience was a caregiver, not an engineer, so wireframes across four mobile screens, a smartwatch, and a kiosk had to be explained in terms of what the person would see rather than how it was built. CS 465 made the same point in a different register, where the Travlr Getaways software design document had to hold up as a contract between a developer and a client who would never read the code.

**Data structures and algorithms.** The second enhancement in this portfolio is where this became concrete rather than academic. Tracking a sliding minimum and maximum over a weight log is the kind of problem where the obvious implementation, rescanning the window on every entry, is fine on a short log and costs more and more as it grows. Replacing it with a monotonic deque holds the work per entry flat no matter how wide the window gets. What I value more than the structure itself is what happened when I measured it: at the window size this application actually uses, the deque was barely faster than the rescan, and I had already written down that it would win. Keeping the structure and rewriting the justification around how the two costs grow, rather than around one favorable number, was the right call and not the comfortable one. CS 370 pushed in a similar direction, where a deep Q-learning agent only became stable once experience replay and a target network were in place, and I could explain what each one was fixing rather than that the combination worked.

**Software engineering and databases.** The third enhancement replaced a hand-written SQLite helper with Room, redesigned the schema around a user foreign key, moved timestamps to an integer column that can actually be indexed and ranged over, and pushed summary figures into SQL aggregates. The part I would point an employer at is the migration. The first version of the application upgraded by dropping every table, which means any schema change would have erased the user's history. Writing the replacement was quick. Convincing myself it was correct meant building a populated version one database, filling it with the awkward rows rather than the convenient ones, and upgrading that. It caught a real defect: entries whose dates could not be parsed were being pushed to the front of the series, so the data survived and the order was wrong. CS 465 and the MEAN stack work gave me the broader picture of how a data layer sits under an application, and CS 350 showed the same separation on much smaller hardware, where a thermostat state machine on a Raspberry Pi had to keep its sensor reads, its state transitions, and its output on separate paths to stay debuggable.

**Security.** CS 405 is where I learned to treat this as a design concern rather than a checklist applied at the end. Buffer overflows, integer overflow and underflow, exception handling, and static analysis with two different tools on the same file all made the same point: most vulnerabilities are ordinary bugs in places where the consequences are not ordinary. The first enhancement in this portfolio applies that directly. Passwords were stored in plaintext, so they became salted PBKDF2 hashes compared in constant time. Input validation was scattered across screens, so it moved into one class with real rules. Sign-in produced no session, so nothing downstream knew who was using the application, and the login gated the interface rather than the data. That last one only finished in the third enhancement, once the schema had an owner column for the identity to match against. The finding I am most pleased with came out of the code review rather than the code: two files read the same preference with different defaults, so on a fresh install the application could send a text message to a user who had never opted in. Neither file was wrong on its own.

Taken together, the three enhancements are one application rebuilt at three levels rather than three separate exercises, and they interlock on purpose. The session identity added in the first enhancement does nothing until the foreign key in the third gives it rows to match. The analytics in the second assume the series arrives in recording order, which the original schema could only manage by accident and the third enhancement guarantees. Reading them in order shows architecture, then algorithms, then the data layer, and shows each one making the next possible. Everything that follows is that same weight tracking application: the original source as I wrote it in CS 360, the enhanced source, and a narrative for each enhancement explaining what changed and why.

I am aiming at backend and systems work, which is the part of this field I find interesting rather than the part that was easiest to write about. The next thing I plan to pick up is Rust, for the same reason the security coursework stuck with me, and a cloud certification before any thought of graduate school. What this program gave me that I did not have before is the ability to say why, and to be specific about it.

---

## The Artifact

A weight tracking application for Android, written in Java against SQLite. I built it as Project Three in CS 360 Mobile Architecture and Programming. It registers and signs in a user, logs weights, tracks a goal, and sends a notification when the goal is reached.

It works, and the problems are structural. Every database call ran on the UI thread. The weights table had no column saying who a row belonged to. Dates were stored as display text that sorts alphabetically and differs by locale. Passwords were stored in plaintext. Upgrading the schema dropped every table.

- [Original source]({{ site.baseurl }}/artifact/original/), the seven classes as I wrote them in CS 360
- [Enhanced source]({{ site.baseurl }}/artifact/enhanced/), all 53 source files after the three enhancements

---

## Code Review

A recorded walkthrough of the original application: what it does, what is wrong with it against a review checklist, and what I planned to change in each of the three categories.

[Watch the code review](https://youtu.be/Ki56DSV0wM0) · [What the review found]({{ site.baseurl }}/code-review/)

---

## Enhancement One: Software Design and Engineering

Restructured the application around MVVM, moved database work off the UI thread, centralized input validation, and replaced plaintext password storage with salted PBKDF2 hashing.

- 31 JUnit tests, up from none
- Four classes opened the database before, zero activities do now

[Read the narrative]({{ site.baseurl }}/narratives/enhancement-one/) · [What changed]({{ site.baseurl }}/narratives/enhancement-one/#files-in-this-enhancement)

---

## Enhancement Two: Algorithms and Data Structures

Added an analytics layer: a single pass moving average, a monotonic deque for the sliding minimum and maximum, streak detection, and a least squares goal projection.

- 4.00 deque operations per entry at every window size tested
- Moving average 0.51 ms against 4.98 ms at 100,000 entries
- 66 JUnit tests, 35 of them new

[Read the narrative]({{ site.baseurl }}/narratives/enhancement-two/) · [What changed]({{ site.baseurl }}/narratives/enhancement-two/#files-in-this-enhancement)

---

## Enhancement Three: Databases

Migrated from a hand written SQLite helper to Room, redesigned the schema around a user foreign key, indexed the user and date columns, and moved summary figures into SQL aggregates.

- SCAN weights became SEARCH ... USING INDEX index_weight_entries_userId_recordedAt
- Migration verified against a populated version 1 database
- 84 JUnit tests, 18 of them new

[Read the narrative]({{ site.baseurl }}/narratives/enhancement-three/) · [What changed]({{ site.baseurl }}/narratives/enhancement-three/#files-in-this-enhancement)

---

## Course Outcomes

| Outcome | Where it is demonstrated |
| --- | --- |
| Collaborative environments | The code review, written and recorded for another developer. In-code documentation that explains a decision where a maintainer would otherwise undo it. |
| Professional communication | The code review video, the three narratives, and this portfolio. |
| Algorithmic principles and trade-offs | Enhancement Two: the monotonic deque, the single pass moving average, and a measured argument for the structure that did not go the way I expected. |
| Well-founded techniques and tools | Enhancement One: MVVM, repository and DAO layering, ViewModel and LiveData, which are the standard Android structures rather than invented ones. Enhancement Three: Room, a redesigned schema, indexing, and a tested non-destructive migration. |
| Security mindset | Enhancement One: salted PBKDF2 credentials, centralized validation, a real session. Enhancement Three: a foreign key that makes the session scope data rather than only the interface. |

---

CS 499 Computer Science Capstone, Southern New Hampshire University
