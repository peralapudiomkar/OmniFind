# OmniFind: Enhanced Search Platform

OmniFind is a Flask-based search prototype. It pulls records from public APIs, ranks them with a hybrid scoring method, supports place and GPS search, exports results, accepts file uploads, and includes an AI answer chat that cites its sources. It also has account registration and login.

> **Status: prototype.** OmniFind is not a full-web crawler or a Google-scale index. Coverage depends on the public APIs it queries.

## Table of Contents

1. [Requirements](#requirements)
2. [Installation and First Run](#installation-and-first-run)
3. [Using the App](#using-the-app)
4. [How Search Ranking Works](#how-search-ranking-works)
5. [AI Answer Chat](#ai-answer-chat)
6. [Accounts and Authentication](#accounts-and-authentication)
7. [Configuration Reference](#configuration-reference)
8. [Security Notes](#security-notes)
9. [Known Limitations](#known-limitations)
10. [Troubleshooting](#troubleshooting)
11. [Moving to Production](#moving-to-production)

---

## Requirements

- Python 3 (the commands below use the Windows `py` launcher; on macOS or Linux use `python3`)
- An internet connection, since search, geocoding, and AI chat call external APIs
- An OpenAI API key, for the AI Answer Chat only
- An SMTP account, for real email verification and password reset only

## Installation and First Run

```bash
py -m pip install -r requirements.txt
py app.py
```

Then open <http://127.0.0.1:5000/login>, register an account or sign in, and start searching.

`hnswlib` is an optional dependency. If it fails to install, the app still runs and falls back to a diagnostic graph (see [Known Limitations](#known-limitations)).

## Using the App

### Searching
Enter a query and run the search. Results are ranked by the hybrid method described below. Search, graph, geocode, and ingest APIs all require you to be signed in.

### Places / GPS search
The GPS button appears only after you choose **Places / GPS**. Place queries are geocoded live through Nominatim, and results open Google Maps directions.

### Exporting results
Use the **JSON** or **CSV** download buttons to export the results of the current search. The export contains only the results returned for that search, not every record that exists anywhere.

### Uploading files
Open the file ingest modal and either browse for a file or drag and drop it. Each upload request returns an indexing status. Uploaded content is held in server memory (see [Known Limitations](#known-limitations)).

## How Search Ranking Works

Each result gets a combined score built from five signals:

| Signal | What it captures |
|---|---|
| Title match | Query terms appearing in the title |
| Phrase match | The query appearing as a phrase |
| Keyword coverage | How many of the query's keywords are present |
| BM25-inspired score | Term frequency and rarity, normalized for length |
| Feature-hash similarity | Lexical similarity from hashed features |

Feature hashing measures lexical similarity. It is **not** a trained semantic model, so it will not understand synonyms or meaning the way an embedding model does.

## AI Answer Chat

The **AI Answer Chat** button is in the top navigation. For each question:

1. The Flask backend retrieves relevant results from the configured public APIs.
2. It sends compact source context to the OpenAI API.
3. It returns an answer together with source links.

API keys stay on the backend and are never embedded in browser JavaScript.

### Setup

1. Copy `.env.example` to `.env` in the same folder as `app.py`.
2. Set `OPENAI_API_KEY` to your own key.
3. Optionally set `OPENAI_MODEL` (default: `gpt-4o-mini`) and a persistent random `SECRET_KEY`.
4. Restart with `py app.py`.

### Behavior to expect

- If the key is missing, the chat shows a setup message rather than inventing an answer.
- If a provider fails or returns no results, the UI reports that openly.
- Answers are based only on the retrieved records, so verify anything important against the linked sources.
- OpenAI API access and billing are separate from a ChatGPT subscription.

## Accounts and Authentication

- Registration and login are backed by SQLite.
- Passwords are stored as salted hashes.
- Sessions are signed with `SECRET_KEY`.
- Logout, a forgot-password endpoint, reset tokens, and optional email verification are included.
- Email syntax is always checked. Ownership is confirmed only after the verification link is sent and clicked.

**Without SMTP configured**, registration creates a local development account, but the app cannot verify inbox ownership or deliver reset emails. Configure SMTP before any production use.

## Configuration Reference

Set these as environment variables, or in `.env` for local development.

| Variable | Required for | Notes |
|---|---|---|
| `OPENAI_API_KEY` | AI Answer Chat | Your own key; keep it private |
| `OPENAI_MODEL` | AI Answer Chat (optional) | Defaults to `gpt-4o-mini` |
| `SECRET_KEY` | Sessions and tokens | Long, random, and persistent across restarts |
| `SMTP_HOST` | Email features | Your provider's SMTP host |
| `SMTP_PORT` | Email features | Usually `587` |
| `SMTP_USERNAME` | Email features | SMTP login |
| `SMTP_PASSWORD` | Email features | Use a provider or app password, not your everyday password |
| `SMTP_FROM` | Email features | Verified sender address |
| `COOKIE_SECURE` | HTTPS deployments | Set to `1` behind HTTPS; leave unset or `0` on localhost HTTP |

## Security Notes

- Never commit `.env`, API keys, or SMTP credentials to GitHub.
- Check `.gitignore` before every commit to confirm `.env` is excluded.
- Use a persistent random `SECRET_KEY`. If it changes, existing sessions and tokens become invalid.
- Set `COOKIE_SECURE=1` whenever the app is served over HTTPS.
- If a key is ever committed by mistake, revoke it and create a new one. Deleting the file in a later commit does not remove it from history.

## Known Limitations

- **Not a full-web crawler.** Source coverage depends on the selected public APIs, which have rate limits and may fail intermittently.
- **HNSW is conditional.** It is truly enabled only when `hnswlib` installs successfully. Otherwise the fallback graph is diagnostic only.
- **In-memory storage.** Uploaded files and graph vectors are held in server memory and are lost on restart.
- **Export scope.** Exports cover the current search's returned results only.
- **Lexical, not semantic.** Feature-hash similarity does not understand meaning.
- **AI answers can be incomplete or wrong.** They are limited to the retrieved context.

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| `py` is not recognized | Install Python from python.org and tick the launcher option, or use `python` / `python3` instead |
| `hnswlib` fails to install | It often needs a C++ build toolchain. The app still runs with the fallback graph |
| Chat shows a setup message | `OPENAI_API_KEY` is missing or `.env` is not next to `app.py`. Fix it and restart |
| Chat errors even with a key | Check the key, your OpenAI usage limits, and billing, which are separate from ChatGPT |
| Logged out after every restart | `SECRET_KEY` is not persistent. Set a fixed value |
| No verification or reset emails | SMTP variables are missing or wrong. Check host, port, credentials, and `SMTP_FROM` |
| Login cookie not working on HTTPS | Set `COOKIE_SECURE=1` |
| Place search returns nothing or fails | Nominatim may be rate-limiting or temporarily down. Wait and retry |
| Uploaded files disappear | Uploads are in memory and reset on restart |

## Moving to Production

Before deploying publicly:

- Persist uploads and indexes in a database or object store.
- Configure SMTP and set `COOKIE_SECURE=1` over HTTPS.
- Set a strong persistent `SECRET_KEY` through your platform's secret manager.
- Run behind a production WSGI server instead of the built-in development server.
- Plan for public API rate limits and failures, for example with caching and retries.
