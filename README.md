# Trade Chat

A one-file chat front end for the n8n webhook at
`https://nikolakuze.app.n8n.cloud/webhook/dtjournal-chat`.

Everything lives in `index.html`: markup, CSS, JavaScript, and both app icons as
inline base64 PNGs. There is no build step and no second file to upload.

## Running it

Locally:

```bash
powershell -ExecutionPolicy Bypass -File dev-server.ps1
```

then open <http://localhost:8000/>. `file://` also works for a quick look, but
the browser blocks the cross-origin POST from a `file://` page, so nothing will
send.

To use it from the phone, upload `index.html` anywhere that serves over HTTPS
(GitHub Pages, Netlify drop, Cloudflare Pages). HTTPS is required: iOS and
Android both refuse to install a home-screen app from a plain HTTP page, and a
`https://` webhook cannot be called from an `http://` page anyway.

## Add to Home Screen

The web app manifest is built in JavaScript at load time and attached as a
`data:` URL, which is what keeps this to a single file. `start_url` and `scope`
are taken from wherever the page is actually served, so the same file works on
any host.

- **Android / Chrome**: menu, then "Add to Home screen" or "Install app".
- **iOS / Safari**: Share, then "Add to Home Screen". iOS reads the
  `apple-touch-icon` and the `apple-mobile-web-app-*` meta tags rather than the
  manifest, so it opens full-screen without Safari's chrome.

## The request

```
POST https://nikolakuze.app.n8n.cloud/webhook/dtjournal-chat
Content-Type: application/json

{ "message": "...", "sessionId": "..." }
```

The expected answer is `{ "reply": "...", "sessionId": "..." }`. If the
workflow's Respond node is left on its default settings n8n often returns the
item wrapped in an array, or names the text `output` instead of `reply`, so the
page reads the first item and accepts `reply`, `output`, `text`, `message`,
`answer` or `response`. When the answer carries a `sessionId`, that value
replaces the stored one.

The CORS headers have to come from n8n. If replies never arrive but the n8n
execution log shows the run succeeding, add
`Access-Control-Allow-Origin: <your page's origin>` to the Respond to Webhook
node, and make sure the webhook answers `OPTIONS` too.

## State

Both keys live in `localStorage`, per browser:

- `dtchat.sessionId`: generated once with `crypto.randomUUID()` on first load
  and sent with every request, so the workflow can keep context.
- `dtchat.log`: the visible transcript, last 200 messages, so reopening the app
  does not show an empty screen. **New chat** clears it and generates a fresh
  session id.

A blocked `localStorage` (private mode, site data turned off) is caught rather
than thrown, so the app still runs, just without memory between loads.

## Behaviour worth knowing

- Requests are given up on after 90 seconds. Failed sends leave the message in
  place with a Retry button, which re-sends the same text without duplicating
  the bubble.
- Reply text is written with `textContent`, never `innerHTML`, so a reply
  containing HTML is shown as text instead of being rendered.
- On a touch screen Enter inserts a newline and the button sends; with a
  physical keyboard Enter sends and Shift+Enter inserts a newline.
- The layout follows `visualViewport` so the composer stays above the iOS
  keyboard, and uses `env(safe-area-inset-*)` for the notch and home bar.
