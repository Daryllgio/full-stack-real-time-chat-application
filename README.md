# Real-Time Chat Application 💬

## **Overview**

A real-time chat app built with **React + Vite** and **Firebase** (**Auth**, **Firestore**, **Storage**). It implements secure email/password auth, user profiles (avatar/name/bio), live chat threads, and image uploads—deployed as a single-page app (SPA) with Firebase Hosting rewrites.

---

## 🔗 Live Demo
[![Live on Vercel](https://img.shields.io/badge/Live%20Demo-Vercel-000?logo=vercel)](https://full-stack-real-time-chat-web-application.vercel.app)

## 👀 Quick Preview
<p align="center">
  <a href="https://full-stack-real-time-chat-web-application.vercel.app">
    <img width="900" alt="Real-Time Chat – conversation view with sidebar and media gallery" src="https://github.com/user-attachments/assets/684fc82e-00a9-4731-8907-e1c6a70ec653" />
  </a>
</p>

---

## **Key Features**

* **Auth (Email/Password):** `signup`, `login`, `logout`, `resetPass` (see `src/config/firebase.js`).
* **Profiles:** `users/{uid}` docs store `username`, `name`, `avatar`, `bio`, `lastSeen`.
* **Real-time chat list:** Per-user `chats/{uid}` doc with a `chatsData` array; UI listens with `onSnapshot` and also refreshes on a timer.
* **Image uploads:** Client uploads to **Storage** under `images/<timestamp+filename>` via `src/lib/upload.js`.
* **Responsive SPA:** Routed pages for login, profile setup/update, and chat; left/right sidebars + chat box.

---

## **Tech Stack**

* **Frontend:** React, Vite, CSS
* **Routing:** React Router
* **State:** React Context (`src/context/AppContext.jsx`)
* **Cloud:** Firebase Auth, Firestore, Storage
* **Deploy:** Firebase Hosting (SPA rewrites)

---

## **Project Structure**

```txt
.
├─ public/
│  ├─ background.png
│  └─ vite.svg
├─ src/
│  ├─ components/
│  │  ├─ LeftSidebar/...
│  │  ├─ ChatBox/...
│  │  └─ RightSidebar/...
│  ├─ config/
│  │  └─ firebase.js          # init + auth/db functions (signup/login/etc.)
│  ├─ context/
│  │  └─ AppContext.jsx       # global user/chat state, listeners
│  ├─ lib/
│  │  └─ upload.js            # Storage upload helper (images/)
│  ├─ pages/
│  │  ├─ Chat/
│  │  │  ├─ Chat.css
│  │  │  └─ Chat.jsx
│  │  ├─ Login/
│  │  │  ├─ Login.css
│  │  │  └─ Login.jsx
│  │  └─ ProfileUpdate/
│  │     ├─ ProfileUpdate.css
│  │     └─ ProfileUpdate.jsx
│  ├─ App.jsx
│  ├─ index.css
│  └─ main.jsx
├─ firebase.json              # SPA rewrites → /index.html
├─ .firebaserc                # default project alias
└─ (…package.json, index.html, etc.)
```

---

## **Data Model (what your code actually uses)**

### `users` (collection)

Created on **signup** in `src/config/firebase.js`:

```json
{
  "id": "<uid>",
  "username": "<lowercase>",
  "email": "<email>",
  "name": "",
  "avatar": "",
  "bio": "Hey, There i am using chat app",
  "lastSeen": <epoch_ms>
}
```

* **Updated**: `lastSeen` is updated in `AppContext.jsx` on login and every 60s.

### `chats` (collection)

One **document per user**: `chats/{uid}`
Created on **signup** with:

```json
{ "chatsData": [] }
```

* **Observed fields in use:** each entry in `chatsData` includes **at least**:

  * `rId` — the other user’s UID (fetched via `doc(db,"users", rId)` for sidebar info)
  * `updatedAt` — number (used for sorting newest first)

> The **messages storage** is handled inside the Chat UI (e.g., `components/ChatBox/*`). Your context maintains `messagesId` and `messages` state, so the ChatBox likely resolves `messagesId` for the active thread and reads/writes messages there. See `src/components/ChatBox/ChatBox.jsx` for the exact message path.

### Storage (images)

Uploads go to:

```
images/<timestamp+originalName>
```

(see `src/lib/upload.js` using `uploadBytesResumable` → `getDownloadURL`)

---

## **How State & Real-Time Listeners Work**

* **Login/Profile routing:** In `AppContext.jsx → loadUserData(uid)`

  * Fetches `users/{uid}` → sets `userData`
  * If `avatar` and `name` exist → `navigate('/chat')`, else `navigate('/profile')`
  * Updates `lastSeen` immediately and on an interval

* **Chat list listener:**

  ```js
  const chatRef = doc(db, 'chats', userData.id);
  onSnapshot(chatRef, async (res) => {
    const chatItems = res.data().chatsData;
    // look up each rId in users/{rId}, attach userData, sort by updatedAt desc
  });
  ```

  There’s also a **10s polling** fallback that repeats this fetch.

> You’re using both **`onSnapshot`** and **polling**. Snapshot alone usually suffices; keep polling only if you need defensive refreshes for edge cases.

---

## **Auth & Backend Functions (in your repo)**

All in `src/config/firebase.js`:

* **`signup(username, email, password)`**

  * Ensures `username` is unique (`where("username","==",username.toLowerCase())`)
  * Creates user with `createUserWithEmailAndPassword`
  * Writes `users/{uid}` (fields above)
  * Creates `chats/{uid}` with `{ chatsData: [] }`
* **`login(email, password)`** → `signInWithEmailAndPassword`
* **`logout()`** → `signOut`
* **`resetPass(email)`** → Validates that email exists in `users` then calls `sendPasswordResetEmail`

---

## **Setup & Run**

### 1) Install

```bash
npm install
```

### 2) Configure Firebase

* You currently **hardcode** config in `src/config/firebase.js`.
  For public repos, move to `.env.local` and read via `import.meta.env`:

  ```bash
  VITE_FIREBASE_API_KEY=...
  VITE_FIREBASE_AUTH_DOMAIN=...
  VITE_FIREBASE_PROJECT_ID=...
  VITE_FIREBASE_STORAGE_BUCKET=...
  VITE_FIREBASE_MESSAGING_SENDER_ID=...
  VITE_FIREBASE_APP_ID=...
  ```

  Then in `firebase.js`:

  ```js
  const firebaseConfig = {
    apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
    authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
    projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
    storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
    messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
    appId: import.meta.env.VITE_FIREBASE_APP_ID,
  };
  ```

  *Do not commit `.env.local`.*

### 3) Run Dev Server

```bash
npm run dev
```

---

## **Deployment (Firebase Hosting)**

Your `firebase.json` already configures SPA rewrites:

```json
{
  "hosting": {
    "public": "dist",
    "rewrites": [{ "source": "**", "destination": "/index.html" }]
  }
}
```

Build and deploy:

```bash
npm run build
firebase deploy
```

---

## **Security Rules (recommended to add)**

You’re managing rules in the console (no `firestore.rules`/`storage.rules` files in repo). Add these **starter** rules locally and deploy, tuned to **your actual paths**:

**Firestore** (restrict chat access to owners; profiles editable by their owners)

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    match /users/{userId} {
      allow read: if request.auth != null;
      allow create, update, delete: if request.auth != null && request.auth.uid == userId;
    }

    // Per-user chat doc: chats/{uid}
    match /chats/{ownerId} {
      allow read, write: if request.auth != null && request.auth.uid == ownerId;
    }
  }
}
```

**Storage** (your uploads go under `images/…`)

```js
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    // Allow only signed-in users; tighten further if you add per-chat paths later
    match /images/{allPaths=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

> If you later store images per chat (e.g., `chat_images/{chatId}/...`), change the rule to check membership against Firestore.

Deploy local rule files:

```bash
firebase deploy --only firestore:rules
firebase deploy --only storage
```

---

## **Screenshots**

<img width="1440" alt="Screenshot 2025-01-16 at 15 28 24" src="https://github.com/user-attachments/assets/684fc82e-00a9-4731-8907-e1c6a70ec653" />
<img width="1440" alt="Screenshot 2025-01-16 at 13 50 56" src="https://github.com/user-attachments/assets/30bb5761-fc7c-46b7-b42b-baa6ee5cfe69" />
<img width="1440" alt="Screenshot 2025-01-16 at 15 10 05" src="https://github.com/user-attachments/assets/36a56608-1a94-40be-9c20-e9f75436ee06" />
<img width="1440" alt="Screenshot 2025-01-16 at 02 57 01" src="https://github.com/user-attachments/assets/b28e6663-e102-4b05-8d70-b8a69ae7d9ef" />

---

## **Known Issues / TODO (from your code)**

* In `AppContext.jsx`, the `setInterval` uses `auth.chatUser`; that property doesn’t exist.
  Use `auth.currentUser`:

  ```js
  if (auth.currentUser) { await updateDoc(userRef, { lastSeen: Date.now() }); }
  ```
* You have both `onSnapshot` **and** a 10s polling fetch of chats. Consider keeping only `onSnapshot` unless you’ve seen missed updates.
* Consider moving Firebase config to `.env.local` to avoid exposing keys in the repo.

---

If you want, send me `src/components/ChatBox/ChatBox.jsx` and I’ll add the **exact** message-storage path and update the README’s data model section accordingly.
