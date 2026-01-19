# 🚨 GitHub Status Monitor (with Slack Alerts)

Automate detection of GitHub platform outages and degraded services using **GitHub Actions**.

This workflow periodically checks the official **GitHub Status API**, tracks unresolved incidents across iterations, and sends a Slack alert **only when an incident persists** — eliminating alert noise and preventing blind debugging.

---

## ✨ Why This Exists

Modern CI/CD pipelines depend heavily on GitHub:

* GitHub Actions
* Webhooks
* Self‑hosted runners
* GitOps workflows

When GitHub itself has an outage, engineers often:

* Debug pipelines unnecessarily
* Restart healthy infrastructure
* Waste 20–30 minutes chasing non‑issues

This workflow provides **early operational awareness** so teams know when the platform — not their system — is the root cause.

---

## 🧠 How It Works (High Level)

Every few minutes the workflow:

1. Fetches unresolved incidents from GitHub Status API
2. Stores current incident IDs as state
3. Compares with previous run
4. Alerts only if an incident is still active in the next iteration
5. Updates state for the next cycle

This ensures:

* No alerts for transient glitches
* No repeated spam
* Alerts only for **confirmed platform incidents**

---

## 🧩 Architecture

```
GitHub Actions (cron)
        │
        ▼
GitHub Status API (public)
        │
        ▼
Compare with previous state
        │
        ▼
Slack Webhook Notification
```

---

## 📦 Prerequisites

Before using this workflow, you need:

### 1. Slack Incoming Webhook

Create a Slack app → Enable **Incoming Webhooks** → Copy the webhook URL.

Add it as a GitHub secret:

```
Settings → Secrets → Actions → New Secret
Name: SLACK_WEBHOOK_URL
Value: <your-slack-webhook-url>
```

### 2. GitHub Repository Access

The workflow commits a small state file back to the repository, so:

* Workflow must have write permission
* Repo must allow workflow commits

---

## 🛠️ Installation

Create the workflow file in your repository:

```
.github/workflows/github-status-monitor.yml
```

Paste the following content:

```yaml
name: GitHub Status Monitor

on:
  schedule:
    - cron: '*/5 * * * *'   # Every 5 minutes
  workflow_dispatch:

jobs:
  monitor:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout (to persist incident state)
        uses: actions/checkout@v3

      - name: Check GitHub Status and notify Slack
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        run: |
          set -e

          DATA=$(curl -s https://www.githubstatus.com/api/v2/incidents/unresolved.json)
          COUNT=$(echo "$DATA" | jq '.incidents | length')

          # No incidents → clear state and exit
          if [ "$COUNT" -eq 0 ]; then
            echo "No active incidents. Clearing state."
            rm -f previous_incidents.txt
            exit 0
          fi

          # Store current incident IDs
          echo "$DATA" | jq -r '.incidents[].id' > current_incidents.txt

          # Compare with previous run
          if [ -f previous_incidents.txt ]; then
            PERSISTENT=$(comm -12 <(sort previous_incidents.txt) <(sort current_incidents.txt))
          else
            PERSISTENT=""
          fi

          # Alert only if incidents persist
          if [ -n "$PERSISTENT" ]; then
            MESSAGE="🚨 GitHub Incident Still Active\n\n"

            for ID in $PERSISTENT; do
              INCIDENT=$(echo "$DATA" | jq ".incidents[] | select(.id==\"$ID\")")

              NAME=$(echo "$INCIDENT" | jq -r '.name')
              STATUS=$(echo "$INCIDENT" | jq -r '.status')
              IMPACT=$(echo "$INCIDENT" | jq -r '.impact')

              MESSAGE+="• $NAME — $STATUS ($IMPACT)\n"
            done

            MESSAGE+="\n🔗 https://www.githubstatus.com/"

            curl -X POST \
              -H "Content-Type: application/json" \
              -d "$(jq -n --arg text \"$MESSAGE\" '{text: $text}')" \
              "$SLACK_WEBHOOK_URL"
          else
            echo "Incident detected, but waiting for confirmation in next iteration."
          fi

          # Update baseline for next run
          mv current_incidents.txt previous_incidents.txt

          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add previous_incidents.txt
          git commit -m "Update incident state" || true
          git push || true
```

---

## 🔔 Alert Behavior

### First Detection

* Incident detected
* State stored
* **No alert sent**

### Second Consecutive Detection

* Incident still active
* Slack alert triggered

### Incident Resolved

* State cleared
* No further alerts

This guarantees:

* No flapping alerts
* No duplicate notifications
* Only meaningful signals

---

## 📨 Sample Slack Alert

```
🚨 GitHub Incident Still Active

• GitHub Actions — degraded_performance (medium)
• Webhooks — partial_outage (high)

🔗 https://www.githubstatus.com/
```

---

## ⚙️ Customization

### Change Schedule

Every 10 minutes:

```yaml
cron: '*/10 * * * *'
```

Hourly:

```yaml
cron: '0 * * * *'
```

---

### Monitor Only Specific Services

Filter by component name inside jq:

```bash
jq '.incidents[] | select(.name | test("Actions|Webhooks"))'
```

---

### Send to Microsoft Teams Instead of Slack

Replace webhook payload with Teams JSON format.

---

## 🧠 Best Practices

* Use this in **platform / infra repos**, not application repos
* Keep schedule between **5–10 minutes**
* Store state file in repo (as shown) for reliability
* Pair with internal incident channels

---

## 🛡️ Security Notes

* Uses only **public GitHub Status API**
* No credentials required for API
* Only secret is Slack webhook
* No sensitive data stored

---

## 🚀 Final Thoughts

This workflow doesn’t prevent outages.

It prevents something more expensive:

> **Wasted engineering time debugging healthy systems.**

Operational awareness is often the highest‑ROI automation you can build.

---

## 📌 Author

**Vivek Thirumoorthy**
Lead Azure DevOps / Cloud Architect

Built from real production debugging experience.

---

If this helped you, consider:

* ⭐ Starring the repo
* 🔁 Sharing with your platform team
* 🧵 Opening a PR with improvements
