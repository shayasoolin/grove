# Grove Scale Test History

This branch stores public Grove scale-test history and the static dashboard served by GitHub Pages.

- `index/metrics.ndjson` is the compact metric index used by the dashboard.
- `results/` contains the raw `scale-test-results.json` files for each stored run.
- `index.html`, `app.js`, and `style.css` render the dashboard.

This branch is updated by the scale-test GitHub Actions workflow. It is not intended for regular user or pull-request edits.
