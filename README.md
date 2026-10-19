# Mobile Security project - documentation

## Groupmember

1. Monica Repolho
2. Elene Kapanadze
3. Mariam Asatryan
4. Josh Van Wyk

## Project summary

A Movie Rating Android app that is used to demonstrate mobile security techniques.
It loads movie data from TMDB over HTTPS, shows a list and detail screen, and lets users rate movies, add comments and record voice‑note reviews. It contains security‑related features and weaknesses: a 3‑rating trial that can be bypassed with Frida, an injected malware module that steals the API key, modification of the APK with apktool to change all posters, root detection that hides functionality on rooted devices, and both secure (internal) and insecure (unencrypted Room) storage examples for demonstrating local data protection issues.
## Requirements

| Application                 |
| --------------------------- |
| **Application**             |
| 2 UI screens                |
| Secure API request          |
| API request with IDOR       |
| Connection to room database |
| Secure storage              |

| Security                            |
| ----------------------------------- |
| Unsafe storage                      |
| Malware                             |
| Frida functionality                 |
| Modify Application                  |
| Detect root and block functionality |

## Overview app

### ![](./documentation/ReadmeImages/Screenshot.png) Screenshots

This app has three main screens:

- The `LoginScreen` shows the simple login form that appears when the app starts.
- The `MovieList_Screen1` shows the list of movies with their titles, posters and current ratings.
- The `MovieDetails_Screen2` shows the detail page for a single movie, where the user can set a star rating, add a comment and work with voice reviews.

<img src="./documentation/ReadmeImages/LoginScreen.png" alt="LoginScreen" width="220" />
<img src="./documentation/ReadmeImages/MovieList_Screen1.png" alt="MovieList_Screen1" width="220" />
<img src="./documentation/ReadmeImages/MovieDetails_Screen2.png" alt="MovieDetails_Screen2" width="220" />

### ![](./documentation/ReadmeImages/API.png) Secure API request

The app uses secure HTTPS API calls to retrieve movie data from the TMDB backend. The networking layer is built with Retrofit and configured centrally in NetworkModule, which sets up the TmdbApiService interface. This interface defines the endpoints and ensures that all traffic goes over TLS, so the JSON responses (titles, poster URLs, descriptions, etc.) are protected while they are sent over the network.

There are two separate secure API requests:

- Secure request 1 - movie list
  - Endpoint method: TmdbApiService.getPopularMovies(apiKey: String) (e.g. GET /movie/popular).
  - Called from MovieRepository.getAllMovies() when the app loads the MovieListScreen.
  - The JSON list of movies is mapped into Kotlin models (MovieApiModel → MovieEntity / MovieWithRating) and displayed as the scrollable list of cards on MovieListScreen, showing each movie’s title, poster and short overview.

- Secure request 2 - movie details
  - Endpoint method: TmdbApiService.getMovieDetails(movieId: Int, apiKey: String) (e.g. GET /movie/{id}).
  - Called from MovieRepository.fetchMovieDetailsFromApi(movieId) when the user opens a movie in MovieDetailScreen or when details are refreshed.
  - The JSON object for that movie is mapped into Kotlin data classes and passed to the UI to build the full movie card on MovieDetailScreen, including the large poster image, title and complete description below the rating, comment and voice‑review sections.

### ![](./documentation/ReadmeImages/API.png) API request with IDOR


In the security-testing setup, the MovieRatingApp communicates with a local backend running on the host machine. The Android emulator accesses this backend through the special loopback address `10.0.2.2`, with traffic redirected from the host using:

```
adb forward tcp:45678 tcp:45678
```

A Wi-Fi proxy is configured inside the emulator to forward all HTTPS traffic through BurpSuite (`10.0.2.2:8080`). Burp’s CA certificate is installed on the device to allow TLS interception, and a hostname override is placed in Burp (mapping 10.0.2.2 → 127.0.0.1) so requests from the emulator correctly resolve to the host backend.
Since the app implements SSL pinning, it initially blocks all interception attempts and displays a MITM attack warning popup.

<img width="448" height="528" alt="image" src="https://github.com/user-attachments/assets/d07cb828-2f14-4c5f-92b6-3e59289a8ec4" />


To bypass this, the application is launched through Frida using:

```
frida -U -f com.example.movierating -l bypass.js
```

This disables SSL pinning at runtime and allows BurpSuite to fully intercept and inspect the API calls.

Once MITM interception is active, two types of API security behaviours appear:
---

- 1. Protected API request (IDOR prevention implemented)

When submitting a review, the client sends:

```
POST /api/reviews HTTP/2
Host: 10.0.2.2:45678
Authorization: Bearer <token>.user1
{"comment":"hello","id":0,"movieId":1448560,"rating":3,"userId":1}
```

If the request is tampered with in Burp (e.g., by changing `userId` or injecting another user’s ID), the backend responds with:

```
403 Forbidden
{"error": "IDOR DETECTED: You cannot modify other users' data!"}
```
<img width="1493" height="608" alt="image" src="https://github.com/user-attachments/assets/3d769fd5-a761-4e7e-97aa-275f75f3298a" />


This shows that the backend implements proper authorization checks and detects horizontal privilege-escalation attempts. Even though interception is active, the attacker cannot submit reviews on behalf of other users.

---

- 2. Insecure API request (IDOR vulnerability – data exposure)

When loading the movie details screen, another request fetches reviews:

```
GET /api/reviews?userId=1
Authorization: Bearer <token>.user1
```

Although the client only requests reviews for user 1, the backend returns data belonging to user 2 as well:

<img width="1487" height="566" alt="image" src="https://github.com/user-attachments/assets/8796c76b-c092-4abf-8618-bdbd3c28ebfe" />


This insecure endpoint leaks other users’ reviews and movie interactions.
The vulnerability becomes visible in BurpSuite: when the response is intercepted or edited, the UI on the emulator updates in real time, confirming that the application directly reflects any returned JSON into the interface.

---

Together, these two requests demonstrate both a secure endpoint (proper IDOR protection during POST review submissions) and an intentionally insecure endpoint(review-fetching leaking data from other users).


### ![](./documentation/ReadmeImages/Database.png) Room database

The app uses a Room database to store local movie and rating information so that data remains available even when the device is offline. The database contains tables for movies (with fields such as movie ID, title, poster URL and description) and for user ratings (including the selected star rating, free‑text comment and any saved voice‑note path linked to a movie and user).

This data is written from the Movie Detail screen when the user rates a movie, adds a comment or saves a voice review, and is then read back to update both the Movie Detail screen and the Movie List screen. In the list view, the stored rating is shown next to each movie, while in the detail view the same data is used to prefill the star rating, the comment text and the list of available voice reviews for that specific movie.

### ![](./documentation/ReadmeImages/Database.png) Secure storage

The secure‑storage example in this app is the handling of voice‑note recordings created from the microphone. When the user records a voice review on the Movie Detail screen, the raw audio file is saved into the app’s private internal storage directory (under filesDir), which is sandboxed by Android so that only this application can read those files. The Room database stores only a reference path to each voice‑note file, not the audio contents themselves.

These stored paths are later used to load and play the recordings again on the Movie Detail screen, but at no point are the voice‑note files written to external or shared storage. This means an attacker cannot simply browse the device’s public folders to obtain the recordings, and must instead compromise the app or the device itself to access the microphone data.

### ![](./documentation/ReadmeImages/Database.png) Unsecure storage

- Insecure storage

The app deliberately stores all user rating data in an unprotected way to demonstrate insecure storage. Star ratings, free‑text comments and the file paths of recorded voice reviews are written directly into the local Room database via RatingDao and MovieRepository without any extra encryption or obfuscation.

This data is created and updated from the MovieDetailScreen when the user rates a movie, adds a comment or saves a voice note, and the same raw values are later read back and shown on both the MovieDetailScreen and MovieListScreen.

Because the Room database is just a normal SQLite file on the device, anyone with local access (for example via adb on an emulator or a rooted device) can open it and view or modify all stored ratings, comments and voice‑note paths in clear text.

- Unsafe storage of credentials (API key)

The project also includes an example of unsafe encryption for credentials through the handling of the TMDB API key.

Instead of being fetched securely at runtime, the API key is bundled inside the app and only Base64‑encoded in Base64Utils, then decoded and injected into Retrofit calls inside NetworkModule and MovieApiService / TmdbApiService. This means the key is not truly encrypted; Base64 can be trivially reversed by anyone who decompiles the APK or inspects the code, so an attacker can easily recover and reuse the TMDB API key.

The app intentionally stores the API key using an unsafe encoding method to show how credentials should not be protected in a real application.

### ![](./documentation/ReadmeImages/Notifications.png) Malware

For the malware component, a small malicious module was added to the original MovieRating app. The goal of this module is to secretly read the TMDB API key from the app and exfiltrate it (for example by logging or emailing it), without the user noticing any change in the normal UI.

The malware logic lives in a separate package `com.example.emailratings`, which contains an `EmailSender` class with two key methods: `testLog()` and `stealApiKey()`. The `stealApiKey()` function is responsible for accessing the API key that is stored unsafely in the app (Base64‑encoded in the networking layer), decoding it and preparing it to be sent out of the device.

To integrate this malicious code into the MovieRating app, both APKs were decompiled with apktool:

`malware.apk` contains the original com.example.emailratings implementation.

`originalmovieapp.apk` contains the original, clean MovieRating app.

`withmalware.apk` contains a test APK used to see how the malware calls were wired.

The `com/example/emailratings` smali package was copied from the malware project into the decompiled MovieRating app directory under `smali_classes6/com/example/emailratings`. Then the `MainActivity` smali in the MovieRating app was modified so that, during `onCreate`, it silently invokes the malware:

```
.line 34
invoke-super {p0, p1}, Landroidx/activity/ComponentActivity;->onCreate(Landroid/os/Bundle;)V

.line 37
sget-object v0, Lcom/example/movierating/EmailSender;->INSTANCE:Lcom/example/movierating/EmailSender;
invoke-virtual {v0}, Lcom/example/movierating/EmailSender;->testLog()V

.line 38
sget-object v0, Lcom/example/movierating/EmailSender;->INSTANCE:Lcom/example/movierating/EmailSender;
invoke-virtual {v0}, Lcom/example/movierating/EmailSender;->stealApiKey()V
```

(Adapted from the original malware smali, where the calls pointed to `com/example/emailratings/EmailSender`.)

These extra calls are placed right after `super.onCreate(...)`, so they run automatically whenever the app starts, before the user sees the main screen. The user interface continues to work as normal, but in the background `EmailSender.testLog()` and `EmailSender.stealApiKey()` are executed once on launch.

After injecting the malware package and changing the smali:

The modified app was rebuilt with apktool:
`apktool b originalmovieapp -o rebuilt.apk`

The resulting APK was aligned:
`zipalign -p -f 4 rebuilt.apk aligned.apk`

A new signing key was created with `keytool`, and the aligned APK was signed with `apksigner`.

The original app was uninstalled from the emulator, and the injected APK was installed with `adb install aligned.apk`.

In this state, the MovieRating app behaves normally from the user’s perspective, but every time it is launched the malicious `EmailSender` code runs inside the app process and can read and leak the hard‑coded API key. This demonstrates how malware can be added to an existing APK by decompiling, injecting extra code and recompiling, without access to the original Kotlin/Java source.




### ![](./documentation/ReadmeImages/Frida.png) Frida

The app has a trial implementation that only allows the user to rate three movies. This check is implemented in the checkTrial() function inside MovieViewModel, and the detail screen calls this function each time a star is tapped on the rating bar.

To bypass this limit with Frida, the app behaviour is modified while it is running:

First, the process ID (PID) of the MovieRatingApp is obtained with `frida-ps -Ua`, which lists all running apps so the correct PID can be selected.

Next, `frida-trace -U -j "*!*Trial*" -p <PID>` is used to trace every method whose name contains “Trial”. When a star is pressed, this trace shows that `MovieViewModel.checkTrial()` is being invoked, confirming it as the target for the bypass.

Frida is then attached again with a small JavaScript script. In this script, the MovieViewModel class is loaded and the checkTrial() method is hooked so that its implementation simply returns true every time. As a result, the trial check always “succeeds” and the “trial limit reached” dialog is never shown.

Because the UI logic only looks at the Boolean result of checkTrial() to decide whether rating is allowed, the user can now rate an unlimited number of movies. The ratings are still written to the Room database as usual; only the in‑memory trial enforcement is overridden during the Frida session.

![BypassTrialUsingFrida](./documentation/ReadmeImages/BypassTrialUsingFrida.png)

### Modify application

We demonstrate how the MovieRatingApp APK can be unpacked, modified at bytecode level, and rebuilt to change its behaviour and visuals without touching the original Kotlin source. The goal was to force a custom SpiderPoster image to appear as the poster for every movie on both the list and detail screens.​

- Decompiling the original APK
  The starting point is the signed MovieRatingApp.apk. Using apktool, the APK is decompiled into a folder structure containing resources and smali files (the human‑readable form of Dalvik bytecode). We locate the generated smali for the movie list item and the movie detail screen, which are responsible for loading and showing poster URLs.​

- Patching smali to override poster URLs
  In the detail screen and the list item smali, the code that receives the poster URL from the app logic originally writes the result of `getPosterUrl` into a register using move-result-object. These instructions are replaced with `const-string` instructions that load a fixed URL pointing to the SpiderPoster image. As a result, any value coming from the API is ignored and the same custom URL is always used instead.​

- Rebuilding, aligning, and signing the modified APK
  After editing the smali, the project is rebuilt into a new APK with apktool, then processed with `zipalign` to meet Android’s alignment requirements, and finally signed with a custom key using `apksigner`. The original application is uninstalled and the injected version is installed. This produces an APK that the system accepts as a normal app, but with the patched behaviour included.​

- Effect on the running app
  When the injected APK is launched, every movie in the list and detail screens now displays the `SpiderPoster` poster, independent of what the backend API returns. This illustrates how reverse engineering and smali patching can be used to alter an application’s presentation and logic purely by working on the compiled APK, an attacker can change how the app looks and behaves without needing the original Kotlin source.

<img src="./documentation/ReadmeImages/SpiderPoster_Screen1.png" alt="SpiderPoster_Screen1" width="220" />
<img src="./documentation/ReadmeImages/SpiderPoster_Screen2.png" alt="SpiderPoster_Screen2" width="220" />

### ![](./documentation/ReadmeImages/Root.png) Root

The app includes a simple root‑detection component that runs when the MovieViewModel is created. It calls a RootDetector helper class, which performs several checks such as looking at the system build tags for test-keys, scanning common file paths for su binaries or superuser apps, and trying to execute the su command to see if it is available. If any of these checks succeed, the device is treated as rooted.

The result of this detection is stored in a state variable that is observed by the UI. When the device is considered rooted, both the Movie List and Movie Detail screens change their behaviour: ratings are hidden or disabled, and voice‑review functionality is blocked. In the list screen this is shown as a warning message instead of the numeric rating, and in the detail screen the rating and voice sections display messages that the features are not available on rooted devices.

## Repositories

- Code
  - https://gitlab.ti.howest.be/ti/2025-2026/s3/mobilesecurity/students/group-17/movieratingapp
