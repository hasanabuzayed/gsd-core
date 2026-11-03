# GSD Loop: Schedule

Use this playbook to configure automated recurring execution of the build or review loops.

## 1. Safety Rules

- Run build loops and review loops in separate sessions.
- Do not schedule interactive lanes (`discover` or `spec`).
- Do not run concurrent builder agents against the same repository branch.
- Never force-push or enable automatic merging.

## 2. Configuration Procedure

1. Verify the repository root:
   ```bash
   git rev-parse --show-toplevel
   ```
2. Select one execution lane:
   - `build`: processes ready issues and repair pull requests.
   - `review`: audits open pull requests.
3. Configure recurrence interval:
   - Initial interval: 15 minutes.
   - Idle interval: 60 minutes after three passes without work.
4. Execute one pass per trigger. Require explicit result status:
   - `COMPLETED`: unit pass finished successfully.
   - `IDLE`: no issues or pull requests queued.
   - `BLOCKED`: dependencies, credentials, or merge conflicts require manual intervention.
5. If status is `BLOCKED`, pause scheduling and alert the user.
