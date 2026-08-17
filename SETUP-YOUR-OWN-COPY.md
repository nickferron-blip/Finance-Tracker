# Set Up Your Own Finance Tracker

This gives you your own private copy of the app — your own web address, your own Google sign-in, and your own data (stored in *your* Google Drive). Nobody else can see it.

**Time:** ~30 minutes. You need a **GitHub account** (free) and a **Google account**.

---

## Part 1 — Put the app online (GitHub Pages)

1. Go to **github.com** and sign in (create a free account if needed).
2. Get the app's files. Ask the person who shared this with you for the repository link, then on that repo click **Fork** (top-right) to copy it into your own GitHub account.
   - *Or*, if they gave you a ZIP of the files: create a **new repository** (the green **New** button), name it e.g. `finance-tracker`, set it to **Public**, and drag the files (`index.html`, `manifest.json`, `sw.js`, `logo.png`, the `icons` folder) into it → **Commit changes**.
3. In your repo, go to **Settings** (top menu) → **Pages** (left menu).
4. Under **Build and deployment → Branch**, choose **main** and folder **/ (root)** → **Save**.
5. Wait ~1–2 minutes, refresh the page. It will show your live URL, like
   `https://YOURNAME.github.io/finance-tracker/`
   **Copy this URL — you'll need it in Part 2.**

---

## Part 2 — Create your Google sign-in (Google Cloud, free)

This is what lets the app save your data to *your* Google Drive.

1. Go to **console.cloud.google.com** and sign in with your Google account.
2. Top-left, click the project dropdown → **New Project**. Name it `Finance Tracker` → **Create**. Make sure it's selected.
3. In the top search bar, type **"Google Drive API"**, open it, and click **Enable**.
4. Left menu → **APIs & Services → OAuth consent screen**.
   - User type: **External** → **Create**.
   - App name: `Finance Tracker`. User support email: your email. Developer contact: your email. → **Save and Continue**.
   - **Scopes:** click **Add or Remove Scopes**, search for `drive`, check the box for **`.../auth/drive`** ("See, edit, create, and delete all of your Google Drive files") → **Update** → **Save and Continue**.
   - **Test users:** click **+ Add Users**, enter **your own Google email** (lowercase) → **Add** → **Save and Continue**.
5. Left menu → **APIs & Services → Credentials** → **+ Create Credentials → OAuth client ID**.
   - Application type: **Web application**.
   - Name: `Finance Tracker Web`.
   - Under **Authorized JavaScript origins**, click **+ Add URI** and paste the base of your URL **without any path** — e.g. `https://YOURNAME.github.io`
   - **Create.**
6. A box pops up with your **Client ID** (looks like `1234...apps.googleusercontent.com`). **Copy it.**

---

## Part 3 — Connect the two

1. In your GitHub repo, open **`index.html`** (click the file, then the pencil ✏️ to edit).
2. Press Ctrl/Cmd-F and search for `const CLIENT_ID`. You'll find a line like:
   ```
   const CLIENT_ID = '……apps.googleusercontent.com';
   ```
3. Replace the value between the quotes with **your** Client ID from Part 2, step 6.
4. Scroll down and **Commit changes**.
5. Wait ~1–2 minutes for GitHub Pages to update.

---

## Part 4 — Open it and sign in

1. Go to your URL (`https://YOURNAME.github.io/finance-tracker/`).
2. Click **Sign in with Google**, choose your account.
3. You'll see an **"unverified app"** screen (normal — it's your own app). Click **Advanced → Go to Finance Tracker (unsafe)**, then **Allow** the Google Drive permission.
4. You're in. Your data now saves automatically to a file called `finance-tracker-data.json` in your Google Drive.

### Install it like an app (optional)
- **iPhone/iPad:** open the URL in Safari → Share button → **Add to Home Screen**.
- **Mac/Windows (Chrome):** open the URL → the install icon in the address bar → **Install**.

---

## Notes & troubleshooting

- **"Access blocked / app not verified":** make sure your email is added under **Test users** (Part 2, step 4) and that you clicked **Advanced → continue** on the warning. It's safe — it's your own project.
- **Sign-in does nothing / error:** double-check the **Authorized JavaScript origin** exactly matches your URL's origin (e.g. `https://YOURNAME.github.io`, no trailing slash, no path).
- **Your data is private:** it lives only in your Google Drive. The app has no server and no shared database — nobody else can see it unless *you* share that Drive file.
- **To update the app later:** re-copy the latest `index.html` into your repo and commit. Your data (in Drive) is untouched by updates.
