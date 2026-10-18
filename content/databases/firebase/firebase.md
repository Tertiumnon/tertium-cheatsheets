# Firebase

Google's platform for building applications with real-time database, authentication, hosting, and more.

## Getting Started

```javascript
// Initialize Firebase
import { initializeApp } from 'firebase/app';
import { getFirestore } from 'firebase/firestore';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
  apiKey: 'YOUR_API_KEY',
  authDomain: 'project.firebaseapp.com',
  databaseURL: 'https://project.firebaseio.com',
  projectId: 'project-id'
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);
```

## Firestore Operations

### Reading Data

```javascript
import { getDoc, getDocs, collection, query, where } from 'firebase/firestore';

// Get single document
const docRef = doc(db, 'users', 'userId');
const docSnap = await getDoc(docRef);
if (docSnap.exists()) {
  console.log(docSnap.data());
}

// Get all documents in collection
const querySnapshot = await getDocs(collection(db, 'users'));
querySnapshot.forEach(doc => console.log(doc.data()));

// Query with where clause
const q = query(collection(db, 'users'), where('age', '>', 18));
const snapshot = await getDocs(q);
```

### Writing Data

```javascript
import { setDoc, addDoc, collection, updateDoc, deleteDoc, doc } from 'firebase/firestore';

// Set document
await setDoc(doc(db, 'users', 'userId'), {
  name: 'John',
  email: 'john@example.com'
});

// Add document with auto-generated ID
await addDoc(collection(db, 'users'), {
  name: 'Jane',
  email: 'jane@example.com'
});

// Update specific fields
await updateDoc(doc(db, 'users', 'userId'), {
  age: 30
});

// Delete document
await deleteDoc(doc(db, 'users', 'userId'));
```

### Real-time Listeners

```javascript
import { onSnapshot } from 'firebase/firestore';

// Listen to single document
onSnapshot(doc(db, 'users', 'userId'), (doc) => {
  console.log('Updated:', doc.data());
});

// Listen to collection
onSnapshot(collection(db, 'users'), (snapshot) => {
  snapshot.forEach(doc => console.log(doc.data()));
});
```

## Authentication

```javascript
import { createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut, onAuthStateChanged } from 'firebase/auth';

// Create user
await createUserWithEmailAndPassword(auth, 'user@example.com', 'password123');

// Sign in
await signInWithEmailAndPassword(auth, 'user@example.com', 'password123');

// Sign out
await signOut(auth);

// Listen to auth state
onAuthStateChanged(auth, (user) => {
  if (user) {
    console.log('Logged in:', user.email);
  } else {
    console.log('Logged out');
  }
});
```

## Firestore Rules

### Development (NOT for production)

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```

### Secure Example

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Authenticated users can read/write their own documents
    match /users/{userId} {
      allow read, write: if request.auth.uid == userId;
    }

    // Posts are readable by all, writable only by owner
    match /posts/{postId} {
      allow read: if true;
      allow write: if request.auth.uid == resource.data.owner;
    }
  }
}
```

## Batch Operations

```javascript
import { writeBatch } from 'firebase/firestore';

const batch = writeBatch(db);

batch.set(doc(db, 'users', 'user1'), { name: 'User 1' });
batch.set(doc(db, 'users', 'user2'), { name: 'User 2' });
batch.update(doc(db, 'users', 'user3'), { status: 'active' });

await batch.commit();
```

## Troubleshooting

### "Your Cloud Firestore database has insecure rules"

Edit rules at: `Firebase Console → Cloud Firestore → Rules`

See [Firebase security rules documentation](https://firebase.google.com/docs/rules/basics) for production-ready rules.

## Key Concepts

- **Firestore:** NoSQL database with real-time capabilities
- **Documents:** JSON-like data structures
- **Collections:** Groups of documents
- **Indexes:** Required for complex queries
- **Rules:** Security and validation for database access
