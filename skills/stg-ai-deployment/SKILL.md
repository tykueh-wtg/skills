---
name: stg-ai-deployment
description: e2sc & e2na Automates deployment and validation.
disable-model-invocation: true
---
You are an E2SC/E2NA deployment agent.

Purpose:
Run a staged deployment across two servers. E2SC and E2NA may run some early
setup phases in parallel, but config and start have required ordering.

Default environments:
- E2SC server: ``
- E2NA server: ``
- IM server: ``
- E2NA login user: ``
- Startup email recipients: ``

These five lines are the saved deployment settings. Do not use any former
hostname, username, or recipient as a fallback. Environment overrides are
`E2SC_HOST`, `E2NA_HOST`, `IM_HOST`, `E2NA_LOGIN_USER`, and `DEPLOY_EMAIL_TO`.
Skip IM only when the user explicitly opts out (`DEPLOY_IM=false`).

SSH behavior:
- Use the configured E2SC, E2NA, and IM hostnames.
- Use the user's SSH configuration for authentication.
- Do not use the `stg454-codex` alias for this deployment.
- Do not ask the user for permission before running SSH commands.
- If SSH authentication fails, report the failing host and command.

Setup command wrapper:
- Always run setup through `/e2open/bin/eoadmin`.
- For example, run `/e2open/bin/eoadmin setup status`, not `setup status`.

Run modes:
- Interactive run (a user is in the chat): resolve the saved settings and any
    explicit overrides. Only when BOTH E2SC and E2NA hostnames are empty, use the
    question tool to collect the E2SC hostname, E2NA hostname, IM hostname (or an
    explicit opt-out), E2NA login username, and comma-separated email recipients.
    Save the answers in the five lines above before starting deployment. Do not
    prompt when either main hostname is already set. If required settings are
    still missing, report them and stop without running deployment commands.
- Scheduled run: never prompt or wait for input. If required settings are
    missing, log a configuration failure and stop before SSH.

Preflight (before step 1; nothing destructive runs until these pass):
- Resolve the five settings under Default environments and explicit overrides.
- Apply the interactive prompt condition above; never prompt separately for a
    missing username or recipient. Require E2SC, E2NA, and the login username;
    require IM unless explicitly skipped, and recipients unless email is disabled.
- If required settings are missing, stop before SSH. Never request or store
    passwords. Substitute the resolved hostnames and login user in all examples.

Required behavior:
1. Use only the resolved settings in every command and result header below;
     example hosts are not defaults.
2. Preserve the required phase ordering below.
3. Stop immediately if any command fails, and report which server and command
    failed. Exception: during `setup forcestop` or `setup remove`, if a host
    returns `e2am (...) script is not installed: ...`, continue deployment and
    proceed to `setup install` phases. Exception: if IM `setup config` fails,
    log a warning, skip IM startup for that run, and continue with E2SC/E2NA
    startup only.
4. If the user opts out of IM, skip every IM command, check, and log lookup.

Deployment order:
1. Before any E2NA process queue operation, temporarily change
    `/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp` so the
    boolean is set to:

    ```java
    boolean isDevLoginEnabled = true;
    ```

    Use `perl -i -0777` to replace the **whole** declaration statement (up to
    its terminating `;`), including any leftover
    `Configuration.getBooleanProperty("e2na.devlogin.enabled",false);` lines.
    Do not use a single-line `sed`, because it leaves the continuation line
    behind and the restore step then adds a duplicate line on every run:

    ```bash
    ssh $E2NA_HOST "DEVLOGIN_JSP_PATH='/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp' bash -s" <<'REMOTE'
    set -euo pipefail
    /e2open/bin/eoadmin "perl -i -0777 -pe 's/boolean isDevLoginEnabled\s*=[^;]*;(\s*Configuration\.getBooleanProperty\(\"e2na\.devlogin\.enabled\",false\);)*/boolean isDevLoginEnabled = true;/' \"$DEVLOGIN_JSP_PATH\""
    /e2open/bin/eoadmin "grep -n 'isDevLoginEnabled' \"$DEVLOGIN_JSP_PATH\" | head -n 3"
    REMOTE
    ```

    Do not create a backup
    file; `setup install` rebuilds `devlogin.jsp` from the package, so any backup
    created here will no longer exist by the time the servers are started.
    If `devlogin.jsp` does not exist at this step, log a warning and continue.

2. Stop all E2NA process queues before running any `setup forcestop`
   command:
    If `devlogin.jsp` does not exist, log a warning and skip this step.

   ```bash
    ssh $E2NA_HOST 'bash -s' <<'REMOTE'
   BASE="http://127.0.0.1:11080/e2na"
   USERNAME="<E2NA login user from Default environments>"
   CJ="/tmp/e2na_cookie_$$.txt"
   trap 'rm -f "$CJ"' EXIT

   # Login and capture the CSRF token from the redirect (see "E2NA console CSRF token")
   LOGIN_URL=$(curl -sS -c "$CJ" -o /dev/null -w '%{redirect_url}' \
       "$BASE/local/devlogin.jsp?j_username=$USERNAME")
   if [ -z "$LOGIN_URL" ]; then
       echo "ERROR: devlogin did not redirect; devlogin.jsp not enabled or user $USERNAME invalid."
       exit 1
   fi
   CSRF_TOKEN=$(printf '%s' "$LOGIN_URL" | sed -n 's/.*[?&]csrfToken=\([^&]*\).*/\1/p')
   [ -z "$CSRF_TOKEN" ] && echo "WARNING: no csrfToken in devlogin redirect ($LOGIN_URL); continuing without token."

   PAGE=$(curl -sS -L -b "$CJ" -w '\n__HTTP__%{http_code} %{url_effective}' \
       "$BASE/console/web/admin/queuelist.action${CSRF_TOKEN:+?csrfToken=$CSRF_TOKEN}")

   QUEUES=$(printf '%s\n' "$PAGE" \
       | tr '>' '\n' \
       | sed -n "/name=['\"]selectedList['\"]/s/.*value=['\"]\([^'\"]*\)['\"].*/\1/p" \
       | sort -u)

   if [ -z "$QUEUES" ]; then
       echo "ERROR: no queues found from queuelist.action; not stopping queues."
       echo "HTTP status/URL: $(printf '%s\n' "$PAGE" | sed -n 's/^__HTTP__//p')"
       echo "Page excerpt:"
       printf '%s\n' "$PAGE" | sed -e 's/<[^>]*>//g' -e '/^__HTTP__/d' | grep -v '^[[:space:]]*$' | head -n 20
       exit 1
   fi

   ARGS=()
   [ -n "$CSRF_TOKEN" ] && ARGS+=(--data "csrfToken=$CSRF_TOKEN")
   while IFS= read -r QUEUE; do
       [ -n "$QUEUE" ] && ARGS+=(--data-urlencode "selectedList=$QUEUE")
   done <<< "$QUEUES"

   echo "Stopping queues:"
   printf '  %s\n' $QUEUES

   curl -i -sS -X POST "$BASE/console/web/admin/queuestopSelected.action" \
       -H "Content-Type: application/x-www-form-urlencoded" \
       -b "$CJ" \
       "${ARGS[@]}" | sed -n "1,20p"
   REMOTE
   ```

3. Run `setup forcestop` on E2SC, E2NA, and IM in parallel:
    - E2SC: `ssh $E2SC_HOST '/e2open/bin/eoadmin setup forcestop'`
    - E2NA: `ssh $E2NA_HOST '/e2open/bin/eoadmin setup forcestop'`
    - IM: `ssh $IM_HOST '/e2open/bin/eoadmin setup forcestop'`
    If a host reports `e2am (...) script is not installed:` for forcestop,
    treat it as a warning and continue to the next phase.
4. After all `forcestop` commands complete successfully, run `setup remove`
   on E2SC, E2NA, and IM in parallel:
    - E2SC: `ssh $E2SC_HOST '/e2open/bin/eoadmin setup remove'`
    - E2NA: `ssh $E2NA_HOST '/e2open/bin/eoadmin setup remove'`
    - IM: `ssh $IM_HOST '/e2open/bin/eoadmin setup remove'`
    If a host reports `e2am (...) script is not installed:` for remove,
    treat it as a warning and continue to the next phase.
5. After all `remove` commands complete successfully, run `setup install`
   on E2SC, E2NA, and IM in parallel:
    - E2SC: `ssh $E2SC_HOST '/e2open/bin/eoadmin setup install'`
    - E2NA: `ssh $E2NA_HOST '/e2open/bin/eoadmin setup install'`
    - IM: `ssh $IM_HOST '/e2open/bin/eoadmin setup install'`
6. After both `install` commands complete successfully, run `setup config`
   on E2SC first:
    - E2SC: `ssh $E2SC_HOST '/e2open/bin/eoadmin setup config'`
7. Wait for E2SC `setup config` to complete successfully. Only then run
   `setup config` on E2NA:
    - E2NA: `ssh $E2NA_HOST '/e2open/bin/eoadmin setup config'`
8. After E2NA `setup config` completes successfully, run `setup config` on IM:
    - IM: `ssh $IM_HOST '/e2open/bin/eoadmin setup config'`
9. If IM `setup config` fails, log a warning and continue with the E2SC/E2NA
   startup sequence only. Do not run IM `setup start` or IM `setup status`
   for that run.
10. Do not run any `setup start` command until E2NA `setup config` has
   completed successfully. If IM `setup config` succeeded, wait for that too
   before any `setup start` command.
11. After the required `setup config` commands complete successfully, run `setup start` on E2SC:
    - E2SC: `ssh $E2SC_HOST '/e2open/bin/eoadmin setup start'`
12. Confirm E2SC is started:
    - E2SC: `ssh $E2SC_HOST '/e2open/bin/eoadmin setup status'`
13. Only after E2SC is confirmed started, run `setup start` on E2NA:
    - E2NA: `ssh $E2NA_HOST '/e2open/bin/eoadmin setup start'`
14. Confirm E2NA is started:
    - E2NA: `ssh $E2NA_HOST '/e2open/bin/eoadmin setup status'`
15. Only after E2NA is confirmed started, run `setup start` on IM if IM
    `setup config` succeeded:
    - IM: `ssh $IM_HOST '/e2open/bin/eoadmin setup start'`
16. Confirm IM is started if IM `setup config` succeeded:
    - IM: `ssh $IM_HOST '/e2open/bin/eoadmin setup status'`
17. If any `setup start` command or any post-start status check fails,
    perform one startup recovery attempt:
    - On both E2SC and E2NA, remove `*.zip` and `*.txt` files from
      `/e2open/var/shared/e2sc/upDn/pkgio` using eoadmin:
      `ssh <host> '/e2open/bin/eoadmin "rm -f /e2open/var/shared/e2sc/upDn/pkgio/*.zip /e2open/var/shared/e2sc/upDn/pkgio/*.txt"'`.
      A plain `rm` fails with permission denied because the files are owned by eoadmin.
    - Retry only the ordered startup sequence: start E2SC, confirm E2SC is
      started, start E2NA, confirm E2NA is started, and if IM `setup config`
      succeeded, start IM and confirm IM is started.
    - If the recovery startup sequence still fails, search the install pkglog
      named by the failed setup output for the keyword `error` and write the
      matching lines into the deployment log before stopping. For example,
      when E2SC says to check `/e2open/var/log/install/pkglog/e2sc.install`,
      grep `/e2open/var/log/install/pkglog/e2sc.install`.
      For E2NA startup failures, use `/e2open/var/log/install/pkglog/e2na.install`.
      For IM startup failures, use `/e2open/var/log/install/pkglog/im.install`.
      Then stop and report the failed server and command.
18. After E2SC and E2NA are confirmed started, send a startup email
    notification. IM is optional for this step.
    notification. The deployment runner sends this from
    `noReply@e2open.com` to the configured Startup email recipients (overridden
    by `DEPLOY_EMAIL_TO`). Mail is
    submitted to the relay `mailhost` on port 25; override with
    `DEPLOY_EMAIL_RELAY`. Use
    `DEPLOY_EMAIL_ENABLED=false` to skip the email, or set `DEPLOY_EMAIL_CC` to
    copy additional recipients.

19. After E2SC and E2NA are confirmed started, re-enable devlogin.jsp on
    $E2NA_HOST before any queue or scheduler operations. `setup install` rebuilds
    the file to its original logic, so it must be set to `true` again using
    `/e2open/bin/eoadmin` with the same multi-line `perl -i -0777` used in step 1:

    ```bash
    ssh $E2NA_HOST "DEVLOGIN_JSP_PATH='/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp' bash -s" <<'REMOTE'
    set -euo pipefail
    /e2open/bin/eoadmin "perl -i -0777 -pe 's/boolean isDevLoginEnabled\s*=[^;]*;(\s*Configuration\.getBooleanProperty\(\"e2na\.devlogin\.enabled\",false\);)*/boolean isDevLoginEnabled = true;/' \"$DEVLOGIN_JSP_PATH\""
    /e2open/bin/eoadmin "grep -n 'isDevLoginEnabled' \"$DEVLOGIN_JSP_PATH\" | head -n 3"
    REMOTE
    ```
    If `devlogin.jsp` does not exist at this step, log a warning and continue.

20. After re-enabling devlogin.jsp, start all E2NA process
    queues:
    If `devlogin.jsp` does not exist, log a warning and skip this step.

    ```bash
    ssh $E2NA_HOST 'bash -s' <<'REMOTE'
    BASE="http://127.0.0.1:11080/e2na"
    USERNAME="<E2NA login user from Default environments>"
    CJ="/tmp/e2na_cookie_$$.txt"
    trap 'rm -f "$CJ"' EXIT

    LOGIN_URL=$(curl -sS -c "$CJ" -o /dev/null -w '%{redirect_url}' \
        "$BASE/local/devlogin.jsp?j_username=$USERNAME")
    if [ -z "$LOGIN_URL" ]; then
        echo "ERROR: devlogin did not redirect; devlogin.jsp not enabled or user $USERNAME invalid."
        exit 1
    fi
    CSRF_TOKEN=$(printf '%s' "$LOGIN_URL" | sed -n 's/.*[?&]csrfToken=\([^&]*\).*/\1/p')
    [ -z "$CSRF_TOKEN" ] && echo "WARNING: no csrfToken in devlogin redirect ($LOGIN_URL); continuing without token."

    PAGE=$(curl -sS -L -b "$CJ" -w '\n__HTTP__%{http_code} %{url_effective}' \
        "$BASE/console/web/admin/queuelist.action${CSRF_TOKEN:+?csrfToken=$CSRF_TOKEN}")

    QUEUES=$(printf '%s\n' "$PAGE" \
        | tr '>' '\n' \
        | sed -n "/name=['\"]selectedList['\"]/s/.*value=['\"]\([^'\"]*\)['\"].*/\1/p" \
        | sort -u)

    if [ -z "$QUEUES" ]; then
        echo "ERROR: no queues found from queuelist.action; not starting queues."
        echo "HTTP status/URL: $(printf '%s\n' "$PAGE" | sed -n 's/^__HTTP__//p')"
        echo "Page excerpt:"
        printf '%s\n' "$PAGE" | sed -e 's/<[^>]*>//g' -e '/^__HTTP__/d' | grep -v '^[[:space:]]*$' | head -n 20
        exit 1
    fi

    ARGS=()
    [ -n "$CSRF_TOKEN" ] && ARGS+=(--data "csrfToken=$CSRF_TOKEN")
    while IFS= read -r QUEUE; do
        [ -n "$QUEUE" ] && ARGS+=(--data-urlencode "selectedList=$QUEUE")
    done <<< "$QUEUES"

    echo "Starting queues:"
    printf '  %s\n' $QUEUES

    curl -i -sS -X POST "$BASE/console/web/admin/queuestartSelected.action" \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -b "$CJ" \
        "${ARGS[@]}" | sed -n "1,20p"
    REMOTE
    ```

21. After all E2NA process queues are started, abort any schedule that is
    currently abortable:
    If `devlogin.jsp` does not exist, log a warning and skip this step.

    ```bash
    ssh $E2NA_HOST 'bash -s' <<'REMOTE'
    BASE="http://127.0.0.1:11080/e2na"
    USERNAME="<E2NA login user from Default environments>"
    PROP="/e2open/config/system.properties"
    ACTION="abort"
    CJ="/tmp/e2na_cookie_$$.txt"
    trap 'rm -f "$CJ"' EXIT

    get_prop() {
        awk -F= -v key="$1" '
            /^[[:space:]]*#/ { next }
            {
                k = $1
                gsub(/^[[:space:]]+|[[:space:]]+$/, "", k)
                if (k == key) {
                    sub(/^[^=]*=/, "")
                    gsub(/^[[:space:]]+|[[:space:]]+$/, "")
                    gsub(/^"|"$/, "")
                    print
                    exit
                }
            }
        ' "$PROP"
    }

    HUB_NAME=$(get_prop "e2.ssp.hub.company.name")
    PROJECT_NAME=$(get_prop "e2.ssp.hub.name")

    if [ -z "$HUB_NAME" ] || [ -z "$PROJECT_NAME" ]; then
        echo "ERROR: hub/project missing from $PROP"
        exit 1
    fi

    LOGIN_URL=$(curl -sS -c "$CJ" -o /dev/null -w '%{redirect_url}' \
        "$BASE/local/devlogin.jsp?j_username=$USERNAME")
    if [ -z "$LOGIN_URL" ]; then
        echo "ERROR: devlogin did not redirect; devlogin.jsp not enabled or user $USERNAME invalid."
        exit 1
    fi
    CSRF_TOKEN=$(printf '%s' "$LOGIN_URL" | sed -n 's/.*[?&]csrfToken=\([^&]*\).*/\1/p')
    TOKEN_ARGS=()
    [ -n "$CSRF_TOKEN" ] && TOKEN_ARGS+=(--data "csrfToken=$CSRF_TOKEN")

    cd /e2open/app/projects/ssp-ext/E2/e2na/ebl/support_file || exit 1

    perl -0777 -ne '
        while (/<group\s+[^>]*name="([^"]+)"[^>]*>(.*?)<\/group>/sg) {
            my ($group, $body) = ($1, $2);
            while ($body =~ /<schedule\s+[^>]*name="([^"]+)"/sg) {
                print "$group\t$1\n";
            }
        }
    ' intel-schedules*.xml | sort -u | while IFS="$(printf '\t')" read -r GROUP_NAME SCHEDULE_NAME; do
        [ -n "$GROUP_NAME" ] && [ -n "$SCHEDULE_NAME" ] || continue

        DETAIL=$(curl -sS -X POST -G "$BASE/console/web/system/scheduledetails.action" \
            -b "$CJ" \
            "${TOKEN_ARGS[@]}" \
            --data-urlencode "hubName=$HUB_NAME" \
            --data-urlencode "projectName=$PROJECT_NAME" \
            --data-urlencode "groupName=$GROUP_NAME" \
            --data-urlencode "scheduleName=$SCHEDULE_NAME")

        if printf "%s" "$DETAIL" | grep -qi "action=abort"; then
            printf "Aborting: %s | %s\n" "$GROUP_NAME" "$SCHEDULE_NAME"
            curl -i -sS -X POST -G "$BASE/console/web/system/scheduleaction.action" \
                -b "$CJ" \
                "${TOKEN_ARGS[@]}" \
                --data-urlencode "hubName=$HUB_NAME" \
                --data-urlencode "projectName=$PROJECT_NAME" \
                --data-urlencode "groupName=$GROUP_NAME" \
                --data-urlencode "scheduleName=$SCHEDULE_NAME" \
                --data-urlencode "action=$ACTION" | sed -n "1p"
        else
            printf "Skipping: %s | %s\n" "$GROUP_NAME" "$SCHEDULE_NAME"
        fi
    done
    REMOTE
    ```

22. After queue start and scheduler abort checks are complete, restore
        `/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp` back to the
    original logic using `/e2open/bin/eoadmin` with `perl -i`. Do not rely on
    a backup file:

        ```bash
    ssh $E2NA_HOST "DEVLOGIN_JSP_PATH='/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp' bash -s" <<'REMOTE'
    set -euo pipefail
    /e2open/bin/eoadmin "perl -i -0777 -pe 's/boolean isDevLoginEnabled\s*=[^;]*;(\s*Configuration\.getBooleanProperty\(\"e2na\.devlogin\.enabled\",false\);)*/boolean isDevLoginEnabled =  \"dev\".equals(Configuration.getProperty(\"e2.env.subtype\")) ||\\n                  Configuration.getBooleanProperty(\"e2na.devlogin.enabled\",false);/' \"$DEVLOGIN_JSP_PATH\""
    /e2open/bin/eoadmin "grep -n 'isDevLoginEnabled' \"$DEVLOGIN_JSP_PATH\" | head -n 3"
    COUNT=$(/e2open/bin/eoadmin "grep -c 'getBooleanProperty(.e2na.devlogin.enabled' \"$DEVLOGIN_JSP_PATH\"" || true)
    [ "$COUNT" = "1" ] || echo "WARNING: devlogin.jsp has $COUNT getBooleanProperty(e2na.devlogin.enabled) lines after restore (expected 1)."
    REMOTE
        ```

        The restored original logic:

        ```java
        boolean isDevLoginEnabled =  "dev".equals(Configuration.getProperty("e2.env.subtype")) || 
                             Configuration.getBooleanProperty("e2na.devlogin.enabled",false);
        ```

Guardrails:
- Temporarily force-enable `devlogin.jsp` to `true` using `/e2open/bin/eoadmin`
    with the multi-line `perl -i -0777` replacement (not single-line `sed`) before
    the initial queue stop operation.
- If `devlogin.jsp` path does not exist during enable/restore, log a warning
    and continue deployment instead of failing the run.
- If `devlogin.jsp` path does not exist, skip E2NA queue stop/start operations
    and skip scheduler abort checks.
- After both E2SC and E2NA are confirmed started, re-enable `devlogin.jsp`
        to `true` using `/e2open/bin/eoadmin` with the multi-line `perl -i -0777`
        replacement before starting queues or aborting schedules.
    `setup install` rebuilds the file to its original logic, so the in-place
    change from step 1 is no longer present by the time the servers start.
- Stop E2NA process queues before running any `setup forcestop` command.
- If `setup forcestop` or `setup remove` fails only because
    `e2am (...) script is not installed: ...`, continue with `setup install`
    instead of stopping the deployment.
- Only `setup forcestop`, `setup remove`, and `setup install` may be run in
  parallel across E2SC, E2NA, and IM.
- Do not run E2NA `setup config` until E2SC `setup config` succeeds.
- Do not run IM `setup config` until E2NA `setup config` succeeds.
- Do not run E2SC `setup start` immediately after E2SC `setup config`.
- Do not run any `setup start` until E2SC and E2NA `setup config`
    commands have completed successfully.
- If IM `setup config` succeeds, do not run any `setup start` until IM
    `setup config` has also completed successfully.
- If IM `setup config` fails, log a warning, skip IM startup, and continue
    with E2SC/E2NA startup only.
- Run E2SC `setup start` before E2NA `setup start`.
- Do not run E2NA `setup start` until E2SC is confirmed started.
- Do not run IM `setup start` until E2NA is confirmed started and only if IM
    `setup config` succeeded.
- If the ordered start sequence fails, remove `*.zip` and `*.txt` files from
  `/e2open/var/shared/e2sc/upDn/pkgio` on both servers using eoadmin, then retry
  the ordered start sequence once.
- Do not retry startup more than once. If the recovery attempt fails, grep the
  failing server's install pkglog for `error`, write the matching lines into
  the deployment log, then stop and report the failed server and command.
- Start E2NA process queues only after E2SC and E2NA are confirmed started.
- If IM `setup config` fails, continue to send startup email and continue with
    E2NA queue start and scheduler abort operations after E2SC/E2NA are
    confirmed started.
- Send the startup email only after E2SC and E2NA are confirmed started.
- Abort currently abortable E2NA schedules only after all E2NA process queues
  are started.
- Restore `devlogin.jsp` to its original boolean logic using
    `/e2open/bin/eoadmin` with `perl -i` after queue start and scheduler abort
    operations are complete. Do not rely on a backup file.
- If any command fails, stop and report which server and command failed.
- Every E2NA console `/console/web/**/*.action` call must carry the CSRF
    token captured from the devlogin redirect (see below).
- If a queue/scheduler console call returns no usable data, print the HTTP
    status, final URL, and a short text excerpt of the page before exiting so
    the cause is visible in the deployment log.

E2NA console CSRF token:
- Newer E2NA builds enable a CSRF filter in `e2na.war/WEB-INF/web.xml`. Only
    paths listed in its `csrfExcludePatterns` (static files, `main.action`,
    `exit.action`, `services_rest/secure`, `csrfTokenerror.action`) work
    without a token. All other `.action` calls (`queuelist`,
    `queuestopSelected`, `queuestartSelected`, `scheduledetails`,
    `scheduleaction`) return an error page instead of data.
- After a successful login, `devlogin.jsp` redirects to
    `/e2na/console/home.jsp?csrfToken=<token>`. Capture it with
    `curl -o /dev/null -w '%{redirect_url}'` (do not use `-L` on the login call)
    and pass it as `?csrfToken=<token>` on GET calls and as
    `--data "csrfToken=<token>"` on POST/`-G` calls.
- If the login does not redirect at all, fail: devlogin is not enabled or the
    user is invalid. If it redirects without a token (older E2NA), log a
    warning and continue without the token.
- Symptom when the token is missing:
    `ERROR: no queues found from queuelist.action; not stopping queues.`
    while `eoadmin setup status` on `$E2NA_HOST` still shows
    `e2na started (active)`.

Failure analysis (when a deployment run fails):
1. Read the failed run log (for example under `logs/deploy/`). Find the
    first `ERROR:` line and the phase logged just before it.
2. Check history to find when the failure started and the last successful
    run:

    ```bash
    cd logs/deploy
    for f in $(ls -tr e2sc_e2na_deploy_*.log); do
        echo "$f: $(grep -m1 -E 'ERROR:|completed successfully|Stopping queues:' "$f")"
    done | awk -F': ' 'prev!=$2{print; prev=$2}'
    ```

3. Compare the last successful log with the first failing log. Look for
    `setup install` / `------ <pkg> installed` output, because a package
    upgrade is the most common cause of a sudden break. Also compare the line
    numbers printed by the `isDevLoginEnabled` grep across runs: if they
    changed, the upgrade replaced `devlogin.jsp`.
4. Verify the live state read-only on the affected host
    (`ssh -o BatchMode=yes <host> ...`): `/e2open/bin/eoadmin setup status`,
    `ls -l --time-style=long-iso` of changed files (for example
    `e2na.war/WEB-INF/web.xml` or `local/devlogin.jsp`), and
    `grep -n csrf e2na.war/WEB-INF/web.xml`.
5. Do not modify remote servers or re-run the deployment during analysis
    unless the user confirms.
6. Report: the failing step, the exact error, the first failing date, the last
    successful date, the number of consecutive failed runs, what changed
    between them, the live evidence, and a concrete fix.

Known failure patterns:

| Log message | Root cause | Fix |
|---|---|---|
| `no queues found from queuelist.action` | Console `.action` called without `csrfToken` (CSRF filter in `web.xml`) | Capture the token from the devlogin redirect and pass it on every console call |
| `devlogin did not redirect` / `Devlogin is not enabled on this stack` | Enable replacement did not match the new `devlogin.jsp`, or the E2NA login user does not exist | Check the `isDevLoginEnabled` declaration format and `E2NA_LOGIN_USER` |
| Repeated `Configuration.getBooleanProperty("e2na.devlogin.enabled",false);` lines in `devlogin.jsp` | Single-line `sed` enable left the continuation line behind, and the restore added another | Use the multi-line `perl -0777` enable/restore. It removes leftover lines |
| `Version Mismatch. e2sc needs to be migrated` during `setup config` | New e2sc package needs a schema migration | Run `/e2open/bin/eoadmin setup migrate e2sc` on `$E2SC_HOST` (after user confirmation), then re-run config |
| `devlogin.jsp not found` | E2NA webapp not deployed or the path changed | Check the `devlogin.jsp` path on `$E2NA_HOST` |
| `e2am (...) script is not installed` | IM host without e2am. This is an expected warning | None |

Prevention:
- Alert when two or more consecutive scheduled runs fail. Do not rely on only
    the success email, because a missing email is easy to overlook.
- Log the installed E2NA package version (or the `web.xml` checksum) on each
    run, and log a warning when it differs from the last successful run.
