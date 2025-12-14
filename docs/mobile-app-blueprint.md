# ShiftSync: Mobile Application Master Blueprint
**Target Platforms:** iOS (App Store) & Android (Google Play)
**Architecture:** React Native (Expo) + Firebase
**Design System:** NativeWind (Tailwind CSS for Mobile)

---

## 1. Executive Summary
This document serves as the master specification for building ShiftSync as a native mobile application. The core philosophy is **"Speed > Features"**. The app must load instantly, feel native (haptics, smooth transitions), and work offline.

We are migrating from a **React Web Prototype** to **React Native (Expo)**.

---

## 2. Technical Stack & Architecture

### Core Framework
*   **Runtime:** **Expo SDK 52+** (Managed Workflow). This handles native build configuration for us.
*   **Language:** TypeScript.
*   **Navigation:** **Expo Router** (File-based routing, similar to Next.js).
*   **Styling:** **NativeWind v4**. Allows us to reuse 90% of the existing Tailwind classes from the web prototype.

### Native Features
*   **Haptics:** `expo-haptics` (Crucial for the "tactile" feel of swapping shifts).
*   **Blur Effects:** `expo-blur` (For modal backgrounds and glass-morphism).
*   **Gestures:** `react-native-gesture-handler` (For swipe-to-accept or drag-and-drop scheduling).
*   **Safe Areas:** `react-native-safe-area-context`.

### Backend (Firebase)
*   **Auth:** Firebase Auth (Phone Number & Social).
*   **Database:** Cloud Firestore (Offline Persistence enabled).
*   **Functions:** Firebase Cloud Functions (For sending invites/emails).
*   **Deep Linking:** Firebase Dynamic Links (For onboarding invite links).

---

## 3. UI/UX Translation Guide (Web -> Mobile)

To build this from the existing files, follow this translation dictionary:

| Web Component (HTML) | Mobile Component (React Native) | Notes |
| :--- | :--- | :--- |
| `<div className="flex...">` | `<View className="flex...">` | Flexbox defaults to `column` in RN. |
| `<span/h1/p>` | `<Text>` | Text cannot be inside View without a Text wrapper. |
| `<button onClick={...}>` | `<Pressable onPress={...}>` | Add `active:opacity-80` for touch feedback. |
| `overflow-x-auto` (Scroll) | `<FlatList horizontal>` | **Critical:** Do not use `ScrollView` for long lists. |
| `<img>` | `<Image source={...} />` | Requires explicit dimensions. |
| `linear-gradient` (CSS) | `<LinearGradient>` | From `expo-linear-gradient`. |
| `fixed bottom-0` | `absolute bottom-0` | Check `useSafeAreaInsets` for padding. |

---

## 4. Navigation Architecture (App Flow)

Using **Expo Router**, the file structure determines navigation.

```text
app/
├── _layout.tsx           <-- Root Provider (Auth Context, Query Client)
├── (auth)/               <-- Login Stack
│   ├── login.tsx
│   └── join.tsx          <-- Deep link landing page for invites
├── (tabs)/               <-- Main Tab Navigator
│   ├── _layout.tsx
│   ├── index.tsx         <-- "Schedule" (Home)
│   ├── swaps.tsx         <-- "Swap Marketplace"
│   └── manager/          <-- "Manager Dashboard"
│       ├── _layout.tsx
│       ├── schedule.tsx
│       └── team.tsx      <-- NEW: Team Roster View
└── modals/               <-- Full screen modals
    ├── sync-calendar.tsx
    └── add-member.tsx    <-- NEW: Manager Invite Flow
```

---

## 5. Component Implementation Details

### A. NextShiftHero (The "Big Card")
*   **Visuals:** Use `<LinearGradient>` for the background (Emerald for "Soon", Blue for "Future").
*   **Animations:** Use `react-native-reanimated`. When the countdown changes, numbers should slide slightly.
*   **Maps:** The "Location" link should use `Linking.openURL('geo:...')` to open Apple Maps/Google Maps natively.

### B. WeeklyTimeline (The Date Picker)
*   **Implementation:** `FlatList` (Horizontal).
*   **Logic:**
    *   `initialScrollIndex`: Auto-scroll to today.
    *   `getItemLayout`: Required for `scrollToIndex` to work accurately.
    *   **Interaction:** Tapping a date triggers `Haptics.selectionAsync()`.

### C. Schedule Carousel (The Main List)
*   **Implementation:** `FlatList` with `pagingEnabled={true}`.
*   **Performance:** This replaces the CSS snap-scroll. It ensures the user lands exactly on a shift card.
*   **Sync:** This list must synchronize with the `WeeklyTimeline`. When `WeeklyTimeline` changes, `Carousel` ref calls `scrollToIndex`.

### D. Manager View & Team Roster
*   **Tab Controller:** Use a custom Segmented Control (Animation between "Schedule" and "Team").
*   **Team List:** `SectionList` is best here.
    *   **Section 1:** "Pending Invites" (Grayed out avatars).
    *   **Section 2:** "Active Staff".
*   **Add Member Modal:**
    *   Use a `KeyboardAvoidingView` to ensure inputs aren't hidden by the keyboard.
    *   **Tag Input:** Implementing a chip input in Native requires managing a `flex-wrap` View of Pressables. Tapping "Enter" on the keyboard should trigger tag creation.

---

## 6. Data & State Management

### Global Store (Zustand)
We move away from `useState` in App.tsx to a global store to handle data across screens.

```typescript
interface AppStore {
  user: User | null;
  shifts: Shift[];
  isOffline: boolean;
  // Actions
  pickupShift: (id: string) => Promise<void>;
  offerShift: (id: string) => Promise<void>;
  syncShifts: () => Promise<void>;
}
```

### Offline-First Strategy (Firestore)
1.  **Read:** Always read from local Firestore cache first (`source: 'cache'`).
2.  **Write:** Optimistic Updates. Update the UI *immediately* (local state), then send to Firestore. If Firestore fails, rollback and show a Toast.
3.  **Sync:** Background task (using `expo-background-fetch`) to sync schedule changes if the app is in the background.

---

## 7. Manager Onboarding Flow (Detailed)

This is the critical new feature for store growth.

### Step 1: Manager Adds User (Mobile)
1.  Manager taps "+" on Team Tab.
2.  Opens `modals/add-member.tsx`.
3.  **Action:** Manager enters email + selects tags (Server, Bar).
4.  **Native Code:**
    ```typescript
    await firestore().collection('stores').doc(storeId).collection('invites').add({
      email: email,
      tags: selectedTags,
      token: generateSecureToken() // Cloud Function does this usually
    });
    ```

### Step 2: The Invite Link (Deep Linking)
The email sent contains a link: `https://shiftsync.app/join?token=xyz`.
*   **Configuration:** Configure `apple-app-site-association` and `assetlinks.json`.
*   **Behavior:**
    *   If app installed: Opens directly to `(auth)/join` screen.
    *   If not installed: Redirects to App Store.

### Step 3: Employee Onboarding
1.  User lands on `join.tsx`.
2.  App reads `token` from URL parameters.
3.  **API Call:** Verifies token with backend.
4.  **UI:** "Welcome to [Restaurant Name]. Confirm your account."
5.  **Completion:** Creates Firebase Auth account -> Links to Store Document -> Navigates to Home.

---

## 8. Deployment Checklist

### iOS (Apple App Store)
1.  **Icons:** Generate assets using `npx expo-image-utils`.
2.  **Permissions:** Update `app.json` `ios.infoPlist`:
    *   `NSCalendarsUsageDescription`: "ShiftSync needs access to add shifts to your calendar."
3.  **Build:** `eas build --platform ios`.

### Android (Google Play)
1.  **Adaptive Icon:** Ensure background/foreground layers are separated.
2.  **Permissions:** `WRITE_CALENDAR`.
3.  **Build:** `eas build --platform android`.

---

## 9. Code Scaffolding Example (Manager Roster Item)

How a component from the prototype translates to React Native:

**Web (Current):**
```tsx
<div className="flex items-center space-x-3">
  <div className="w-12 h-12 rounded-full bg-black...">AR</div>
  <div>
     <h3>Alex Rivera</h3>
     <div className="flex gap-1">...tags</div>
  </div>
</div>
```

**Mobile (Goal):**
```tsx
import { View, Text, Image } from 'react-native';
import { styled } from 'nativewind';

const StyledView = styled(View);
const StyledText = styled(Text);

export const RosterItem = ({ user }) => (
  <StyledView className="flex-row items-center space-x-3 p-4 bg-white border-b border-gray-100">
    <StyledView className="w-12 h-12 rounded-full bg-black items-center justify-center">
      <StyledText className="text-white font-bold">{user.initials}</StyledText>
    </StyledView>
    <StyledView className="flex-1">
      <StyledText className="font-bold text-lg text-gray-900">{user.name}</StyledText>
      <StyledView className="flex-row flex-wrap gap-1 mt-1">
        {user.tags.map(tag => (
           <TagChip key={tag.id} color={tag.color} label={tag.name} />
        ))}
      </StyledView>
    </StyledView>
  </StyledView>
);
```
