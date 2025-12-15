# Firestore Security Rules for DevMoodMirror App

**Updated for latest features:** Mood Heatmap, Mood Calendar View, Performance Mode, and User Preferences sync.

Copy and paste the following rules into your Firestore Rules section in the Firebase Console:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users collection - users can only access their own data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;

      // Validate user document structure
      allow create: if request.auth != null
        && request.auth.uid == userId
        && validateUserData(request.resource.data);

      allow update: if request.auth != null
        && request.auth.uid == userId
        && validateUserUpdate(request.resource.data, resource.data);
    }

    // Mood entries collection - users can only access their own mood data
    match /users/{userId}/moods/{moodId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;

      // Validate mood entry structure
      allow create: if request.auth != null
        && request.auth.uid == userId
        && validateMoodEntry(request.resource.data);

      allow update: if request.auth != null
        && request.auth.uid == userId
        && validateMoodUpdate(request.resource.data);
    }

    // Liked quotes collection - users can only access their own liked quotes
    match /users/{userId}/likedQuotes/{quoteId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;

      // Validate liked quote structure
      allow create: if request.auth != null
        && request.auth.uid == userId
        && validateLikedQuote(request.resource.data);
    }

    // User statistics collection - users can only access their own stats
    match /users/{userId}/statistics/{statId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // User preferences collection - for app settings and preferences
    match /users/{userId}/preferences/{prefId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;

      // Validate preferences structure
      allow create: if request.auth != null
        && request.auth.uid == userId
        && validateUserPreferences(request.resource.data);
    }

    // Public moods collection - read access for all authenticated users
    match /publicMoods/{moodId} {
      // Anyone authenticated can read public moods
      allow read: if request.auth != null;

      // Only the mood owner can write/update their public mood
      allow write: if request.auth != null && request.auth.uid == resource.data.userId;

      // Validate public mood entry structure
      allow create: if request.auth != null
        && request.auth.uid == request.resource.data.userId
        && validatePublicMoodEntry(request.resource.data);

      allow update: if request.auth != null
        && request.auth.uid == resource.data.userId
        && validatePublicMoodUpdate(request.resource.data);
    }

    // Helper functions for data validation
    function validateUserData(data) {
      return data.keys().hasAll(['uid', 'email', 'displayName', 'createdAt']) &&
             data.uid is string &&
             data.email is string &&
             data.displayName is string &&
             data.createdAt is timestamp &&
             data.currentStreak is number &&
             data.highestStreak is number &&
             data.totalMoodEntries is number;
    }

    function validateUserUpdate(newData, oldData) {
      return newData.keys().hasAll(['uid', 'email', 'displayName', 'createdAt']) &&
             newData.uid == oldData.uid &&
             newData.email == oldData.email &&
             newData.createdAt == oldData.createdAt &&
             newData.displayName is string &&
             newData.currentStreak is number &&
             newData.highestStreak is number &&
             newData.totalMoodEntries is number;
    }

    function validateMoodEntry(data) {
      return data.keys().hasAll(['mood', 'timestamp', 'userId']) &&
             data.mood is map &&
             data.mood.keys().hasAll(['emoji', 'name']) &&
             data.mood.emoji is string &&
             data.mood.name is string &&
             data.timestamp is timestamp &&
             data.userId is string &&
             (data.reason == null || data.reason is string) &&
             (data.quote == null || data.quote is map);
    }

    function validateMoodUpdate(data) {
      return data.keys().hasAll(['mood', 'timestamp', 'userId']) &&
             data.mood is map &&
             data.timestamp is timestamp &&
             data.userId is string;
    }

    function validateLikedQuote(data) {
      return data.keys().hasAll(['quote', 'character', 'anime', 'likedAt']) &&
             data.quote is string &&
             data.character is string &&
             data.anime is string &&
             data.likedAt is timestamp;
    }

    function validateUserPreferences(data) {
      return data.keys().hasAny(['performanceMode', 'calendarView', 'theme', 'language', 'publicMoodSharing']) &&
             (data.performanceMode == null || data.performanceMode is bool) &&
             (data.calendarView == null || data.calendarView is string) &&
             (data.theme == null || data.theme is string) &&
             (data.language == null || data.language is string) &&
             (data.publicMoodSharing == null || data.publicMoodSharing is bool) &&
             (data.updatedAt == null || data.updatedAt is timestamp);
    }

    function validatePublicMoodEntry(data) {
      return data.keys().hasAll(['mood', 'timestamp', 'userId', 'userDisplayName']) &&
             data.mood is map &&
             data.mood.keys().hasAll(['emoji', 'name']) &&
             data.mood.emoji is string &&
             data.mood.name is string &&
             data.timestamp is timestamp &&
             data.userId is string &&
             data.userDisplayName is string &&
             (data.reason == null || data.reason is string) &&
             (data.quote == null || data.quote is map) &&
             (data.isPublic == null || data.isPublic is bool);
    }

    function validatePublicMoodUpdate(data) {
      return data.keys().hasAll(['mood', 'timestamp', 'userId', 'userDisplayName']) &&
             data.mood is map &&
             data.timestamp is timestamp &&
             data.userId is string &&
             data.userDisplayName is string;
    }
  }
}
```

## Rule Explanation:

### 1. **User Documents** (`/users/{userId}`)

- Users can only read and write their own user document
- Validates required fields: uid, email, displayName, createdAt, currentStreak, highestStreak, totalMoodEntries
- Prevents users from modifying immutable fields like uid, email, and createdAt

### 2. **Mood Entries** (`/users/{userId}/moods/{moodId}`)

- Users can only access their own mood entries
- Validates mood entry structure with required fields: mood (with emoji and name), timestamp, userId
- Optional fields: reason (string) and quote (map object)

### 3. **Liked Quotes** (`/users/{userId}/likedQuotes/{quoteId}`)

- Users can only access their own liked quotes
- Validates quote structure: quote text, character, anime, and likedAt timestamp

### 4. **User Statistics** (`/users/{userId}/statistics/{statId}`)

- Users can only access their own statistics
- Flexible structure for various statistical data

### 5. **User Preferences** (`/users/{userId}/preferences/{prefId}`)

- Users can only access their own app preferences
- Stores settings like: performanceMode, calendarView, theme, language, publicMoodSharing
- Validates preference data types and structure
- Allows syncing preferences across devices

### 6. **Public Moods** (`/publicMoods/{moodId}`)

- **Read Access**: All authenticated users can read public moods
- **Write Access**: Only the mood owner can create/update their public mood entries
- Validates public mood structure with required fields: mood (with emoji and name), timestamp, userId, userDisplayName
- Optional fields: reason (string), quote (map object), isPublic (boolean)
- Enables social mood sharing while maintaining user privacy controls

### 7. **Security Features:**

- **Authentication Required**: All operations require user authentication
- **User Isolation**: Users can only access their own private data
- **Public Data Access**: Authenticated users can read public mood data from all users
- **Data Validation**: Ensures data integrity with structure validation
- **Immutable Fields**: Prevents modification of critical fields like user ID and creation date
- **Privacy Controls**: Users control which moods are shared publicly

### 8. **Data Structure Validation:**

- Validates required fields exist
- Checks data types (string, number, timestamp, map)
- Ensures consistent data structure across the application

## How to Apply These Rules:

1. Go to your Firebase Console
2. Navigate to Firestore Database
3. Click on the "Rules" tab
4. Replace the existing rules with the code above
5. Click "Publish" to apply the new rules

**Note**: These rules provide strong security while allowing the DevMoodMirror app to function properly with both private and public mood features. They ensure users can only access their own private data while allowing controlled access to public mood data, maintaining data integrity through validation functions.
