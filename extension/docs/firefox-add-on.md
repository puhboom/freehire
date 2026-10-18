# Publishing to Firefox Add-ons (AMO)

Everything the dashboard asks for, kept next to the manifest it has to match.
Reviewers cross-check this copy against `extension/wxt.config.ts` and the code —
when one changes, change the other.

## Build

```bash
cd extension && npm run build:firefox   # .output/firefox-mv3/
cd extension && npm run zip:firefox     # .output/freehire-extension-<version>-firefox.zip
```

The Firefox build drops Chrome-only manifest keys (`key`, `icon_variants`,
`debugger` permission, `icon_variants`) and sets
`browser_specific_settings.gecko.id` instead — a stable id so the extension's
identity (and therefore its OAuth redirect origin) stays the same across builds.

## Release checklist

1. **Bump the version** in `extension/package.json` (`version`). AMO rejects a
   re-upload of a version it already has.
2. **Get the package** — `cd extension && npm run zip:firefox`
   (`.output/freehire-extension-<version>-firefox.zip`).
3. **Upload** to [addons.mozilla.org](https://addons.mozilla.org/developers/)
   (a Firefox Account, no one-time fee).
4. Fill the listing, privacy practices, and permission justifications below.
5. Submit. Expect an in-depth manual review — `<all_urls>` guarantees it.
6. **After the item exists**, take its extension ID from the dashboard and add
   `https://<id>.mozilla-extension.org/` to the redirect-URI allowlist of
   `freehire.me/api/v1/auth/extension/connect`. Until that lands, sign-in fails
   for everyone who installs from AMO — the published build carries AMO's own id,
   not the dev `gecko.id` pinned in `wxt.config.ts`.

## Listing

**Name:** freehire

**Short description** (≤132 chars):

> A job-application agent in your side panel: it reads the posting you are on, matches it to your CV, and fills the form.

**Single purpose** (the store requires exactly one):

> Assist the user with job applications on the page they are currently viewing.

**Detailed description:**

> freehire puts a job-application agent in Firefox's side panel, next to whatever page you are on.
>
> - Ask it about the posting you are reading. It reads the page itself when your question needs it, so you can just say "is this a fit?" and get an answer about the tab in front of you.
> - See how the posting matches your freehire profile — a deterministic skill-coverage card, not a guess.
> - Let it fill the application form. It reads the form's fields and writes the values from your freehire profile; you review and submit yourself.
>
> You need a freehire account. Sign in once from the panel; the extension does nothing until you do.

**Category:** Productivity. **Language:** English.

## Permission justifications

Paste one per field in the dashboard.

| Permission | Justification |
| --- | --- |
| `storage` | Stores the freehire session token issued by sign-in, plus nothing else. Local to the browser profile. |
| `tabs` | The panel needs the active tab's id and URL to know which page the user is asking about and to route the agent's read/fill calls to it. |
| `sidePanel` | The entire UI is a side panel opened from the toolbar icon. |
| `scripting` | Injects the reader into the active tab's frames on demand, so the agent can read the application form and fill the fields the user asked it to. |
| `activeTab` | Scopes page access to the tab the user explicitly acted on from the panel. |
| `identity` | `launchWebAuthFlow` runs "Sign in with freehire" and returns the session token to the extension. Not used for Firefox account data. |
| `downloads` | Briefly saves the user's tailored CV PDF to disk so it can be placed into a file upload field ("Attach tailored CV"). The file is read from disk and never otherwise retained. |
| `host_permissions: <all_urls>` | Applications live on arbitrary hosts — Greenhouse, Lever, Workday, Ashby, and thousands of company career pages. The user decides which page the agent works on, so the host set cannot be enumerated in advance. Page content is read only while the panel is open and only in service of what the user asked for — the question they typed, or the Autofill they pressed. Nothing is read in the background: the channel the reads travel over exists only for as long as the panel is. |

## Feature differences vs Chrome

**Attach tailored CV** — unavailable on Firefox. Firefox does not implement the
Chrome debugging protocol (`browser.debugger`), which this action uses to write a
file into a page's `<input type="file">`. On Firefox the action reports
"Attaching a tailored CV is not available in Firefox — open the job page in
Chrome instead." Everything else (sign in, match card, chat, agent autofill,
form autofill, read page) works identically.

## Privacy practices

Data collected — declare all three:

- **Personally identifiable information** — name, email, phone and CV fields from
  the user's freehire profile, fetched to fill application forms.
- **Authentication information** — the freehire session JWT, kept in
  `storage.local`.
- **Website content** — the URL, title and visible text (capped at 5000
  characters, see `extension/lib/scraper.ts`) of the page open while the user is
  asking about it, plus the form fields Autofill reads. Sent to `freehire.me`,
  which is the only host this extension contacts.

Two things worth stating plainly, because they changed: the agent now decides
when to read the page — during a turn, when the user's question is about what is
in front of them — rather than the user pressing a button to hand it over. And a
page read this way is kept in that conversation's transcript, which the user can
read and delete from their freehire account.

Reads are bounded and visible. Only ordinary web pages are read: a browser
settings page, another extension's page, or a local file is refused before it is
touched. And every read is named in the conversation itself — the panel shows
which page it read, without the query string, so a read the user did not expect
is one they can see.

Not collected: health, financial, location, personal communications, browsing
history, keystroke logging.

Certifications to tick: data is not sold to third parties, not used or
transferred for purposes unrelated to the extension's single purpose, and not
used for creditworthiness or lending.

The privacy policy at the URL you enter must say the same thing and state
compliance with the Firefox Add-ons policies. It has to be reachable over HTTPS
without a login — the review bot crawls it.

## Automated publishing (after the first manual release)

The AMO API can update an existing item once it exists. The first listing must
be created manually in the dashboard. Subsequent version bumps can be uploaded
through AMO's tooling (`web-ext` or the AMO API) — gated on a GitHub Release
having created a new tag, so an unchanged version is never re-uploaded.

It's a no-op until secrets exist on the repo. One-time setup:

1. [AMO Developer Hub](https://addons.mozilla.org/developers/) → create an API
   key (JWT issuer + secret).
2. Add two repo secrets (Settings → Secrets and variables → Actions):
   `AMO_API_KEY` and `AMO_API_SECRET`.

Until those secrets exist, every release is a manual dashboard upload — which
is also the rollback: remove the published version from AMO and the previous
one (or none) is what users have.
