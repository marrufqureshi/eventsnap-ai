# EventSnap AI

QR signup, selfie, face matching, private gallery link, WhatsApp delivery.

## Why this stack
Next.js 14 (App Router) with Prisma + SQLite and Tailwind: one codebase, one `npm install`, no Python or native build step. Face detection and embedding (`@vladmandic/face-api`, 128-d descriptors) run in the browser, so selfies and admin batches are processed on the user's own device. The server only stores descriptors and does the distance matching (`lib/matching.js`). For Postgres, change `provider` in `prisma/schema.prisma` and `DATABASE_URL`.

## Data flow
1. Attendee scans the QR and lands on `/e/<event>`. The browser captures a selfie, checks there is exactly one face, and extracts its descriptor. `POST /api/register` stores the profile, selfie, descriptor, consent time and a random gallery token.
2. Admin signs in at `/admin` and drops photos. Each photo is downscaled, faces are detected and described in the browser, then `POST /api/admin/upload` saves the file and its faces.
3. `POST /api/admin/match` assigns each face to the nearest attendee when Euclidean distance <= threshold (default 0.5, lower is stricter) and writes `photo_matches`. Manual tags survive re-runs.
4. `notifyPending` sends each matched attendee a WhatsApp template message (plus email if `SMTP_URL` is set) with `/g/<token>`. Each attempt is logged `pending`, `sent` or `failed`.
5. `/g/<token>` shows only that attendee's photos. Files are served by `/api/img/<id>?t=<token>` (token checked) and `/api/zip/<token>`.

## Schema
- `Attendee`: id, eventId, name, phone, email, selfiePath, embedding, galleryToken (unique), consentAt
- `EventPhoto`: id, eventId, path, filename, faces (JSON: box + descriptor per face)
- `PhotoMatch`: photoId, faceIndex (unique together), attendeeId, distance, source `auto|manual`
- `NotificationLog`: attendeeId, channel `whatsapp|email`, status `pending|sent|failed`, providerId, error

## Setup
```bash
npm install                      # also copies face models into /public
cp .env.example .env             # set ADMIN_PASSWORD and SESSION_SECRET
npx prisma db push
npm run dev
```
Test locally: open `/admin` (event `default`), download the QR, open `/e/default` on your laptop (localhost counts as secure for the camera), register with a selfie, upload photos containing your face, press "Run matching". With `WHATSAPP_MODE=mock` the message and gallery link print in the server console.

Test on a phone: camera access needs HTTPS. Run `npx cloudflared tunnel --url http://localhost:3000` (or ngrok), set `APP_URL` to the https URL, and scan the QR from `/admin` opened on that URL.

## Live WhatsApp
Create a Meta app with WhatsApp Cloud API, then set `WHATSAPP_MODE=live`, `WHATSAPP_TOKEN` (permanent system-user token) and `WHATSAPP_PHONE_NUMBER_ID`. Create and get approved a Utility template named `gallery_ready` with body `Hi {{1}}, your event photos are ready: {{2}}`. Template messages are required because attendees have not messaged you in the last 24 hours.

## Production notes
- Serve over HTTPS (camera and secure cookies need it). Set a strong `SESSION_SECRET` and `ADMIN_PASSWORD`.
- Consent: the form records explicit consent. Add a privacy notice, a retention period, and a delete-my-data flow (India's DPDP Act 2023, GDPR and similar rules treat face data as sensitive).
- Descriptors are computed on the client, so a malicious client could submit a forged one. For stricter trust, recompute descriptors on the server from the stored images.
- The rate limiter is in-memory; use Redis or your CDN for multi-instance deployments. Matching re-runs the whole event; for thousands of faces move it to a background job or a vector index (pgvector).
- Swap local `storage/` for S3 by replacing `saveFile`, `readFile` and `absPath` in `lib/core.js`, and use signed URLs for downloads.
