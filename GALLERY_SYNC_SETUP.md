# Live Gallery Sync — Setup Guide

This makes your site's Gallery page automatically pick up new photos from your
shared Google Photos album, without you ever editing the HTML file again.

**Heads up before you start:** this involves Google Cloud Console and Google
Apps Script — genuinely a developer-facing setup, not a wedding-planning task.
It's about 15–20 minutes of clicking through screens that may not look
*exactly* like the steps below (Google changes these UIs periodically). If at
any point it feels like more trouble than it's worth, the fallback is simple:
just send me new photos whenever you like and I'll add them to the site
directly — that always works, no setup required.

**Also important:** `fetch()` (step 5) only works when the site is loaded over
`http://` or `https://` — opening the HTML file directly from your Desktop
(`file://...`) will silently skip the sync and show your existing curated
photos instead. This needs the site to actually be hosted somewhere (GitHub
Pages, Netlify, Vercel, or similar — all have free tiers) before it'll work.

---

## 1. Find your album ID

1. Open your shared album at [photos.google.com](https://photos.google.com) —
   the one guests are already uploading to
   (`https://photos.app.goo.gl/wgSf6wYjjk5B7c597`).
2. You need the album's internal ID (not the short share link). The easiest
   way: open [Google's OAuth Playground](https://developers.google.com/oauthplayground),
   or simpler — once you've deployed the script in step 4 below, you can
   temporarily log the album list and copy the right ID from there. If that
   sounds circular, the practical shortcut is: skip pre-finding it, deploy the
   script first, then come back and fill in `ALBUM_ID` once you can see your
   albums listed (see the troubleshooting note at the bottom).

## 2. Create the Apps Script project

1. Go to [script.google.com](https://script.google.com), signed in with the
   **same Google account** that owns/administers the shared album.
2. Click **New project**.
3. Delete the placeholder code and paste this in:

```javascript
/**
 * Sasha & Aman Wedding — Google Photos Gallery Sync
 * Deploy as a Web App. Proxies the Google Photos Library API so the
 * wedding site can fetch fresh photo URLs from the shared album live,
 * on every page load.
 */

const ALBUM_ID = 'PASTE_YOUR_ALBUM_ID_HERE';

function doGet(e) {
  const token = ScriptApp.getOAuthToken();
  const photos = [];
  let pageToken = null;

  do {
    const response = UrlFetchApp.fetch(
      'https://photoslibrary.googleapis.com/v1/mediaItems:search',
      {
        method: 'post',
        contentType: 'application/json',
        headers: { Authorization: 'Bearer ' + token },
        payload: JSON.stringify({
          albumId: ALBUM_ID,
          pageSize: 100,
          pageToken: pageToken
        }),
        muteHttpExceptions: true
      }
    );

    const data = JSON.parse(response.getContentText());
    if (data.mediaItems) {
      data.mediaItems.forEach(item => {
        // '=w800' asks Google for an 800px-wide version — plenty for
        // the gallery grid, and far lighter than full resolution.
        photos.push({ url: item.baseUrl + '=w800', caption: item.filename || '' });
      });
    }
    pageToken = data.nextPageToken;
  } while (pageToken);

  return ContentService
    .createTextOutput(JSON.stringify(photos))
    .setMimeType(ContentService.MimeType.JSON);
}

// Temporary helper — run this once (see step 3) to print your albums
// and their IDs to the execution log, then delete it.
function listMyAlbums() {
  const token = ScriptApp.getOAuthToken();
  const response = UrlFetchApp.fetch(
    'https://photoslibrary.googleapis.com/v1/albums?pageSize=50',
    { headers: { Authorization: 'Bearer ' + token }, muteHttpExceptions: true }
  );
  Logger.log(response.getContentText());
}
```

## 3. Grant it access to the Photos Library API

By default, Apps Script only has permission scopes for the services your code
obviously uses. `photoslibrary.readonly` needs to be added by hand:

1. In the editor, click the gear icon (**Project Settings**) in the left
   sidebar and check **"Show appsscript.json manifest file in editor"**.
2. Open `appsscript.json` (now visible in the file list) and add an
   `oauthScopes` array so it looks like this:

```json
{
  "timeZone": "Asia/Kolkata",
  "dependencies": {},
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/script.external_request",
    "https://www.googleapis.com/auth/photoslibrary.readonly"
  ],
  "webapp": {
    "access": "ANYONE_ANONYMOUS",
    "executeAs": "USER_DEPLOYING"
  }
}
```

3. You also need the **Photos Library API enabled** on the Google Cloud
   project behind this script:
   - Still in Project Settings, under **Google Cloud Platform (GCP) Project**,
     note the project number (or switch to a project you control if you have
     one — either works).
   - Visit [console.cloud.google.com](https://console.cloud.google.com),
     select that project, go to **APIs & Services → Library**, search
     **"Photos Library API"**, and click **Enable**.
   - If prompted to configure an **OAuth consent screen**, choose **External**,
     fill in an app name (e.g. "Sasha Aman Wedding Gallery") and your email,
     and add your own Google account under **Test users**. Since only you are
     ever authorizing this script, it doesn't need Google's formal
     verification — it can stay in "Testing" mode indefinitely.

## 4. Find your album ID (for real this time)

1. Back in the Apps Script editor, select the `listMyAlbums` function from
   the function dropdown (top toolbar) and click **Run**.
2. The first run will prompt you to authorize the script — click through,
   choose your account, and click **Advanced → Go to [project name] (unsafe)**
   if Google shows an "unverified app" warning (expected, since this is your
   own private script).
3. Open **View → Logs** (or `Ctrl+Enter`) to see the album list with IDs.
   Find your shared wedding album and copy its `id`.
4. Paste that into `ALBUM_ID` near the top of the script, replacing
   `PASTE_YOUR_ALBUM_ID_HERE`.
5. Delete the `listMyAlbums` function now — it's served its purpose.

## 5. Deploy as a Web App

1. Click **Deploy → New deployment**.
2. Click the gear next to "Select type" and choose **Web app**.
3. Set **Execute as: Me**, **Who has access: Anyone**.
4. Click **Deploy**, authorize again if asked, and copy the resulting URL —
   it ends in `/exec`.

## 6. Wire it into the site

Open `Sasha_Aman_Wedding_v8.html`, find this line (search for `GALLERY_SYNC_URL`):

```javascript
const GALLERY_SYNC_URL = 'PASTE_YOUR_APPS_SCRIPT_WEB_APP_URL_HERE';
```

Replace the placeholder with the `/exec` URL from step 5, save, and re-deploy
the site. New photos guests add to the album will now appear in the gallery
grid automatically, the next time anyone opens the site.

---

## If something doesn't work

- **Gallery just shows the same old photos, no error:** most likely the site
  is still being opened as a local file rather than hosted — `fetch()` fails
  silently in that case by design (so it never breaks the gallery for
  guests). Double check it's deployed to an actual `https://` URL.
- **"Photos Library API has not been used in project..." error:** the API
  wasn't enabled on the right Cloud project — revisit step 3.
- **Nothing in the album shows up:** the account running the script needs to
  actually be a member of the shared album (owner or someone who's tapped
  "Join album"), not just someone with the link.

Send me whatever error you're seeing and I'll help you debug it from there.
