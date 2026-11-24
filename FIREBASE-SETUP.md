# Firebase setup for teacher accounts and sharing

This guide covers the Firebase Console steps for per-teacher students, sharing, and photos of writing. It takes about 10 minutes. You do **not** create any collections by hand: the app creates them the first time someone saves.

## How the data is laid out

```
progress/{teacherUid}                 one per teacher: their settings and the student they last worked with
progress/{teacherUid}/pupils/{id}     OLD layout from before sharing. Copied into "students" once, then left as a backup.

students/{pupilId}                    one per student
   ownerId      the uid of the teacher who created the student
   ownerEmail   shown to colleagues as "shared by …"
   name, lang, langName
   editors      ["colleague@school.org", …]   can log progress, assessments and writing
   viewers      ["helper@school.org", …]      can look but not change anything
   json         all of the student's progress, writing transcriptions and notes (as text)
   updatedAt, updatedBy

students/{pupilId}/photos/{photoId}   one photo of handwritten work (compressed JPEG, about 200–600 KB)
```

- A teacher sees the students where `ownerId` is their uid, plus the students whose `editors` or `viewers` list contains their email.
- Only the owner can rename a student, change the sharing lists or delete the pupil. Editors can change only the progress itself.
- The rules check all of this on Google's servers, so hiding a button in the app is not the only protection.

## Step 1: Sign-in methods
Firebase Console, then **Authentication**, then **Sign-in method**:
1. **Email/Password**: enabled (you already have this).
2. **Google**: enabled, with a support email chosen.

Then go to **Authentication**, **Settings**, **Authorized domains** and check that **`acruzale.github.io`** is in the list. Google sign-in only works on domains listed here. `localhost` is there by default, for testing.

## Step 2: Firestore rules
Firebase Console, then **Firestore Database**, then the **Rules** tab:
1. Delete everything in the editor.
2. Paste in the whole of [`firestore.rules`](firestore.rules) from this repository.
3. Click **Publish**.

The old rules only allowed `progress/{uid}`. The new rules keep that and add `students`. If you skip this step, the app shows "Firebase refused access".

**Why email verification matters:** sharing works by email, and anyone can sign up with email and password using any address. So the rules only give shared access when the email is **verified**. Google accounts are always verified. When a colleague creates an email-and-password account, the app sends them a verification link. Until they click it, they see a "Verify your email" box and no shared pupils. Their own students work as normal.

## Step 3: Indexes
Nothing to do. The app's queries (`ownerId ==`, `editors array-contains`, `viewers array-contains`) use the indexes Firestore creates automatically.

## Step 4: Publish the app
Push `index.html` to `main`. The first time each teacher logs in, their existing students are copied from the old layout into `students` automatically. The old copies stay where they were as a backup. You can delete `progress/{uid}/pupils` in the console later if you want to.

## How to share a student (in the app)
1. **👥 Manage students**, then **🔗 Share** next to the pupil.
2. Type the colleague's email, choose **Can edit** or **View only**, then tap **Share**.
3. The colleague logs in with that exact email. The student appears under "Shared with me", either straight away or when they tap **↻ Check for shared students** in their Account box.
4. To change the access, use the drop-down next to their email. To take it away, tap **Stop sharing**.

## Photos: why Firestore and not Firebase Storage
Photos are kept in Firestore, as compressed JPEGs in each student's `photos` subcollection. This keeps the setup simple:
- **No paid plan.** Since late 2024, new Cloud Storage buckets (like `sounds-and-words.firebasestorage.app`) need the Blaze pay-as-you-go plan. Firestore works on the free Spark plan: 1 GB of storage is roughly 2,000–4,000 photos, and there are 50,000 reads a day.
- **One set of rules.** Photo access follows the student's sharing automatically.
- **No CORS setup.** The app reads photos through the Firestore SDK, so GitHub Pages needs no extra configuration.

Limits to know about:
- The app shrinks each photo to fit Firestore's 1 MB limit (up to 1600 px on the long side). That's plenty for reading a child's handwriting.
- Photos are **not** in the "Download backup" file. They stay in the account.
- Firebase Storage would make sense later, if you collect many thousands of photos or need full-resolution scans. That would need the Blaze plan, Storage rules that mirror the student's sharing, and a CORS setting on the bucket (`gsutil cors set`) for `https://acruzale.github.io`.

## Checking the rules in the console (optional)
**Firestore**, then **Rules**, then **Rules Playground** lets you simulate a request. For example: choose `get` on `students/SOME_ID`, turn on Authenticated, set the email in the token to a colleague's address with `email_verified` set to true, and check that it is allowed only if that email is in the student's `editors` or `viewers`.
