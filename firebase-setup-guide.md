# Firebase Auth + Firestore — Setup Guide

## 1. Create a Firebase project

1. Go to **console.firebase.google.com** → Add project
2. Name it (e.g. `fittrack-prod`)
3. Disable Google Analytics (optional)

---

## 2. Enable Authentication

Firebase Console → **Authentication** → Get started

Enable the **Email/Password** provider.

---

## 3. Create a Firestore database

Firebase Console → **Firestore Database** → Create database

- Choose **production mode** (we'll set rules next)
- Pick the region closest to your users

---

## 4. Register a Web app

Firebase Console → Project Settings → **Your apps** → Add app → Web

Copy the config object. Create `.env` in your project root:

```
VITE_FIREBASE_API_KEY=AIza...
VITE_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=your-project-id
VITE_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=1234567890
VITE_FIREBASE_APP_ID=1:1234567890:web:abc123
```

Add `.env` to `.gitignore` — never commit credentials.

---

## 5. Install Firebase SDK

```bash
npm install firebase
```

---

## 6. Deploy Firestore Security Rules

In **Firestore → Rules**, paste:

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // ── User documents ─────────────────────────────────────────────────────
    match /users/{uid} {
      // Only the authenticated user can read/write their own document
      allow read, write: if request.auth != null && request.auth.uid == uid;

      // ── Workouts sub-collection ──────────────────────────────────────────
      match /workouts/{workoutId} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }

      // ── Nutrition logs sub-collection ────────────────────────────────────
      match /nutritionLogs/{logId} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }

      // ── Body weight logs ─────────────────────────────────────────────────
      match /bodyWeightLogs/{logId} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }

      // ── Personal records ─────────────────────────────────────────────────
      match /personalRecords/{recordId} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }
    }
  }
}
```

Click **Publish**.

---

## 7. Wire into your React app

```jsx
// main.jsx or App.jsx
import { AuthProvider, useAuth } from "./context/AuthContext";

// AuthContext.jsx
import { createContext, useContext, useState, useEffect } from "react";
import { onAuthChange, subscribeUserProfile } from "./firebase/services";

const AuthContext = createContext(null);
export const useAuth = () => useContext(AuthContext);

export function AuthProvider({ children }) {
  const [user,        setUser]        = useState(undefined); // undefined = loading
  const [userProfile, setUserProfile] = useState(null);

  useEffect(() => {
    return onAuthChange(u => {
      setUser(u ?? null);
    });
  }, []);

  useEffect(() => {
    if (!user) return;
    return subscribeUserProfile(user.uid, setUserProfile);
  }, [user]);

  if (user === undefined) return <SplashScreen />;

  return (
    <AuthContext.Provider value={{ user, userProfile }}>
      {user ? <AppShell /> : <AuthFlow />}
    </AuthContext.Provider>
  );
}
```

---

## 8. Firestore indexes (create as needed)

For the nutrition log query `where("date", "==", dateStr)` no index is needed (single field).

If you add compound queries like `where("date", ">=", start).where("date", "<=", end).orderBy("date")`,
Firebase will prompt you with a direct link to create the index.

---

## 9. Security checklist

- [x] Firestore rules: users can only read/write their own `uid` path
- [x] Passwords never stored — Firebase Auth handles hashing (bcrypt)
- [x] Password reset uses signed, time-limited tokens (1 hr TTL, handled by Firebase)
- [x] Re-authentication required before: password change, email change, account deletion
- [x] All sensitive operations wrapped in try/catch with mapped error messages
- [x] `.env` in `.gitignore`; environment variables used for all credentials
- [x] `deleteAccount` deletes all sub-collections before deleting the auth user
- [ ] Enable **App Check** in production (Firebase Console → App Check) to block non-app traffic
- [ ] Set up **Firebase Alerts** for anomalous auth activity
- [ ] Consider **Email enumeration protection** (Firebase Console → Auth → Settings)

---

## 10. File structure

```
src/
├── firebase/
│   ├── config.js          ← firebaseConfig (reads from .env)
│   └── services.js        ← all auth + Firestore functions
├── context/
│   └── AuthContext.jsx    ← AuthProvider + useAuth hook
├── screens/
│   ├── LoginScreen.jsx
│   ├── SignUpScreen.jsx
│   ├── ResetScreen.jsx
│   └── Dashboard.jsx
└── main.jsx
```

---

## Auth error codes handled

| Code | Message shown to user |
|---|---|
| `auth/email-already-in-use` | An account with this email already exists. |
| `auth/invalid-email` | That doesn't look like a valid email address. |
| `auth/weak-password` | Password must be at least 6 characters. |
| `auth/user-not-found` | No account found with this email. |
| `auth/wrong-password` | Incorrect password. |
| `auth/too-many-requests` | Too many attempts. Try again later or reset your password. |
| `auth/network-request-failed` | Network error. Check your connection and try again. |
| `auth/requires-recent-login` | Please sign in again to complete this action. |
| `auth/invalid-credential` | Incorrect email or password. |
