# Telegram Integration Guide for Label-Based Promotion Workflow

## Overview

This document shows exactly what changes are needed to add Telegram notifications to the existing `promote-to-master.yml` workflow. **No new workflow file is needed** — we extend the existing one.

---

## Prerequisites (One-Time Setup)

### Step 1: Create Telegram Bot

1. Open Telegram and search for **@BotFather**
2. Send: `/newbot`
3. Follow prompts:
   - Choose a name: `GitHub Promotion Bot`
   - Choose a username: `your_repo_promotion_bot`
4. Save the **Bot Token** (format: `123456789:ABCdefGHIjklMNOpqrSTUvwxYZ`)

### Step 2: Get Channel Chat ID

1. Add the bot to your Telegram channel as an **administrator**
2. Send a test message in the channel
3. Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
4. Find `chat` → `id` in the response (this is your `TELEGRAM_CHAT_ID`)

### Step 3: Add GitHub Secrets

Go to **Repository → Settings → Secrets and variables → Actions → New repository secret**:

| Secret Name | Value | Description |
|-------------|-------|-------------|
| `TELEGRAM_BOT_TOKEN` | `123456789:ABCdef...` | Bot token from BotFather |
| `TELEGRAM_CHAT_ID` | `-1001234567890` | Channel ID (negative for channels) |

---

## Required Changes to `promote-to-master.yml`

### Change 1: Add Telegram Success Notification

**Location:** After the `Create PR` step (line ~25)

**Add these lines:**

```yaml
      - name: Notify Telegram - Success
        if: steps.cherry.outcome == 'success'
        run: |
          curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
            -H "Content-Type: application/json" \
            -d '{
              "chat_id": "${{ secrets.TELEGRAM_CHAT_ID }}",
              "text": "✅ *PR Promovido Exitosamente*\n\n*Título:* ${{ github.event.pull_request.title }}\n*Autor:* ${{ github.event.pull_request.user.login }}\n*PR:* #${{ github.event.pull_request.number }}\n*PR de Promoción:* [Ver](${{ steps.create_pr.outputs.pr_url }})",
              "parse_mode": "Markdown",
              "disable_web_page_preview": true
            }'
```

### Change 2: Add Telegram Conflict Notification

**Location:** After the existing `Notify conflict` step (line ~35)

**Add these lines:**

```yaml
      - name: Notify Telegram - Conflict
        if: steps.cherry.outcome == 'failure'
        run: |
          curl -s -X POST "https://api.telegram.org/bot${{ secrets.TELEGRAM_BOT_TOKEN }}/sendMessage" \
            -H "Content-Type: application/json" \
            -d '{
              "chat_id": "${{ secrets.TELEGRAM_CHAT_ID }}",
              "text": "⚠️ *Conflicto de Promoción*\n\n*Título:* ${{ github.event.pull_request.title }}\n*Autor:* ${{ github.event.pull_request.user.login }}\n*PR:* #${{ github.event.pull_request.number }}\n\n*Acción Requerida:* Se necesita resolución manual del conflicto",
              "parse_mode": "Markdown",
              "disable_web_page_preview": true
            }'
```

### Change 3: Capture PR URL Output

**Location:** Modify the existing `Create PR` step to capture the URL

**Replace:**

```yaml
      - name: Create PR
        if: steps.cherry.outcome == 'success'
        run: |
          gh pr create --base master --head promote/pr-${{ github.event.pull_request.number }} \
            --title "Promote: ${{ github.event.pull_request.title }}" \
            --body "Auto-promovido desde dev PR #${{ github.event.pull_request.number }}"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**With:**

```yaml
      - name: Create PR
        if: steps.cherry.outcome == 'success'
        id: create_pr
        run: |
          PR_URL=$(gh pr create --base master --head promote/pr-${{ github.event.pull_request.number }} \
            --title "Promote: ${{ github.event.pull_request.title }}" \
            --body "Auto-promovido desde dev PR #${{ github.event.pull_request.number }}")
          echo "pr_url=$PR_URL" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Complete Modified Workflow

See `promote-to-master-with-telegram.yml.example` for the full implementation.

---

## Message Templates

### Success Message
```
✅ PR Promovido Exitosamente

Título: Add new feature
Autor: developer1
PR: #42
PR de Promoción: Ver
```

### Conflict Message
```
⚠️ Conflicto de Promoción

Título: Add new feature
Autor: developer1
PR: #42

Acción Requerida: Se necesita resolución manual del conflicto
```

---

## Testing the Integration

### 1. Test Telegram Connection (Manual)

```bash
export BOT_TOKEN="your_bot_token_here"
export CHAT_ID="your_chat_id_here"

curl -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d "{
    \"chat_id\": \"${CHAT_ID}\",
    \"text\": \"Test notification from GitHub Actions\",
    \"parse_mode\": \"Markdown\"
  }"
```

### 2. Test in Workflow

1. Create a test PR from `feature/test` to `dev`
2. Merge the PR
3. Add the `ready-for-master` label
4. Monitor:
   - GitHub Actions logs
   - Telegram channel for notifications

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Bot not posting | Verify bot is admin in channel |
| "Chat not found" error | Check `TELEGRAM_CHAT_ID` (must be negative for channels) |
| Message not formatted | Ensure `parse_mode: Markdown` is set |
| Rate limit errors | GitHub Actions handles this automatically |
| Secret not accessible | Re-add the secret, check name spelling exactly |

---

## Security Notes

- ✅ Tokens stored in GitHub Secrets (encrypted)
- ✅ Never hardcoded in workflow files
- ✅ Uses environment variables: `${{ secrets.TELEGRAM_BOT_TOKEN }}`
- ✅ No sensitive data in message content

---

## Optional Enhancements

### Add Workflow Run Link

```yaml
"text": "✅ *PR Promovido*\n\n*Título:* ${{ github.event.pull_request.title }}\n*Workflow:* [Ver Ejecución](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})"
```

### Add Labels/Tags to Messages

```yaml
"text": "✅ [PROMOCIÓN] PR #${{ github.event.pull_request.number }} fusionado a master"
```

### Add Author Avatar

```yaml
"text": "✅ *PR Promovido por* ${{ github.event.pull_request.user.login }}\n\n*Título:* ${{ github.event.pull_request.title }}"
```

---

## Summary

| Component | Status | Notes |
|-----------|--------|-------|
| Existing workflow | Keep | No changes to core logic |
| Telegram bot | Create | Via @BotFather (5 min) |
| GitHub Secrets | Add | 2 secrets: token + chat ID |
| Workflow changes | Extend | Add 3 notification steps |
| New files | None | All in existing workflow |
| Testing | Required | Test before production |

**Total changes:** ~30 lines added to existing workflow file.
