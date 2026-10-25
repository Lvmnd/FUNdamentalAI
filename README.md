# FUNdamental AI Lab Quiz

A single-file, Firebase-backed quiz application for the **FUNdamental AI** course. It provides three practice sets and one graded real test, participant accounts, and an admin dashboard for monitoring and exam control.

The interface is bilingual (**English / Indonesian**) and works on desktop and mobile browsers. No build step, framework, or server is required.

> **Status:** v1.0.0 · question bank `2026-10-25.1` · validated 2026-10-25

---

## Features

### For participants

- Self-registration with **NIM** (student ID number), full name, and password.
- Mandatory acceptance of the Terms & Conditions, recorded with version and timestamp.
- Three practice sets (`Trial 1–3`), each available once, with immediate answer review.
- One graded real test: **30 questions**, **45 minutes**, **60% to pass**.
- Real-test answers stay hidden until the admin releases them.
- Answers and progress are saved to Firestore, with a local backup draft in the browser.
- Progress survives a page reload; closing the tab ends the session.
- EN / ID language switch and light / dark theme.

### For administrators

- Dedicated admin login, separate from participant login.
- Open or close registration and practice mode.
- Schedule the real test with an optional open/close window.
- Enable/disable the real test and release/lock its answers.
- Monitor participants, drafts, submissions, scores, durations, tab switches, and audit events.
- Recompute scores from stored answers for an integrity check.
- Reset a participant password with a temporary password.
- Disable or re-enable a participant account.
- Export results and participant data as CSV.
- View the embedded question bank and its SHA-256 fingerprint.

### Integrity controls

- Firestore Security Rules identify the admin by Firebase Auth UID **and** email.
- Participants can read only their own profile, drafts, and results.
- The real test is accepted only while the admin's switch is on and the server time is inside the configured window.
- One attempt per set is enforced by deterministic document IDs and create-only rules.
- Submissions are immutable from the participant side.
- Audit logs are append-only and readable only by the admin.
- Registration is refused server-side when the admin closes it.

---

## Tech stack

| Layer | Technology |
| --- | --- |
| UI | Vanilla HTML, CSS, and JavaScript |
| Auth | Firebase Authentication (email/password) |
| Database | Cloud Firestore |
| Hosting | Any static host, including GitHub Pages |
| Build | None — one `index.html` file |

Firebase SDK version: `10.14.1` (compat builds, loaded from `gstatic.com`).

---

## Repository layout

```text
FUNdamental AI/
├── index.html          # The entire application (UI + Firebase + question bank)
├── README.md           # This file
├── firestore.rules     # Security rules (publish separately in Firebase Console)
├── SETUP.md            # PRIVATE operations guide — never commit this file
├── validation/         # Local Firestore rules regression suite
├── test/               # Original CSV question sources
└── learning material/  # Original course slides
```

Only `index.html` is required to run the app. `firestore.rules` must be published to Firebase separately.

> **Never commit `SETUP.md` or any file containing admin credentials, real participant data, or exported CSV/JSON results.** The repository should contain the public app and its documentation, not operational secrets.

---

## Getting started

### 1. Preview locally

Because the app loads Firebase over HTTPS, open it through a local web server rather than `file://`.

```bash
# Python 3
python -m http.server 8000

# or Node.js
npx serve .
```

Then open `http://localhost:8000/`.

### 2. Configure Firebase

The app expects a Firebase project with:

- **Authentication → Sign-in method → Email/Password** enabled.
- **Cloud Firestore** created.

Replace the `FIREBASE_CONFIG` object near the top of the inline script in `index.html` with your own project's web configuration.

### 3. Create the admin account

Create one Firebase Authentication user for the administrator, then update `adminUid()` in `firestore.rules` to that user's UID. The rules also check the admin email; keep both values in sync.

The admin account is **not** created by the app, and no credentials are included in this repository.

### 4. Publish the security rules

Until the rules are published, Firestore returns `403 PERMISSION_DENIED`.

1. Open **Firebase Console → Firestore Database → Rules**.
2. Paste the entire contents of `firestore.rules`.
3. Click **Publish**.
4. Reload the app.

> **Project-wide impact:** Firestore rules apply to every app in the same Firebase project. If the project hosts other quizzes or services, review and merge the rules deliberately instead of blindly replacing existing rules. `firestore.rules` contains a catch-all deny at the bottom and a clearly marked compatibility block for a legacy quiz; adjust both to your deployment.

### 5. Deploy to GitHub Pages

1. Create a repository and upload **`index.html`** (and this `README.md` if you want repository documentation).
2. Open **Settings → Pages**.
3. Choose **Deploy from a branch**, select `main` / `master`, folder `/ (root)`.
4. Save and wait for the published URL.
5. Open the site and test registration and login.

Do **not** upload `SETUP.md`, `test/`, `learning material/`, or exported student data. The `validation/` folder is for local testing and is not needed in production.

A typical URL looks like:

```text
https://<username>.github.io/<repository>/
```

---

## Exam workflow

1. Participants register with their NIM and password and accept the Terms.
2. They complete the practice sets. Each practice set can be submitted once.
3. The admin configures the real test:
   - timer: 45 minutes (default)
   - pass mark: 60% (default)
   - optional open/close schedule
   - question/option shuffling
   - real-test switch: **Open**
4. Participants take the real test once. The timer auto-submits at zero.
5. The admin monitors submissions and integrity checks.
6. The admin releases the real-test answers when appropriate.
7. The admin exports results if needed.

The real test is locked by default until the admin opens it.

---

## Data model

All collections are prefixed with `fundai_` to avoid collisions in a shared Firebase project.

| Collection | Document ID | Purpose |
| --- | --- | --- |
| `fundai_settings` | `exam` | Exam switches, schedule, timer, pass mark, answer release |
| `fundai_login_index` | `{NIM}` | Maps a NIM to its Firebase Auth account before login |
| `fundai_participants` | `{AuthUID}` | Participant profile and status |
| `fundai_drafts` | `{NIM}_{setId}` | In-progress attempt |
| `fundai_submissions` | `real_{NIM}` / `practice_{setId}_{NIM}` | Immutable submitted result |
| `fundai_audit_logs` | auto ID | Append-only activity log |

`optOrder` is stored as a **flat array of 120 numbers** (30 questions × 4 options), because Firestore rejects nested arrays. The app packs and unpacks this value internally and still reads older local drafts that used the nested form.

---

## Validation

The repository includes a local Firestore rules regression suite in `validation/`.

```bash
cd validation
npm install

# In another terminal:
npx firebase emulators:start --only firestore --project demo-fundai

# Then:
npm test
```

Expected result:

```text
SUMMARY 67/67 rules checks passed
```

The suite covers anonymous access, admin identity, profile updates, atomic registration, closed registration, disabled accounts, drafts, one-attempt submissions, the real-test schedule, result isolation, audit immutability, password-reset profile creation, legacy compatibility, and the catch-all deny.

The delivered build was also checked with:

- a **22/22** browser smoke test;
- an **18/18** participant end-to-end test;
- a **46/46** admin and real-test end-to-end test.

All end-to-end tests ran against the Firebase emulator with the exact delivered rules file.

---

## Security model and limitations

### Protections

- Passwords are hashed by Firebase Authentication; the app never stores plaintext passwords.
- Admin access is enforced by Firestore Rules using UID and email, not by a client-side flag.
- Participants can access only their own data.
- One real-test attempt and each practice attempt are enforced server-side.
- The real-test schedule is checked against Google server time.
- Submitted results are create-only for participants and recomputed by the admin dashboard.
- Audit events are append-only and admin-readable.

### Limitations

1. **The answer key is embedded in the static HTML.** A determined user can inspect the source. The “answers locked” interface is a workflow control, not a cryptographic boundary.
2. **The timer is client-side.** The rules enforce the schedule window and one attempt, but not the exact 45-minute duration. A user who edits their own draft can reset the countdown. Use server-side timing for high-stakes exams.
3. **GitHub Pages is public.** Anyone with the URL can load the app. Use access-controlled hosting if the exam must be private.
4. **No proctoring.** The app records tab switches and timings but cannot detect a second device, screen sharing, or outside help.
5. **Password resets leave an orphaned Auth account.** The old account is unlinked from the NIM and its profile is deleted, but the Firebase Auth user remains. Remove it manually in the Firebase Console if desired.
6. **CSV/JSON exports contain personal data.** Store and delete them according to your institution's policy.
7. **The app is intended for course use, not as a high-stakes certification platform.** For stronger guarantees, use server-side grading and a proctored LMS.

---

## Privacy

The app stores participant names, NIMs, scores, durations, and audit events in Cloud Firestore. Only the admin can list all records; participants can read only their own. Inform participants through the Terms & Conditions and follow your institution's data-protection rules.

Do not commit real participant data, exports, or Firebase service-account credentials to this repository.

---

## Browser support

Tested with current versions of Google Chrome and Microsoft Edge. The app uses standard browser APIs and should work in current Firefox and Safari, but those were not part of the final test run.

JavaScript must be enabled. An internet connection is required for Firebase Authentication and Firestore.

---

## Contributing

This project is maintained for the FUNdamental AI course. Before changing the question bank:

1. Update the CSV sources in `test/`.
2. Regenerate the embedded bank in `index.html`.
3. Update the bank version and SHA-256 fingerprint.
4. Re-run the rules suite and a browser smoke test.
5. Verify all four sets still contain 30 unique questions.

For security-rule changes, update `validation/rules-test.js` and keep the suite passing.

---

## License and attribution

This project was built for internal course use. No open-source license is declared; add a `LICENSE` file if you intend to redistribute it publicly.

Questions are derived from the FUNdamental AI course materials. Firebase is a trademark of Google LLC. This project is not affiliated with or endorsed by Google.

---

## Ringkasan (Bahasa Indonesia)

Aplikasi kuis **FUNdamental AI** berbasis satu file HTML dengan Firebase. Peserta mendaftar menggunakan NIM dan password, mengerjakan tiga set latihan, lalu satu tes asli selama 45 menit dengan batas kelulusan 60%. Admin dapat memantau aktivitas, mengatur jadwal tes, merilis jawaban, dan mereset password peserta.

Aturan Firestore **wajib dipublikasikan secara manual** melalui Firebase Console. Jangan pernah mengunggah `SETUP.md`, data peserta, atau file ekspor ke repositori publik. Karena ini aplikasi statis, kunci jawaban tertanam di `index.html` dan timer berjalan di sisi klien; gunakan penilaian sisi server untuk ujian berskala tinggi.
