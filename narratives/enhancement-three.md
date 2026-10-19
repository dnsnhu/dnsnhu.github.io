---
layout: default
title: "Enhancement Three: Databases"
description: "Migrated from a hand written SQLite helper to Room, redesigned the schema around a user foreign key, indexed the user and date columns, and moved summary figures into SQL aggregates."
permalink: /narratives/enhancement-three/
---


# Enhancement Three: Databases

[Back to the portfolio]({{ site.baseurl }}/) · [Original source]({{ site.baseurl }}/artifact/original/) · [Enhanced source]({{ site.baseurl }}/artifact/enhanced/)

Migrated from a hand written SQLite helper to Room, redesigned the schema around a user foreign key, indexed the user and date columns, and moved summary figures into SQL aggregates.

## Files in this enhancement

**Added (14)**

`AppDatabase.java`, `GoalEntity.java`, `LegacyCredential.java`, `LegacyCredentialTest.java`, `LegacyTimestampParser.java`, `LegacyTimestampParserTest.java`, `Migrations.java`, `RoomUserDao.java`, `RoomUserDaoAdapter.java`, `RoomWeightDao.java`, `RoomWeightDaoAdapter.java`, `UserEntity.java`, `WeightEntryEntity.java`, `WeightStats.java`

**Changed (5)**

`ServiceLocator.java`, `WeightDao.java`, `WeightItem.java`, `WeightAdapter.java`, `WeightRepository.java`

**Removed (3)**

`DatabaseHelper.java`, `SqliteUserDao.java`, `SqliteWeightDao.java`

Full source for every file is on the [enhanced source]({{ site.baseurl }}/artifact/enhanced/) page.

## Narrative

### Description of the Artifact

The artifact is an Android weight tracking application written in Java. I built it as Project Three in CS 360 Mobile Architecture and Programming. It is the same artifact I examined in my Module Two code review and the same one I restructured and then added an analytics layer to in the two previous enhancements.

The application registers and signs in a user, lists logged weights, adds and edits entries, stores a goal weight, and sends an SMS when the goal is reached. Persistence was SQLite through a hand written SQLiteOpenHelper with three tables for login, weights, and goal.

The data layer is where the worst problems in this artifact were. The weights table had no column saying who a row belonged to, so three tables existed and not one referenced another. The date column held the output of Date.toString, which is locale dependent, sorts alphabetically rather than chronologically, and cannot appear in a date range. There were no indexes beyond the implicit primary keys. And the upgrade path dropped all three tables and recreated them, so any schema change in a shipped version would have erased the user history it was supposed to carry forward.

### Justification for Inclusion and Summary of Improvements

I selected this artifact for the database category because its schema problems are the kind that cannot be worked around in application code. A missing foreign key is not a bug in a method. No amount of careful querying scopes data to its owner when the table has no owner column, and no index can serve a date range when the dates are stored as display text. Those are decisions made in the schema, and the only way to demonstrate fixing them is to change the schema.

The enhancement replaces the hand written helper with Room and redesigns the schema underneath it. The pieces worth looking at:

- Three entities with real relationships. A users table owns the data, and weight\_entries and goals both carry a userId foreign key with cascading delete. Deleting an account now takes its rows with it instead of leaving orphans pointing at an id that no longer exists.

- recordedAt as epoch milliseconds rather than display text. This is the change that unlocks the others, because a column that sorts and subtracts can be indexed, ranged over, and aggregated. Formatting moved to the point of display, where the device locale belongs.

- A composite index on userId and recordedAt, in that order, matching the only access path the application has: one user, in time order, sometimes over a range. I checked it rather than assuming. EXPLAIN QUERY PLAN on the old schema reports SCAN weights; on the new one the same question reports SEARCH weight\_entries USING INDEX index\_weight\_entries\_userId\_recordedAt with both the equality and the range covered.

- Summary figures as SQL aggregates. Count, minimum, maximum, mean, and the span of the series come back as one row from one query, over the index, instead of pulling every entry into memory to add them up.

- Every statement naming the owner. A delete that matches only on row id will remove somebody else’s row if an id is guessed or reused. Putting userId in the WHERE clause makes the database refuse rather than relying on the caller to check first.

- The credential split into algorithm, iteration count, salt, and hash columns. Keeping the work factor in a column of its own means finding every credential still on an old cost is a query rather than a scan through every row in Java.

- A versioned migration replacing the destructive upgrade, which is the centre of this enhancement and the thing I spent the most time on.

The division of work between SQL and Java is deliberate and I want to be explicit about it, because it looks like a reversal of what I argued last time. The previous enhancement kept the moving average, the sliding window extremes, the streaks, and the trend projection in Java, and they stay there. Those are sequential calculations where each result depends on the one before it. What moved into SQL is the set of figures an index can answer in a single pass. The dividing line is not where the code is easier to write, it is whether the question is one an index can serve.

Verification is 18 new JUnit tests covering the two parts of the migration that are pure Java, on top of the 66 already covering the earlier enhancements, and a separate harness that runs the migration SQL against a real populated SQLite database. That harness pulls the statements straight out of Migrations.java by name rather than from a copy, builds a version 1 database with three accounts and six entries, upgrades it, and then checks that every row survived, that no foreign key is violated, that deleting the owner cascades, and that the query plan uses the index.

### Course Outcome Coverage

I planned this enhancement to cover Outcome Four, on using well founded and innovative techniques, skills, and tools to implement solutions that deliver value. I met it. Room is the current standard persistence library on the platform rather than something I invented for the assignment, and the specific value it delivers here is when errors surface. Room verifies every query against the schema when the project is built, so a column renamed in an entity and missed in a query is a failed build instead of an exception thrown on a device the first time somebody opens that screen.

I also left something out on purpose, which is part of the same outcome. Room offers fallbackToDestructiveMigration, which makes a schema change compile and run immediately by wiping the database. Using it would have reproduced exactly the behavior I set out to remove. Without it, a missing migration is a crash during development, which is the right place to find out.

This is the outcome-coverage update I recorded in Module Four and then in my Module Five journal. My original code review had this category covering Outcome Four and Outcome Five, with the algorithms work covering Three and Four. Spreading Outcome Four across two enhancements was vague. It belongs here, where adopting the platform standard tool is the actual claim. Outcome Five stays with the first enhancement, where the credential and validation work was.

That leaves one outcome, the collaborative environments one. The code review addressed part of it by being written for another developer rather than for a grader. The ePortfolio in Module Seven is what finishes it.

### Reflection on the Enhancement Process

The migration was the hard part, and it was hard for a reason I did not anticipate. Writing it was straightforward. Convincing myself it was correct was not, because the failure mode is silent: an upgrade that loses data looks exactly like an upgrade that worked until somebody opens the app and finds their history gone, and by then the old rows are already dropped. Testing against a fresh install proves nothing at all, since a fresh install has nothing to lose. So I built a version 1 database, filled it with the awkward rows rather than the convenient ones, and upgraded that.

Doing it that way caught a real bug. My first version gave rows with unreadable dates a synthetic timestamp derived from the row id, placed far in the past. Every count and constraint check passed. What failed was an assertion that the series still descended in recorded order, because the two rows with unparseable dates had been shoved to the front of the series ahead of entries that came before them. The weights were all there and the order was wrong, which for an application whose entire purpose is a trend over time is close to useless. I replaced it with a pass that fills each gap from the rows either side of it, so an unreadable entry stays between the two it was recorded between. I would not have found that from a row count.

The awkward rows were worth more than the clean ones generally. I seeded a date written on a French device, an empty date, and a password sitting in the credential column in plaintext from before hashing existed. The plaintext one forced a decision I had not thought about: copying it into a column named passwordHash would have been a lie, and a worse one than leaving it visibly wrong, because the name would then assert something false about every row in the column. It gets marked unmigrated instead, which locks that account rather than silently accepting any password against it.

The other thing I learned is how much of this enhancement is really about the previous two. The session identity I added in the first enhancement did nothing at the time, because there was no owner column for it to match against. It only became access control once the foreign key existed. In the other direction, the analytics from the second enhancement assume the series arrives in recording order, which the old schema could only manage by accident. The three enhancements read as separate pieces of work and they are not; this one is what makes the first one mean something.

What I could not verify here is the Room build itself. Room generates its implementation through an annotation processor at build time, so the schema validation and the generated queries only exist once the project is assembled. Everything I could check without that, I did: the Java compiles, the pure functions are unit tested, and the migration SQL runs against a real database file. Confirming the generated layer agrees with the hand written migration is a Gradle build, and it is the first thing I will run.


[Back to the portfolio]({{ site.baseurl }}/)
