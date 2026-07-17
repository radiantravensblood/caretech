# Care-Tech Synthetic Prototype

This package contains a static concept demonstration for a consent-directed care companion.

## Files

- `index.html` — public landing page
- `dashboard.html` — clinician-facing synthetic dashboard
- `companion.html` — scripted client-facing interaction preview
- `ARCHITECTURE.md` — production implementation and trust-boundary notes

## Run locally

Open `index.html` directly, or serve the folder with a basic local web server:

```bash
python3 -m http.server 8080
```

Then visit `http://localhost:8080`.

## Prototype limits

- synthetic data only;
- no conversational model connected;
- no backend or database;
- no real clinician or patient connection;
- no alert, crisis-line, or emergency integration;
- not for clinical use.

The scripted companion exists only to demonstrate privacy state, summary sharing, provenance, and a simulated safety-plan event.
