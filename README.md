# LABORO — PWA ➜ TWA (APK) Guide

---

## 0) Prerequisites & layout

* Your Next.js repo should keep **web code at root**, and the **Android (Bubblewrap) project under `android/`**.
* Ensure `.gitignore` excludes Android build output:

  ```
  android/.gradle/
  android/app/build/
  android/build/
  *.apk
  *.apks
  *.aab
  *.keystore
  *.jks
  *.idsig
  ```

---

## 1) Install PWA libraries

```
npm i next-pwa workbox-window
```

* `next-pwa` generates a service worker into `public/` (`dest: 'public'`).
* If you use the App Router and server actions, test that the SW does not cache POSTs you need to reach the server.

---

## 2) Configure `next.config.js`

```js
// next.config.js
const withPWA = require('next-pwa')({
  dest: 'public',
  register: true,
  skipWaiting: true
});
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  images: { domains: ["lh3.googleusercontent.com","pbs.twimg.com","images.unsplash.com","logos-world.net"] },
  swcMinify: true,
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production' ? { exclude: ['error','warn'] } : false,
  },
  poweredByHeader: false,
  productionBrowserSourceMaps: false,
};
module.exports = withPWA(nextConfig);
```

* The PWA SW will be served from `/sw.js`. Confirm your CDN/ingress doesn’t block it.
* If you later switch to a static export or standalone output, re-test SW registration.

---

## 3) Web App Manifest

```ts
// app/manifest.ts
import { MetadataRoute } from 'next';

export default function manifest(): MetadataRoute.Manifest {
  return {
    name: 'Laboro',
    short_name: 'Laboro',
    start_url: '/',
    scope: '/',
    display: 'standalone',
    background_color: '#ffffff',
    theme_color: '#5b5bd6',
    icons: [
      { src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
      { src: '/icon-512.png', sizes: '512x512', type: 'image/png' },
      { src: '/maskable-512.png', sizes: '512x512', type: 'image/png', purpose: 'maskable' },
    ]
  };
}
```

* If your icons live under `/twa-icons/`, either:

  * change the `src` values to `/twa-icons/icon-192.png`, `/twa-icons/icon-512.png`, etc., **or**
  * keep the above and place copies of those icon files at the root of `/public` with those names.
* The manifest must be reachable at `/manifest.webmanifest`. With App Router, `app/manifest.ts` serves this automatically.

---

## 4) Icons

Add icons in public/twa-icons; I already generated the following in the repo:

```
"iconUrl": "https://pre.laboro.co/twa-icons/icon-512.png",
"maskableIconUrl": "https://pre.laboro.co/twa-icons/maskable-512.png",
"monochromeIconUrl": "https://pre.laboro.co/twa-icons/monochrome-512.png",
```

Note: I created a public repo for icons and manifest hosting bcs or ingress rules blocked their public hosting without auth header via the gradle curl

```
https://github.com/ihcsof/laboro-pwa-assets/tree/master
```

* For Bubblewrap, use **public, anonymous** URLs (e.g., `raw.githubusercontent.com/...`) if your domain enforces auth.
* In production, ONLY your own domain should serve these with **200** and `image/png`.

---

## 5) Install Bubblewrap & initialize

```
npm i -g @bubblewrap/cli

bubblewrap init --manifest=https://pre.laboro.co/manifest.webmanifest

-> chose standalone (it's the default)
-> note, the color is: #2596be
```

* If your domain blocks the manifest/icons, you can point `--manifest` to the **RAW GitHub manifest** temporarily.
* After init, Bubblewrap creates `twa-manifest.json`. Keep this **in `android/`** so builds run cleanly there.

---

## 6) Keystore creation (signing)

```powershell
& "$env:USERPROFILE\.bubblewrap\jdk\jdk-17.0.11+9\bin\keytool.exe" `
  -genkeypair -v `
  -keystore "C:\Users\loren\Desktop\LABORO\frontend_service\android.keystore" `
  -alias android `
  -keyalg RSA -keysize 2048 -validity 10000 `
  -storepass 123456 -keypass 123456 `
  -dname "CN=LABORO, OU=IT, O=LABORO, L=Milan, S=Lombardy, C=IT"
```

* **Do not commit** prod keystores; store them **outside the repo** (e.g., `C:\Keys\laboro-release.keystore`) and back them up securely.
* In `android/twa-manifest.json`:

  ```json
  "signingKey": {
    "path": "C:\\Keys\\laboro-release.keystore",
    "alias": "android"
  }
  ```

---

## 7) **Certificate handling (Digital Asset Links)** ✅

This is crucial for opening **without the browser bar**.

### 7.1 Get the SHA-256 fingerprint (release)

Option A — from keystore:

```powershell
$env:JAVA_HOME = "$env:USERPROFILE\.bubblewrap\jdk\jdk-17.0.11+9"
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
& "$env:JAVA_HOME\bin\keytool.exe" -list -v `
  -keystore "C:\Keys\laboro-release.keystore" `
  -alias android
```

Option B — from the signed APK:

```powershell
& "$env:USERPROFILE\.bubblewrap\android_sdk\build-tools\34.0.0\apksigner.bat" `
  verify --print-certs .\android\app-release-signed.apk
```

Copy the **SHA-256** (with colons).

### 7.2 Publish `/.well-known/assetlinks.json`

```
public/.well-known/assetlinks.json
```

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "co.laboro.pre.twa",
    "sha256_cert_fingerprints": ["PASTE:YOUR:SHA256:HERE"]
  }
}]
```

**Ingress/APISIX allowlist (if you gate the site):**
Add a **public** route (no auth) for:

* `/.well-known/assetlinks.json`
* `/manifest.webmanifest`
* `/twa-icons/*`

**Verify (must be 200):**

```powershell
curl.exe -I "https://pre.laboro.co/.well-known/assetlinks.json"
```

### 7.3 Reinstall & test TWA

* **Uninstall** the app from your phone, then install the **signed** APK.
* Open it: with matching `assetlinks.json` on **the same origin** and **no redirect to a different domain on first load**, the app opens **full-screen (no URL bar)**.

### 7.4 Play Store (if publishing later)

If you use **Play App Signing**, Google re-signs the AAB. Add **both** fingerprints to `assetlinks.json`:

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": { "namespace": "android_app", "package_name": "co.laboro.pre.twa",
      "sha256_cert_fingerprints": ["YOUR:LOCAL:SHA256"] }
  },
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": { "namespace": "android_app", "package_name": "co.laboro.pre.twa",
      "sha256_cert_fingerprints": ["PLAY:SIGNING:SHA256"] }
  }
]
```

You can copy the Play signing cert fingerprint from the Play Console once you enroll in Play App Signing.

**Heads-up:**

* If you ever **change keystore**, update `assetlinks.json` with the new SHA-256.
* If you **redirect** from `pre.laboro.co` to another origin on first paint, either stop the redirect or also publish `assetlinks.json` on the **final** origin and change Bubblewrap’s `host/fullScopeUrl`.

---

## 8) Next.js `middleware.ts` bypass


```ts
// --- ALWAYS BYPASS static + PWA/TWA assets ---
  if (
    pathname.startsWith("/_next") ||
    pathname.startsWith("/twa-icons") ||
    pathname.startsWith("/.well-known") ||        // assetlinks.json
    pathname === "/manifest.webmanifest" ||
    pathname === "/robots.txt" ||
    pathname === "/favicon.ico" ||
    /\.(?:png|jpg|jpeg|gif|webp|svg|ico|json|txt|css|js|map)$/i.test(pathname)
  ) {
    return NextResponse.next();
  }
```

* Keep this **at the top** of your middleware exported function to avoid accidental rewrites/401s for PWA/TWA assets.

---

## 9) Gradle memory settings (Windows)

**Original (keep):**

```
org.gradle.jvmargs=-Xmx768m -Dfile.encoding=UTF-8
org.gradle.workers.max=1
org.gradle.parallel=false
org.gradle.daemon=false
kotlin.daemon.jvmargs=-Xmx512m
```

**Notes (new):**

* If memory errors persist, drop to `-Xmx512m` and `kotlin.daemon.jvmargs=-Xmx384m`.

---

## 10) Build the Android app (in `android/`)

```
mkdir android
cd android
bubblewrap build
(o bubblewrap update)

(password versione di prova: 123456)
```

* Ensure `android/twa-manifest.json` exists (or run with `--manifest=..\twa-manifest.json`).
* If Bubblewrap fails to fetch icons/manifest from your domain, temporarily use **RAW GitHub** URLs for those fields during build, then switch back.

---

## 11) Troubleshooting 🎯

* **Address bar visible:** `assetlinks.json` missing/wrong/mismatched SHA-256, or first load redirects to a **different** origin.
* **401 on icons/manifest/assetlinks:** Ingress/WAF enforced auth. Add a **public** route for those paths.
* **404 on assetlinks/manifest:** Not shipped in the image. Either:

  * confirm `public/.well-known/assetlinks.json` is in the container (and not ignored by `.dockerignore`)
  * **Bubblewrap “invalid Content-Type”:** Use **raw\.githubusercontent.com** for icons when building.

---
