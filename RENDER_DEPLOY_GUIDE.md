# Deploy OmniFind on Render

## Files
This archive is arranged so that `app.py`, `requirements.txt`, `render.yaml`, and
`OmniFind_Frontend/` are at the repository root. Local `.env` secrets and the
local SQLite database were intentionally excluded.

## GitHub
1. Extract this ZIP.
2. Create/open your GitHub repository.
3. Upload the *contents* of the `OmniFind_Render_Ready` folder (not an extra
   parent folder) to the repository root.
4. Commit and push.

## Render
1. In Render, choose **New + → Blueprint** if available and select this repository;
   Render will read `render.yaml`. Alternatively create a **Web Service** and set:
   - Runtime: Python
   - Build command: `pip install -r requirements.txt`
   - Start command: `gunicorn app:app --bind 0.0.0.0:$PORT --workers 1 --threads 4 --timeout 120`
   - Root Directory: leave blank if these files are in the repository root.
2. In the service's Environment settings, add `OPENAI_API_KEY` with your own API key.
   Do not put the key in GitHub, source files, screenshots, or this ZIP.
3. Keep `AI_PROVIDER=openai`, `OPENAI_MODEL=gpt-4o-mini`, and `COOKIE_SECURE=1`.
4. Deploy and check the Logs if the build fails.

## Notes
- OpenAI API usage may incur separate charges. A ChatGPT subscription does not include
  API usage.
- Without `OPENAI_API_KEY`, the site may load but AI Answer Chat will not return AI answers.
- The service creates its SQLite database on startup. On Render's free ephemeral disk,
  local account data can be lost when the instance is replaced/restarted. Use a persistent
  database for production.
- Email verification and password reset email delivery require SMTP environment variables.
- Never upload `.env`, API keys, or a local database containing user information to GitHub.
