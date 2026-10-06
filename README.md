# multiplication4fun

## Configure online accounts and leaderboard

The app uses Firebase Authentication (email/password) and the Firebase Realtime
Database. To enable sign-up, login, and the shared leaderboard:

1. In your Firebase project, enable **Authentication → Email/Password** and
   create a **Realtime Database**.
2. Copy the web app config from Firebase project settings. In `index.html`,
   replace each `PASTE_FIREBASE_...` value in `firebaseConfig` with the
   corresponding value from your Firebase web app, including the API key,
   auth domain, database URL, project ID, storage bucket, sender ID, and app ID.
3. Set these Realtime Database rules:

   ```json
   {
     "rules": {
       "leaderboard": {
         ".read": true,
         "$uid": {
           ".write": "auth != null && auth.uid === $uid",
           ".validate": "newData.hasChildren(['name', 'score'])",
           "name": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 18"
           },
           "score": {
             ".validate": "newData.isNumber() && newData.val() >= 0"
           },
           "$other": {
             ".validate": false
           }
         }
       },
       "profiles": {
         "$uid": {
           ".read": "auth != null && auth.uid === $uid",
           ".write": "auth != null && auth.uid === $uid && (!data.exists() || data.child('role').val() === newData.child('role').val())",
           ".validate": "newData.hasChildren(['firstName', 'lastName', 'username', 'nickname', 'role'])",
           "firstName": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 40"
           },
           "lastName": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 40"
           },
           "username": {
             ".validate": "newData.isString() && newData.val().matches(/^[A-Za-z0-9_]{3,24}$/)"
           },
           "nickname": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 18"
           },
           "role": {
             ".validate": "newData.val() === 'teacher' || newData.val() === 'student'"
           },
           "$other": {
             ".validate": false
           }
         }
       },
       "classes": {
         "$code": {
           ".read": "auth != null",
           ".write": "auth != null && !data.exists() && auth.uid === newData.child('teacherUid').val() && root.child('profiles').child(auth.uid).child('role').val() === 'teacher'",
           ".validate": "newData.hasChildren(['teacherUid', 'className', 'subject', 'teacherName', 'createdAt'])",
           "teacherUid": {
             ".validate": "newData.isString() && newData.val() === auth.uid"
           },
           "className": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 60"
           },
           "subject": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 60"
           },
           "teacherName": {
             ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 100"
           },
           "createdAt": {
             ".validate": "newData.isNumber()"
           },
           "members": {
             "$studentUid": {
               ".write": "auth != null && auth.uid === $studentUid && root.child('profiles').child(auth.uid).child('role').val() === 'student'",
               ".validate": "newData.hasChildren(['name', 'joinedAt'])",
               "name": {
                 ".validate": "newData.isString() && newData.val().length > 0 && newData.val().length <= 18"
               },
               "joinedAt": {
                 ".validate": "newData.isNumber()"
               },
               "$other": {
                 ".validate": false
               }
             }
           },
           "$other": {
             ".validate": false
           }
         }
       },
       "classLinks": {
         "$uid": {
           ".read": "auth != null && auth.uid === $uid",
           ".write": "auth != null && auth.uid === $uid",
           "$code": {
             ".validate": "$code.matches(/^[0-9]{6}$/) && newData.val() === true"
           }
         }
       }
     }
   }
   ```

Each Firebase account has one database entry, keyed by its user ID, so saving a
new score updates that player's existing entry rather than adding another
name. Sign-up asks for first and last name, email, username, nickname, password,
and whether the player is a student or teacher. Firebase rejects duplicate
email addresses; the app directs that person to log in instead. Profile data is
saved privately to the player's own database profile, and their saved sign-in
session is remembered on that device without storing the password. Keep the
provided rules deployed so profile data is private and users can only write
their own profile and leaderboard entry. Teachers can create classes under
six-digit room codes, students can join by code, and each signed-in user can
only read and update their own class links. Class records are readable to
signed-in users who know a room code; only the teacher can create a class and
only students can add their own membership.

The leaderboard is public and refreshes immediately when opened and then every
30 seconds. Game progress, coins, unlocked levels, and an in-progress question
are saved in this browser so closing and reopening the tab does not erase them.
Firebase web config values are intended to be used by the client; scores are
client-submitted and should not be treated as tamper-proof.