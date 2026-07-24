---
name: dashboard-screenshots
description: Regenerate the Grafana dashboard PNGs in docs/images/ via grafana-image-renderer. Use when a dashboard JSON changed and its screenshot in docs is now stale.
---

Regenerates the screenshots under `docs/images/*.png`. Background: `docs/grafana-image-renderer-setup.md`.

## Preconditions

1. `imageRenderer.enabled=true` on the `monitoring-extras` release:
   ```bash
   kubectl get deployment/grafana-image-renderer -n monitoring
   # if missing/not enabled:
   helm upgrade --install monitoring-extras ./charts/monitoring-extras \
     --namespace monitoring --values charts/monitoring-extras/values.yaml \
     --set imageRenderer.enabled=true --reset-values
   kubectl rollout status deployment/grafana-image-renderer -n monitoring --timeout=180s
   ```
2. Port-forward Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/vm-grafana 3000:80 &
   ```

## Per-dashboard steps

1. Find the dashboard's `uid`/`slug` (don't hardcode — dashboards get re-provisioned with new UIDs
   occasionally):
   ```bash
   curl -s -u admin:admin "http://localhost:3000/api/search?query=<title or partial title>"
   ```
2. Render it, matching the existing filename in `docs/images/` for that dashboard (check `ls
   docs/images/` for the current name — don't invent a new one):
   ```bash
   curl -s -o docs/images/<existing-filename>.png -u admin:admin \
     "http://localhost:3000/render/d/<uid>/<slug>?width=1600&height=1000&from=now-24h&to=now&kiosk=tv"
   ```
   Adjust `width`/`height` to whatever avoids excessive whitespace or panel cutoff for that specific
   dashboard (existing files vary — e.g. wider for multi-column, taller for stacked panels).
3. Verify it's an actual image, not the renderer's plaintext error body:
   ```bash
   file docs/images/<existing-filename>.png   # must say "PNG image data", not "ASCII text"
   ```
4. If a panel shows "No data" unexpectedly, check the renderer's own logs (it captures the headless
   Chromium console, including the exact query that actually ran) rather than guessing at query
   params:
   ```bash
   kubectl logs -n monitoring -l app=grafana-image-renderer --tail=200 | grep -i "500\|error"
   ```

## After

Kill the port-forward. If dashboard JSON itself changed as part of this session, mention it in the
commit alongside the screenshot update — per CLAUDE.md, one logical change per commit, but a
dashboard fix and its regenerated screenshot belong together.
