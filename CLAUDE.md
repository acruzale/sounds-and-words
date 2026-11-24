# Sounds and Words: notes for Claude Code

A teacher-led phonics and vocabulary web app for a German-English bilingual primary school. It's used one-to-one with EAL pupils (Spanish, German) and with whole classes (English only). The teacher operates it, not the pupils.

## How it's built
- **One file: `index.html`.** HTML, CSS and plain JavaScript all live in it. There's no framework and no build step. Keep it as one file.
- **Hosted on GitHub Pages.** Pushing to `main` publishes it.
- **External resources:** Google Fonts (Andika for letters and words, because it has the single-storey a and g used in phonics; Fredoka for headings), and the Firebase compat SDK 10.12.2 from gstatic.
- **Audio:** the browser's own `speechSynthesis`, preferring an en-GB voice. It only ever reads whole English words and phrases. Never use it for single phonics sounds (computer voices say "tuh" or "em"), and never for Spanish or German (it reads them with an English accent).

## Code layout (sections in the script, in order)
1. **DATA:** `SOUNDS` (RWI Set 1, 2, 3 speed sounds with chart words or phrases and Spanish tips), `LEVELS` (blending words; sound buttons written as `c-a-t`, split digraphs as `c-a*-k-*e`), `TOPICS` (vocabulary).
2. **LANGUAGES:** `DE` (German meaning for every built-in word, keyed `"english|spanish"` so words with two meanings get the right one), `TOPIC_DE`, `SOUND_DE` (German pronunciation tips), `LANGS` (`en`, `es`, `de`, `other`), and helpers `curLang()`, `L()`, `mKey()`, `meaning(w)`, `blendMeaning(w)`, `cheer()`, `tryAgain()`, `withL()`.
3. **ASSESSMENT DATA:** Phase 2 and Phase 3 (Letters and Sounds) assessment sheets.
4. **NELSON:** Nelson Spelling Pupil Book 1A, units 1–14. Each word has flags: `k` = key word, `t` = tricky word.
5. **STATE**, **HELPERS**, **ROUTING:** hash routes `#sounds #blending #vocabulary #writing #assess #progress`.
6. One render function per tab: `renderSounds`, `renderBlending`, `renderVocab` (plus editors), `renderAssess`, `renderWriting`, `renderProgress`.
7. **LOG-IN AND CLOUD SAVE:** Firebase.
8. **EVENTS:** all clicks go through one delegated listener. Buttons use `data-act="name" data-arg="…"`, and the handlers live in the `A` object.

## Patterns to follow
- **Rendering:** each render function rebuilds `app.innerHTML` from `state` and `ui`. To change something: update `state`, call `save()`, then call the render function.
- **Saving:** `save()` stamps `updatedAt`, writes localStorage (`persist()`) and schedules the cloud save. Use `persist()` alone when the change shouldn't count as an edit.
- **Confirmations are in-page.** `alert`, `confirm` and `prompt` are not allowed. Use the `ui.confirm` pattern instead.
- **Scroll position:** after a button press the page must stay exactly where it is. The click listener restores `scrollY`. Only actions listed in `NEW_SCREEN` scroll to the top. Never add `window.scrollTo(0,0)` to Next or Blend buttons: this was a real complaint during assessments.
- **Text input:** typing in note fields updates `state` through the `input` listener, without re-rendering, so focus isn't lost.
- **Escaping:** all user text goes through `esc()`.
- **Neutral wording:** on-screen text says "student" (never "pupil", except the book title "Nelson Spelling Pupil Book 1A") and uses they/them, never she/her or he/him. Code names (`state.pupils`, `pupilName`, the old Firestore `pupils` subcollection) stay as they are.

## Languages
- **Each pupil has a home language:** `en` (the default; English only, no translations, no tips), `es`, `de`, or `other` (the teacher types the language name; translations start empty).
- **Meanings are stored per language** on each word: `es`, `de`, `tr` (other). Read them with `meaning(w)`. When saving an edit, write to `w[mKey()]` and keep the other language fields.
- **Pronunciation tips exist only for Spanish and German.** Never add tips for other languages: the teacher can't check them.
- **Pupils saved before languages existed have no `lang`:** treat them as Spanish.

## Pupils and data
- **Shared per account:** `settings` (speaking speed, show-translation, Explore language).
- **Separate per pupil** (`PUPIL_KEYS`): sounds, stars, blends, assessments, blendSessions, writing, draft, custom (own word lists) and topicEdits (added or removed words in built-in topics).
- **In memory:** the current pupil's data sits flat on `state` (a copy). The other pupils are in `state.pupilStore[id]`. Switch with `switchPupil(id)`; copy the current pupil back with `stashCurrent()`, never by hand.
- **Explore mode** (not logged in): no pupils, no name shown, progress stays only in that browser. Logging out always wipes the computer back to Explore mode.

## Firebase
- **Auth:** email and password, and Google (popup). Sign-up is open, so teachers can create their own accounts. Anonymous log-in is off. New email accounts get a verification email.
- **Firestore** (rules in `firestore.rules`, setup in `FIREBASE-SETUP.md`):
  - `progress/{uid}` holds `{json, updatedAt}`: the teacher's own record (version 3: settings and current pupil).
  - `students/{pupilId}`: one pupil. Fields `ownerId`, `ownerEmail`, `name`, `lang`, `langName`, `editors[]`, `viewers[]` (lower-case emails), `json` (the pupil data), `updatedAt`, `updatedBy`. JSON strings are used because Firestore can't store nested arrays.
  - `students/{pupilId}/photos/{photoId}` holds `{data}`: a compressed JPEG data URL of handwritten work (kept under 1 MB). Firestore is used instead of Storage, so no paid plan is needed.
- **Roles:** each entry in `state.pupils` has `role`: `owner`, `edit` or `view` (read it with `roleOf(p)`; a missing role means owner). Only the owner renames, shares or deletes. Editors may change only `json`, `updatedAt` and `updatedBy`. For view-only pupils, `save()` undoes the change at once (`restorePupilData`), and editing buttons are hidden.
- **Shared access is by verified email** (`request.auth.token.email_verified`), because anyone can sign up with any address.
- **Save order:** write pupil documents first, then `progress/{uid}`. Mark a pupil as changed with `touchPupil(id)` (`save()` does this for the current pupil).
- **Conflicts:** compared per pupil, by `p.t` (local) against `updatedAt` (cloud); the newer copy wins. Right after logging in, the cloud copy wins.
- **Old data:** version 1 and 2 accounts (pupils under `progress/{uid}/pupils`) are copied into `students` once by `migrateOld()`. The old documents are left as a backup.

## Testing
- **No test suite in the repo.** Test in a browser.
- **With Playwright**, serve or open the file. Swap the gstatic Firebase scripts for a small mock that implements `auth()` and `firestore().collection().doc().collection()` on localStorage.
- **Password minimum:** the log-in form needs passwords of at least 6 characters.
- **Clicks and scrolling:** Playwright scrolls an element into view before clicking it. To test scroll position, click through `element.click()` in the page instead.
- **Printing:** check the print layout with `emulate_media(print)` and `page.pdf`. A Phase 3 assessment should fit on 2 A4 pages.

## Things the teacher cares about
- **A calm, predictable experience:** no jumping, no surprises. Teacher-only information is clearly marked.
- **Matching the physical resources:** RWI chart words and phrases, Nelson 1A unit words exactly as printed in the book, and the Phase 2/3 sheets.
- **Privacy:** pupils are stored by first name only.
