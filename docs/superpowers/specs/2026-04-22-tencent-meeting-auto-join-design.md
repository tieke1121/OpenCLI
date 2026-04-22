# Tencent Meeting Auto Join Management Platform Design

## Background

The target system is a backend management platform for automatically joining Tencent Meeting sessions using managed accounts. An operator enters a Tencent Meeting ID and the desired number of attendees. The platform selects account credentials from a database, starts isolated browser environments, and uses OpenCLI automation to log in and join the meeting through the Tencent Meeting web interface.

This design assumes Tencent Meeting web login supports mobile phone number plus password login. Captcha, SMS verification, and other risk-control challenges are treated as explicit failure states for the first version, not as flows to bypass.

## Goals

- Let an operator create a meeting join task with meeting ID and attendee count.
- Select available accounts from a managed account pool.
- Use each account's mobile phone number and password to join Tencent Meeting through the web UI.
- Run each account in an isolated Chrome profile or browser instance to avoid login-state collisions.
- Record per-account execution status, failure reason, and diagnostic evidence.
- Keep OpenCLI responsible for single-account browser automation, while the backend platform owns scheduling, persistence, and concurrency.

## Non-Goals For The First Version

- Automatically solving captcha, SMS verification, slider verification, or device-risk challenges.
- Long-running in-meeting monitoring after a successful join.
- Audio/video participation beyond default mute/camera-off preparation.
- Distributed multi-machine scheduling.
- Official Tencent Meeting API integration unless it becomes available and appropriate.
- Reusing one browser profile across multiple accounts.

## Existing OpenCLI Fit

OpenCLI already provides the right lower-level automation model:

- Site commands are implemented as adapters under `clis/<site>/<command>.js`.
- `cli({...})` registers a command with arguments, browser strategy, output columns, and a handler.
- Browser commands use Chrome/Chromium automation through Browser Bridge or CDP.
- Existing adapters already demonstrate UI automation and account/password login patterns.

For this requirement, OpenCLI should expose a single-account capability such as:

```bash
OPENCLI_CDP_ENDPOINT=http://127.0.0.1:9222 \
opencli tencent-meeting join \
  --meeting-id 123456789 \
  --phone 13800000000 \
  --password '***' \
  --nickname 'attendee-001'
```

The backend platform should call this command once per selected account, with a different browser endpoint/profile per account.

## Architecture

### Components

1. Backend management platform
   - Provides task creation APIs and admin UI.
   - Stores accounts, tasks, task items, browser profile metadata, and execution logs.
   - Dispatches join jobs with bounded concurrency.

2. Account pool
   - Stores encrypted phone/password credentials.
   - Tracks availability, last used time, failure counts, cooldown state, and verification-required state.

3. Browser instance manager
   - Creates and reuses per-account Chrome profile directories.
   - Starts one Chrome instance per active account with a unique remote debugging port.
   - Stops idle browser instances and releases ports.

4. OpenCLI Tencent Meeting adapter
   - Implements `tencent-meeting join`.
   - Automates one account joining one meeting.
   - Returns structured status and failure details.

5. Task executor
   - Pulls pending task items.
   - Starts browser instance for the account.
   - Calls OpenCLI with the account-specific CDP endpoint.
   - Updates task item status based on OpenCLI output and process result.

### Runtime Isolation

The accepted isolation model is:

- One account maps to one persistent Chrome profile directory.
- Each active profile runs in its own Chrome process or browser instance.
- Each browser instance exposes a unique CDP port.
- OpenCLI connects to the target account browser using `OPENCLI_CDP_ENDPOINT`.

This avoids cookie/session contamination between accounts and makes failure recovery simpler.

Example launch shape:

```bash
chrome \
  --user-data-dir=/data/tencent-meeting/profiles/account-123 \
  --remote-debugging-port=9301 \
  --no-first-run \
  --disable-default-apps
```

## Data Model

### accounts

Stores managed Tencent Meeting accounts.

- `id`
- `phone`
- `password_encrypted`
- `status`: `available`, `in_use`, `cooldown`, `disabled`, `verification_required`
- `profile_dir`
- `last_used_at`
- `last_success_at`
- `last_failure_at`
- `failure_count`
- `created_at`
- `updated_at`

### meeting_tasks

Stores operator-created meeting join tasks.

- `id`
- `meeting_id`
- `requested_attendees`
- `concurrency_limit`
- `nickname_prefix`
- `status`: `pending`, `running`, `completed`, `partial_failed`, `failed`, `cancelled`
- `created_by`
- `created_at`
- `started_at`
- `finished_at`

### meeting_task_items

Stores one account execution under a task.

- `id`
- `task_id`
- `account_id`
- `meeting_id`
- `nickname`
- `status`: `pending`, `starting_browser`, `logging_in`, `joining`, `joined`, `failed`, `need_verification`, `waiting_host_approval`, `cancelled`
- `attempt_count`
- `browser_port`
- `opencli_exit_code`
- `failure_code`
- `failure_message`
- `screenshot_path`
- `log_path`
- `started_at`
- `finished_at`

### browser_sessions

Tracks running browser instances.

- `id`
- `account_id`
- `profile_dir`
- `cdp_endpoint`
- `port`
- `pid`
- `status`: `starting`, `ready`, `busy`, `idle`, `stopped`, `failed`
- `last_heartbeat_at`
- `created_at`
- `updated_at`

## Task State Flow

Task-level state:

```text
pending -> running -> completed
                 \-> partial_failed
                 \-> failed
                 \-> cancelled
```

Task-item state:

```text
pending
  -> starting_browser
  -> logging_in
  -> joining
  -> joined
```

Failure branches:

```text
logging_in -> need_verification
logging_in -> failed
joining -> waiting_host_approval
joining -> failed
starting_browser -> failed
```

`waiting_host_approval` can be treated as success-like or pending-like depending on business rules. For the first version, it should be recorded separately because the account reached the meeting lobby but may not be fully admitted.

## OpenCLI Adapter Contract

### Command

```bash
opencli tencent-meeting join \
  --meeting-id <meetingId> \
  --phone <phone> \
  --password <password> \
  --nickname <nickname>
```

### Arguments

- `meeting-id`: required Tencent Meeting ID.
- `phone`: required account phone number.
- `password`: required account password.
- `nickname`: optional display name.
- `mute`: optional boolean, default `true`.
- `camera-off`: optional boolean, default `true`.

### Output

The adapter should return one row with stable columns:

- `status`
- `meetingId`
- `phone`
- `nickname`
- `message`
- `failureCode`

Example success:

```json
[
  {
    "status": "joined",
    "meetingId": "123456789",
    "phone": "138****0000",
    "nickname": "attendee-001",
    "message": "Joined meeting",
    "failureCode": ""
  }
]
```

Example verification failure:

```json
[
  {
    "status": "need_verification",
    "meetingId": "123456789",
    "phone": "138****0000",
    "nickname": "attendee-001",
    "message": "Login requires additional verification",
    "failureCode": "VERIFICATION_REQUIRED"
  }
]
```

## Adapter Flow

The first implementation should automate this single-account sequence:

1. Open Tencent Meeting web join/login page.
2. Detect whether the current browser profile is already logged in.
3. If not logged in:
   - Choose phone/password login.
   - Fill phone and password.
   - Submit login.
   - Detect login success or verification requirement.
4. Navigate to meeting join page if needed.
5. Enter meeting ID.
6. Enter nickname if required.
7. Set microphone muted and camera off where possible.
8. Click join.
9. Detect final state:
   - Joined meeting.
   - Waiting for host approval.
   - Meeting not found.
   - Meeting full.
   - Password or account error.
   - Verification required.
   - Unknown page state.

The adapter should use DOM selectors only after live reconnaissance confirms the current Tencent Meeting web UI. Selectors should be defensive and return structured failure states rather than hanging indefinitely.

## Backend Execution Flow

1. Operator creates a task.
2. Backend validates `meeting_id`, `requested_attendees`, and `concurrency_limit`.
3. Backend locks available accounts.
4. Backend creates one task item per account.
5. Worker loop processes task items up to the concurrency limit.
6. For each task item:
   - Start or reuse the account's Chrome profile.
   - Wait for CDP endpoint readiness.
   - Invoke OpenCLI with `OPENCLI_CDP_ENDPOINT`.
   - Parse JSON output.
   - Save logs and screenshots on failure.
   - Update account and task item status.
7. When all task items reach terminal states, update task status.

## Error Handling

Recommended failure codes:

- `BROWSER_START_FAILED`
- `CDP_NOT_READY`
- `LOGIN_FAILED`
- `VERIFICATION_REQUIRED`
- `MEETING_NOT_FOUND`
- `MEETING_FULL`
- `WAITING_HOST_APPROVAL`
- `JOIN_BUTTON_NOT_FOUND`
- `JOIN_TIMEOUT`
- `UNKNOWN_PAGE_STATE`
- `OPENCLI_FAILED`

Retry policy:

- Browser start/CDP failures may retry 1-2 times.
- Selector or unknown page state failures should capture screenshot and log, then stop.
- Verification-required accounts should be marked `verification_required` and removed from the available pool.
- Password/login failures should increment account failure count and eventually disable the account.

## Security Requirements

- Store account passwords encrypted at rest.
- Do not print raw passwords in logs, process summaries, or OpenCLI output.
- Mask phone numbers in user-facing output.
- Restrict credential decryption to the worker process.
- Keep per-account profile directories access-controlled.
- Record operator identity for task creation and cancellation.
- Avoid storing screenshots if they contain sensitive information unless retention policy allows it.

## Operational Constraints

- Set a conservative concurrency limit in the first version.
- Keep a cooldown between repeated logins for the same account.
- Avoid deleting profile directories after every task; reuse profiles to reduce login frequency.
- Provide manual reset for accounts stuck in `verification_required` or `disabled`.
- Add periodic cleanup for stopped browser processes and stale CDP ports.

## MVP Plan

### Phase 1: Single Account Verification

- Connect Browser Bridge or use a dedicated Chrome CDP endpoint.
- Recon Tencent Meeting web login and join pages.
- Implement local `tencent-meeting join` adapter.
- Verify one test account can log in and join one test meeting.
- Identify real success/failure DOM states.

### Phase 2: Backend Scheduler MVP

- Add account, task, task item, and browser session tables.
- Implement Chrome profile manager.
- Implement OpenCLI command runner.
- Implement task-item state updates.
- Run small batch tasks with low concurrency.

### Phase 3: Hardening

- Add screenshots and structured logs.
- Add retry/cooldown rules.
- Add account health states.
- Add admin UI for task progress and failures.
- Add cleanup for stale browsers and profiles.

## Current Environment Blocker

On this machine, `npm run dev -- doctor` reports:

```text
[OK] Daemon: running on port 19825
[MISSING] Extension: not connected
[FAIL] Connectivity: Browser Bridge extension not connected
```

Tencent Meeting live reconnaissance requires either:

- Loading the repository `extension/` directory into Chrome, or
- Installing the released OpenCLI Browser Bridge extension, or
- Using an independently launched Chrome with a CDP endpoint.

Until browser connectivity is available, implementation can proceed only up to static adapter scaffolding and backend design.

