# Learning-Management-System---Google-Apps-Script-

# Cohort — Interactive Learning Module App

A two-sided interactive learning module web application built entirely on **Google Apps Script (GAS)** with **Google Sheets as the database**. Lecturers create courses and interactive activities (Question, Table, and Checklist), and each activity generates an embed that students can use anonymously within platforms such as Notion or Canvas.

Developed for the **Wits/MERSETA AI & Software Systems Internship (Task 2)**.

Everything described below has been implemented and tested in a deployed, live Google Apps Script web application, which is the same system provided for evaluation.

> **Status:** Fully implemented, designed, and validated end-to-end. Includes course and activity CRUD, all three activity types with peer interaction, lecturer response views, an AI-powered lecturer insight feature, Google OAuth sign-in for lecturers, and a polished Cohort user interface.

---

# Live Links

* **Lecturer Dashboard:**
  https://script.google.com/macros/s/AKfycbw7qfXAy3rdjlhDdD8KR91-TONhDkPcgNGR7ECuM84OC-fKCt6czq-wroIUhEXALNVw/exec?view=lecturer 
  Sign in with Google. The first visit will display Google's **Unverified App** screen. Select **Advanced → Continue**. This is expected for an unverified GAS application using sensitive OAuth scopes (see Sections 4 and 7).

* **Student Experience (embedded in Notion):**
  https://app.notion.com/p/Cohort-Learner-Activity-Space-3776d517caae809c810afa86f21cb883 
  Activities exactly as students use them. No login required.

* **Source Code**
  https://script.google.com/d/1P9BW1fy4dnr7F9qbcM8U00ueRVxVZisdsByFpvOs9f_k8Iia5eA1A6w1/edit?usp=sharing l 

> **Multi-account login note:** If your browser is signed into multiple Google accounts simultaneously, Google's serving layer may fail to load embeds. This is a known platform limitation rather than an application issue (see Section 7). Use a single-account browser session or an Incognito window if necessary.

---

# 1. Architecture Overview

The application is deployed as a single GAS web app (`Execute as: Me`, `Access: Anyone`) and routed through `doGet`.

| Route                                 | Purpose                                                | Authentication                 |
| ------------------------------------- | ------------------------------------------------------ | ------------------------------ |
| `.../exec`                            | Home (`Home.html`) – Lecturer / Student entry          | None                           |
| `.../exec?code=...`                   | OAuth callback → session creation → lecturer dashboard | Google OAuth                   |
| `.../exec?view=lecturer`              | Lecturer dashboard (`Lecturer.html`)                   | Session token (`localStorage`) |
| `.../exec?view=student&activity=<id>` | Student embed (`Student.html`)                         | None                           |

### Authentication Model

* Lecturer identity is established through **Google OAuth 2.0**, verified server-side, and represented by a signed session token.
* Authentication is independent of deployment permissions.
* The deployment remains `Execute as: Me`, allowing access to a private spreadsheet while supporting multiple lecturers.
* Students never sign in. They provide a name and student number that are stored locally in the browser.

### Technology Stack

* **Frontend:** HtmlService templated pages
* **Database:** Google Sheets (`SpreadsheetApp`)
* **Images:** Google Drive (`DriveApp`)
* **AI:** Gemini via `UrlFetchApp`
* **Concurrency:** `LockService`
* **IDs:** `Utilities.getUuid()`
* **Configuration & Secrets:** `PropertiesService`

### File Structure

| File                                                       | Responsibility                                                         |
| ---------------------------------------------------------- | ---------------------------------------------------------------------- |
| `Code.js`                                                  | Routing, OAuth callback, authenticated dispatcher, setup helpers       |
| `Auth.js`                                                  | OAuth flow, token verification, HMAC session tokens                    |
| `Util.js`                                                  | Utilities, properties, spreadsheet access, locking, generic data layer |
| `Courses.js`                                               | Course CRUD and ownership validation                                   |
| `Activities.js`                                            | Activity CRUD, embed generation, image uploads                         |
| `Responses.js`                                             | Student submissions and response retrieval                             |
| `Interactions.js`                                          | Likes, dislikes, constructive replies                                  |
| `Summary.js`                                               | AI Lecturer Insight feature                                            |
| `Home.html`                                                | Landing page and lecturer sign-in                                      |
| `Lecturer.html`                                            | Lecturer dashboard                                                     |
| `Student.html`                                             | Student activity embed                                                 |
| `Styles.html`, `LecturerStyles.html`, `StudentStyles.html` | Shared and contextual styling                                          |
| `AccountsJS.html`                                          | Student profile storage and management                                 |

All server-callable functions return:

```javascript
{ ok: true, data }
```

or

```javascript
{ ok: false, error }
```

## UI & Responsiveness (Criterion 3.1)

### Lecturer Interface

* Course and activity creation/editing use accessible modal dialogs.
* Modals support:

  * Escape key dismissal
  * Click-outside dismissal
  * Close button
  * Automatic focus management
* Profile menu includes:

  * Lecturer email
  * Sign Out
  * Master Dashboard navigation

### Student Interface

* Profile menu available directly within the embed.
* Stores up to three local student profiles for testing and convenience.
* Supports:

  * Login by student number
  * Logout
  * Profile removal
  * Tooltips and profile management UI

### Responsive Design

* Student experience is mobile-first.
* Images stack above text below 540px width.
* No horizontal scrolling.
* Touch-friendly controls.
* Lecturer dashboard automatically reflows on smaller screens, including:

  * Course grids
  * Forms
  * Activity selectors
  * Modal layouts

---

# 2. Google Sheets Data Structure & Rationale (Criterion 3.2)

The system uses a single spreadsheet with four tabs.

This approach provides:

* One permission boundary
* Simplified maintenance
* No cross-file joins
* Co-located data for efficient GAS operations

| Tab          | Columns                                                                                                                          |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Courses      | `courseId, ownerEmail, name, code, description, createdAt`                                                                       |
| Activities   | `activityId, courseId, type, moduleNumber, sessionName, title, text, imageFileId, viewingMode, anonymityMode, config, createdAt` |
| Responses    | `responseId, activityId, studentName, studentNumber, isAnonymous, content, likes, dislikes, createdAt`                           |
| Interactions | `interactionId, parentResponseId, activityId, authorName, authorStudentNumber, type, text, createdAt`                            |

### Design Notes

* `config` stores activity-specific settings as JSON.
* `content` stores activity responses as JSON.
* Likes and dislikes are stored both as counters and interaction records:

  * Fast sorting and display
  * Audit trail
  * Double-vote prevention
* Every entity uses UUIDs.
* Replies reference `parentResponseId`.

### Performance

Operations read entire sheets using:

```javascript
getDataRange().getValues()
```

and perform bulk writes rather than cell-by-cell updates, helping remain within Apps Script execution limits.

---

# 3. Notion and LMS Embedding Compatibility (Criterion 3.3)

### Embed Support

The student page is served using:

* `HtmlService.XFrameOptionsMode.ALLOWALL`
* `SandboxMode.IFRAME`

This allows embedding within external platforms.

### Responsive Embeds

The embed:

* Uses fluid layouts
* Avoids fixed heights
* Does not rely on `postMessage` resizing
* Works within Notion's iframe restrictions

### Embed Code Generation

`generateEmbedCode(activityId)` provides:

#### Notion / Google Sites

A URL that the platform converts into an iframe.

#### Canvas / LMS Editors

A complete iframe snippet ready for pasting into HTML editors.

### Anonymous Access

Deployment settings:

* Execute as: Me
* Access: Anyone

This allows students to access activities without authentication.

### Known Platform Limitation

Browsers signed into multiple Google accounts may occasionally encounter embed-loading issues originating from Google's serving layer. The application works correctly in:

* Single-account sessions
* Incognito mode
* Notion's partitioned iframe environment

---

# 4. Google Sign-In & Authentication Design (Criterion 3.4)

## Objective

Authenticate lecturers using Google accounts while:

* Keeping the spreadsheet private
* Supporting multiple lecturers
* Maintaining a single deployment

## Why OAuth Redirect Instead of GIS

Google Apps Script HtmlService pages run inside sandboxed iframes hosted on `googleusercontent.com`.

Google OAuth does not allow these domains as JavaScript origins, preventing the standard Google Identity Services button from functioning.

The application therefore uses the OAuth 2.0 Authorization Code Flow with a redirect URI hosted on `script.google.com`.

## Authentication Flow

### 1. Lecturer Sign-In

`Home.html` redirects users to Google's consent screen using:

```javascript
buildAuthUrl_()
```

A CSRF state token is generated and stored server-side.

### 2. OAuth Callback

`serveAuthCallback_()`:

* Verifies state
* Exchanges the authorization code
* Validates the returned ID token

Validation includes:

* Audience
* Issuer
* Expiry
* `email_verified`

### 3. Session Creation

```javascript
mintSession_()
```

creates an HMAC-SHA256 signed token:

```text
base64(email|expiry).signature
```

### 4. Authenticated Requests

All lecturer actions pass through:

```javascript
lecturerCall(sessionToken, action, args)
```

which verifies the session and dispatches requests.

### 5. Ownership Enforcement

```javascript
requireLecturer_()
assertOwnership_()
```

validate ownership for every mutation request.

### Configuration

Stored in Script Properties:

* OAuth Client ID
* OAuth Client Secret
* Redirect URI
* Session Secret

---

# 5. Innovative Feature: AI Lecturer Insight (Criterion 3.5)

A Generate button converts student responses into actionable teaching insights.

The system is type-aware and uses AI only where it adds value.

### Question Activities

Uses Gemini (`gemini-2.5-flash`) to identify:

* Themes
* Misconceptions
* Outliers
* Suggested follow-up teaching points

Responses are anonymised before analysis.

### Checklist Activities

No LLM usage.

Provides:

* Completion percentages
* Lowest-completion items
* Average completion rate
* Submission counts

### Table Activities

Analysis varies by column type:

* Text → Gemini summary
* Numeric → Min, max, average
* Categorical/date → Distributions

### Rationale

Provides lecturers with rapid visibility into class understanding while keeping costs low and analysis targeted.

The Gemini API key is stored securely in Script Properties.

---

# 6. Setup & Deployment

## Initial Setup

1. Run `initSpreadsheetId()`.
2. Run `setup()` to create:

   * Four sheets
   * Image folder
3. Run `initStudentBaseUrl()`.
4. Create a Google OAuth Web Client and run:

   * `initOAuthConfig()`
5. Run:

   * `initGeminiKey()`

### OAuth Requirements

* OAuth Consent Screen must be published to **Production**.
* Redirect URI must match the deployed `/exec` URL.

## Deployment

Deploy as:

* Execute as: Me
* Access: Anyone

Update using new deployment versions to preserve the existing `/exec` URL and avoid breaking embeds.

## Sign-In Experience

The lecturer workflow is:

1. Sign in with Google
2. Consent screen
3. Success page
4. Continue to Master Dashboard

Session tokens are stored in `localStorage`, enabling automatic re-entry on subsequent visits.

### Demo Helpers

* `wipeTestData()`

  * Clears all rows
  * Retains sheet structure
  * Removes uploaded images

* `seedDemoData()`

  * Creates sample courses
  * Creates sample activities
  * Creates sample responses and interactions

---

# 7. Known Limitations

### Multi-Account Google Sessions

Google's serving layer may fail when multiple accounts are signed in simultaneously.

Works correctly in:

* Single-account sessions
* Incognito windows
* Notion embeds

### Browser Storage Restrictions

Some browsers partition or block third-party iframe storage.

The application uses:

* `localStorage`
* In-memory fallback when needed

### GAS Constraints

* No real-time push updates
* Views refresh after actions

### Identity Model

Double-submission and double-vote prevention is based on student number rather than authenticated identity.

This is an intentional trade-off to preserve the no-login student experience.

### Image Delivery

Images are served using:

```text
thumbnail?id=...
```

which is more reliable than:

```text
uc?export=view
```

for shared Drive content.

---

# 8. Security Considerations

### Lecturer Authentication

* OAuth ID token validation
* HMAC-signed session tokens
* Session expiry
* CSRF state validation

### XSS Protection

* User content rendered via `textContent`
* Server-side activity ID sanitisation
* `escapeHtml_()` for generated HTML

### Server-Side Enforcement

* Validation performed on every write
* Client input never trusted
* Ownership verified on every mutation

### Secret Management

Stored in Script Properties:

* Spreadsheet ID
* OAuth credentials
* Session secret
* Gemini API key

### Concurrency

`LockService` protects all read-modify-write operations and prevents lost updates.
