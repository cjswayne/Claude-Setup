# Flag User Action: ALL CAPS + Windows Notification

The user often has the Claude tab open while looking at something else. When you need them, make it impossible to miss.

## When this applies

- You need the user to do something you can't: log in to a site in the browser, approve an OAuth/SSO screen, solve a CAPTCHA, enter a password or payment details, plug something in, restart an app, etc.
- You are asking the user a question and are waiting on the answer (including AskUserQuestion prompts and permission-style "should I proceed?" questions).

## Required behavior

1. **Write the request in ALL CAPS** in your chat response so it stands out, e.g. `PLEASE LOG IN TO SHOPIFY IN THE BROWSER PANE, THEN TELL ME WHEN YOU'RE DONE.`
2. **Send a Windows toast notification right before you stop and wait.** Run it from the Bash tool:

   ```bash
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File "$HOME/.claude/rules/scripts/notify-user.ps1" -Message "Log in to Shopify in the browser pane"
   ```

   Or from the PowerShell tool:

   ```powershell
   & "$env:USERPROFILE\.claude\rules\scripts\notify-user.ps1" -Message "Log in to Shopify in the browser pane"
   ```

   - Keep `-Message` short (under ~80 chars) and say what is needed. Normal case is fine in the toast.
   - Optional `-Title` overrides the default "Claude needs you".
   - Never put secrets, tokens, or personal data in the message.
3. Send **one** toast per wait point. Don't toast for routine end-of-task summaries where nothing is needed from the user.
4. If the toast command fails, mention it briefly and continue; still use ALL CAPS.
