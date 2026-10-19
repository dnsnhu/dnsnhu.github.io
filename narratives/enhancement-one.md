---
layout: default
title: "Enhancement One: Software Design and Engineering"
description: "Restructured the application around MVVM, moved database work off the UI thread, centralized input validation, and replaced plaintext password storage with salted PBKDF2 hashing."
permalink: /narratives/enhancement-one/
---


# Enhancement One: Software Design and Engineering

[Back to the portfolio]({{ site.baseurl }}/) · [Original source]({{ site.baseurl }}/artifact/original/) · [Enhanced source]({{ site.baseurl }}/artifact/enhanced/)

Restructured the application around MVVM, moved database work off the UI thread, centralized input validation, and replaced plaintext password storage with salted PBKDF2 hashing.

## Files in this enhancement

**Added (32)**

`AddWeightActivity.java`, `AddWeightViewModel.java`, `AppExecutors.java`, `AuthenticatedUser.java`, `Callback.java`, `DatabaseHelper.java`, `LoginActivity.java`, `LoginViewModel.java`, `Outcome.java`, `PasswordHasher.java`, `PasswordHasherTest.java`, `ServiceLocator.java`, `SessionManager.java`, `SettingsActivity.java`, `SettingsPreferences.java`, `SettingsViewModel.java`, `SqliteUserDao.java`, `SqliteWeightDao.java`, `User.java`, `UserDao.java`, `UserRepository.java`, `UserRepositoryTest.java`, `ValidationResult.java`, `Validator.java`, `ValidatorTest.java`, `ViewModelFactory.java`, `WeightAdapter.java`, `WeightDao.java`, `WeightItem.java`, `WeightListActivity.java`, `WeightListViewModel.java`, `WeightRepository.java`

Full source for every file is on the [enhanced source]({{ site.baseurl }}/artifact/enhanced/) page.

## Narrative

### Description of the Artifact

The artifact is an Android weight tracking application written in Java. I built it as Project Three in CS 360 Mobile Architecture and Programming. It is the same artifact I examined in my Module Two code review.

The application does what the CS 360 assignment asked for. A user registers an account and signs in. The main screen lists logged weights in a RecyclerView, with a button on each row to delete the entry and a tap on the row to edit it. A second screen adds or edits a weight and optionally sets a goal weight, and it sends an SMS notification when the logged weight reaches that goal. A settings screen holds an SMS opt-in switch and a button that deletes all weight data. Persistence is SQLite through a SQLiteOpenHelper subclass with three tables for login, weights, and goal.

The original is 546 lines across seven classes: LoginActivity, WeightListActivity, WeightAdapter, WeightItem, AddWeightActivity, SettingsActivity, and DatabaseHelper. It works. The problems are structural rather than functional, which is what made it worth enhancing.

### Justification for Inclusion and Summary of Improvements

I selected this artifact because its weaknesses are the kind a software design and engineering enhancement is meant to address. The application behaves correctly, so there was no functionality to repair. What it lacked was structure, and the code review found five distinct structural problems.

There was no layering, because four classes opened the database and ran their own SQL. There was duplication, because saveWeight and updateWeight in AddWeightActivity were the same five lines apart from insert versus update, and DatabaseHelper carried a dead updateWeight that nothing called. There was almost no documentation, fifteen one-line comments in 546 lines, all of them labels rather than explanations, and no Javadoc anywhere. There was no defensive programming, so Float.parseFloat ran on raw text and any non-numeric entry crashed the activity. And there were two security problems: passwords stored in plaintext, and a login that produced no session, so nothing downstream knew who had signed in.

The enhanced version restructures the application into MVVM. Activities no longer touch the database at all. A repository layer sits between the UI and persistence, DAO interfaces sit under the repositories, and each screen has a ViewModel that exposes state as LiveData. The list screen observes its data instead of re-querying the whole table in onCreate, again in onResume, and again after every delete, which is what the original did.

The components I would point a reviewer at are these:

- WeightRepository and UserRepository, which are now the only path to stored data, and the WeightDao and UserDao interfaces underneath them. The interfaces are deliberate. The storage implementation can be replaced without any change to the repositories, the ViewModels, or the screens.

- AppExecutors, which moves every database call off the main thread. In the original, every query ran inside a click listener on the main thread, which blocks the UI and risks an Application Not Responding dialog once the table grows.

- Validator, which holds every input rule in one place and has no Android imports. It parses inside a try block and reports a message instead of throwing, and it applies a plausible range check the original never had.

- PasswordHasher, which replaces plaintext storage with PBKDF2 using HMAC-SHA256, a random per-password salt, and 120,000 iterations. Verification uses MessageDigest.isEqual so the comparison does not return early on the first differing byte.

- SessionManager, which gives the application an actual identity for the signed-in user, and a check in WeightListActivity that sends an unauthenticated user back to the login screen.

- SettingsPreferences, which fixes a live bug. AddWeightActivity read the sms\_enabled preference with a default of true and SettingsActivity read the same key with a default of false. On a fresh install the two disagreed, so the application could request the SEND\_SMS permission and send a message to a user who had never opted in. The key and the default are now declared once, and the default is opt-out.

The measurable change is that four classes touched the database before and zero activities touch it now, and that the project went from no tests to 31 passing JUnit tests. The tests matter more than the count. The original could not be unit tested at all, because the sign-in logic lived inside a click listener next to a concrete DatabaseHelper and needed an emulator to reach. Because the DAOs are interfaces and the executors are passed through a constructor, UserRepositoryTest now exercises the real registration and sign-in logic on the JVM with a fake data store. That is the difference the restructuring actually bought.

### Course Outcome Coverage

In Module One I planned this enhancement to cover Outcome Four on well-founded techniques and tools, and Outcome Five on a security mindset, with partial coverage of Outcome Three on managing design trade-offs. I met all three.

Outcome Four is covered by the MVVM restructuring itself. Repository, DAO, ViewModel, and LiveData are standard Android architecture rather than invented structure, and the layering is what makes the two later enhancements possible without rewriting the screens.

Outcome Five is covered by the credential work and by the two access problems. Salted iterated hashing and a constant-time comparison address the stored credential. The session holder addresses the fact that the original login gated the interface and not the data. The SMS default bug is the third piece, and it is the one I would call a genuine security finding, because neither file was wrong on its own.

Outcome Three is partial, as planned. The trade-off I made and can defend is where the password salt lives. Storing the salt in a column of its own is the cleaner schema, but the upgrade path in this application drops every table and recreates them, so introducing a column would destroy a user’s stored history. Redesigning the schema is also not a software design and engineering change. I packed the format identifier, iteration count, salt, and hash into the existing password column instead. The encoded string carries its own iteration count, so the work factor can be raised later without invalidating stored credentials.

I have one update to my outcome-coverage plan. I did not claim Outcome Two for this enhancement in Module One, and I still do not, but the Javadoc added across the new classes is written for a reader rather than as labels, and I plan to draw on it when I write the ePortfolio pages that have to explain this work to a non-specialist audience. Beyond that, my coverage plan is unchanged.

### Reflection on the Enhancement Process

The hardest part was scope, not code. Every problem in this application connects to another one, and the pull to fix all of them at once was constant. The session identity is a good example. It establishes who is signed in, but the weights table has no column to record an owner, so nothing is scoped by it yet. Adding that column is a schema change, which is not what a software design and engineering enhancement is for. The timestamp is the same story. The repository still writes the locale-dependent output of Date.toString, and I know that is a defect, but correcting it means changing a column type. What helped was writing a short note in the Javadoc of each class recording what I left alone and why. That kept the boundary visible while I worked instead of something I had to hold in my head.

The threading change taught me more than I expected. Moving work to a background executor is straightforward, but LiveData has two different setters and only one of them is safe off the main thread. Getting that wrong does not fail loudly, so I ended up being deliberate about which layer publishes and which layer delivers. Single-shot signals gave me the same kind of trouble. A LiveData holding a boolean for navigation re-delivers its last value when the screen is recreated, so the screen navigates again after a rotation. I settled on having the view call an explicit handled method that clears the flag.

Two things surprised me. The first was a compile error I did not expect: I wrote a multi-catch for NumberFormatException and IllegalArgumentException, and Java rejected it because the first is a subclass of the second. That was a quick fix once I read the error, but it is the kind of thing I would not have found without compiling, and it is a reminder that I should not trust code I have only read. The second was thinking through the sign-in path. My first version skipped password verification entirely when the username was not found, which is the obvious implementation. It also means an unknown username returns noticeably faster than a known one, which tells an attacker which accounts exist. Verifying against a deliberately unmatchable hash costs nothing to write and removes the difference. I would not have thought about it at all if the code review had not put me in the habit of asking what an attacker sees.

The main challenge left is verification breadth. The 31 tests cover validation, hashing, and the authentication repository, which are the parts that are pure Java. The ViewModels are testable in principle now, but covering them needs a test rule for LiveData that I have not set up yet. I would rather say that plainly than claim coverage I do not have.


[Back to the portfolio]({{ site.baseurl }}/)
