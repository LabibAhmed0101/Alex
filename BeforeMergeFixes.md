
## Part One — What is new

### Five new features

- **Bulk document import.** Point the system at a Google Drive folder, or select a batch of files from your computer, and it attaches them to the right leads in one pass — instead of opening each lead and uploading one at a time.
- **Leads from forwarded emails.** A lead that arrives as a forwarded email can be read automatically — business name, EIN, address, the owner's name and contact details — and turned into a lead with the attachments already on it. **This is switched off at the moment.** There is a question about it at the end of this document.
- **Hot and warm leads.** A lead can be marked Hot. It drops back to Warm on its own after three weeks.
- **The PSF authorisation form.** Enter the merchant's bank details and the amount on screen, have them sign on screen, and the completed form saves itself to the deal. It checks the routing number before printing anything, and refuses if the PSF amount has changed since the form was opened.
- **Large imports survive a dropped connection.** A five-hundred-row spreadsheet can now resume where it stopped instead of starting over, and retrying will not create anything twice.

### The rotation timer, now settled

- **The timer is fourteen days, flat.** The old seven days plus a three-day bonus is gone.
- **A deal past Sold no longer rotates away from its broker.** Docs Out, Signed and BV Completed had been running a live timer, which meant a deal could be handed to a different broker days before it funded — with the commission attached. Past Sold the deal stays with the broker who did the work.
- **Going backwards no longer buys time.** Moving a lead back a stage and forward again was starting a fresh fourteen days, every time, with no limit. The clock now only resets when a lead reaches a stage it has not reached before under its current broker.
- **A rotated lead arrives as a Cold Lead**, as your specification describes. Before this, a lead that rotated while at Shopped Out landed on the new broker still reading Shopped Out — a stage they had nothing to do with reaching.
- **Timers are counted in real calendar days, New York time**, so the clocks stay honest across the March and November clock changes rather than drifting by an hour.

## Part Two — What was found and fixed

### Things that could have put a merchant's details in the wrong place

- **Documents were being matched to leads by file name.** Your specification says plainly that the file names in those folders are random and must not be used — and it was using them anyway, with nothing showing the result to a person first. A file that happened to be named after the wrong business would have attached one merchant's bank statements to another merchant's lead, silently, with nobody ever seeing it happen. File names are no longer used at all, and every proposed match now waits for an admin to confirm it.
- **Merchant bank statements were being sent to an outside AI service.** Up to four documents at a time, as part of reading those forwarded emails. That has stopped completely — no attachment leaves the system now, regardless of any setting.
- **A screenshot in the code repository contained a real merchant's Social Security number**, along with their EIN, date of birth and thirteen phone numbers. It has been removed and the folder it sat in is now blocked so nothing like it can be added again. One further step on this needs your decision — see the end of this document.

### Things that would have cost money or credited the wrong person

- **A lead's source was being overwritten.** When an emailed lead matched a lead you already had, the system relabelled the source as "Gmail" and threw away where the lead actually came from. Since a source can be owed a cut of the deal, that is a money question rather than a label. It now keeps the original and makes a note that the email suggested otherwise.
- **Automatic approvals were credited to the wrong person.** Whoever last saved the intake settings was recorded as having approved every lead the system approved on its own — leads they had never seen. Worse, changing any unrelated setting quietly changed who that was. Automatic approval is now switched off until somebody is deliberately named for the role.

### Things that were not being recorded

- **Three actions that should have been in the audit log were not being recorded at all** — changing the intake settings, editing an emailed lead's details before approving it, and attaching a document from an email. All three are now logged with who did it, what changed and when.
- **One of those was being saved without a safety net.** If saving the intake settings had failed halfway, the system could have been left with the change recorded but not made, or made but not recorded. Both now happen together or not at all.
- Where a setting is changed, the log now keeps the old value alongside the new one — so a question like "when did the automatic approval threshold move, and from what?" can be answered from the log itself.

### Things that showed people more than they should see

- **Opening the email intake screen pulled every merchant's private details across at once** — the full text of every email ever received and everything extracted from it, including EINs, addresses, owner names, emails and phone numbers, for every message in the list. The screen only ever needed those details for the one message you have open, and that is now all it receives.
- **The document screen could be used to confirm a document existed on a lead a broker had no access to.** Trying to delete it gave a different answer depending on what kind of document it was. It now gives the same answer either way.
- **Emailed attachments were being filed as bank statements** whether they were or not. They now arrive tagged "Other" for you to re-tag, which is what your specification asks for.

### Things that were simply broken

- **The email reading feature had never worked.** It was pointed at an AI service that does not exist, so every attempt failed silently. That is now fixed — though the feature stays switched off pending your answer below.
- **A setting mismatch would have stopped new leads being created.** Two parts of the system disagreed about what values the Hot/Warm marking could take. On a database set up one way, every new lead would have been rejected outright. Found and fixed.
- **Two upgrade steps had been given the same number**, with instructions saying to apply them in numerical order. Renumbered, so there is no ambiguity about what runs when.
- **Three tests were failing** on the batch as it arrived. None turned out to be a new bug — two were checking for behaviour the system had deliberately changed, and one needed a setting the test itself should have provided. All three now pass.
- **Nine pieces of software were being installed on every deployment** to support a one-off task that had already been done. Removed.

