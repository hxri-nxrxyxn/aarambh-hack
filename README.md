# HELPOSZ - Emergency Assistance & Monitoring System
### Aarambh 3.0 Hackathon Submission

> **DISCLAIMER: HACKATHON PROTOTYPE**
> This application is a prototype developed for the Aarambh 3.0 Hackathon. It is **NOT** certified for real-world medical, legal, or emergency use. It relies on consumer-grade hardware sensors and assumes human oversight.
> *Compliance Reference: Rule 6 (Safety, Responsibility & Disclaimers)*

---

## 1. Project Overview & Problem Statement
**Helposz** is a mobile-first emergency response and monitoring application designed to assist users in critical situations. It leverages native device capabilities (sensors, camera, volume buttons) to provide low-friction trigger mechanisms for distress signals and fall detection.

**Problem Addressed:** 
In high-stress emergencies, unlocking a phone and dialing numbers is often too slow or impossible. Helposz solves this by using "always-available" hardware triggers (like volume buttons) and background monitoring (posture detection) to initiate alerts.

## 2. Key Features (System Workflow)
- **Hardware Trigger:** Activates emergency protocols via volume button sequences (bypassing the need to look at the screen).
- **Fall/Posture Detection:** Uses `MediaPipe Pose` (pre-trained model) to analyze camera feed for fall detection or specific distress postures.
- **Location Tracking:** Captures and logs GPS coordinates during active alerts.
- **Dock/Premium Modes:** specialized monitoring modes for stationary or on-the-go scenarios.

## 3. Technical Architecture & Stack
*Compliance Reference: Rule 7 (Technical Guidelines) & Rule 8 (Plagiarism & Honesty)*

This project is built using:
- **Frontend/UI:** SvelteKit (Svelte 5)
- **Native Bridge:** CapacitorJS (Android)
- **AI/ML Model:** Google MediaPipe Pose (Pre-trained) - *Used for posture analysis.*
- **State Management:** Svelte Stores

**Third-Party Libraries & Plugins:**
- `@capacitor/core`, `@capacitor/android`
- `@capacitor/status-bar` (UI immersion)
- `@capacitor/geolocation` (Location services)
- `@capacitor-community/keep-awake` (Prevent sleep during monitoring)
- `@odion-cloud/capacitor-volume-control` (Hardware trigger interface)

## 4. Setup & Installation
*Compliance Reference: Rule 10 (Submission Requirements)*

### Prerequisites
- Node.js (v18+)
- Android Studio (for native build)
- Android Device (recommended for full sensor testing)

### Installation Steps
1. **Clone the repository:**
   ```bash
   git clone <repository_url>
   cd helposz
   ```

2. **Install Dependencies:**
   ```bash
   npm install
   ```

3. **Development (Browser Mode):**
   *Note: Native features like volume triggers will not work in the browser.*
   ```bash
   npm run dev
   ```

4. **Build & Sync for Android:**
   ```bash
   npm run build
   npx cap sync
   ```

5. **Run on Device:**
   ```bash
   npx cap open android
   # Run the app from Android Studio onto your connected device
   ```

## 5. Usage Guide (Demo Instructions)
1. **Launch App:** Grant necessary permissions (Camera, Location).
2. **Select Mode:** Choose "Dock Monitor" or "Premium Monitor" from the home screen.
3. **Trigger:** 
   - *Hardware:* Press Volume buttons as configured.
   - *Visual:* Visual cues will indicate active monitoring state.

## 6. Ethical Data Usage & Privacy
*Compliance Reference: Rule 5 (Data Usage)*
- **Local Processing:** Video feeds for posture detection are processed locally on the device. No video stream is sent to the cloud.
- **No PII:** The app does not collect personally identifiable information beyond temporary location data used strictly for the emergency function.

---

**Developed for Aarambh 3.0**
*Built with integrity and responsible innovation.*