---
name: stg-AI-Deployment
description: e2sc & e2na Automates deployment and validation.
disable-model-invocation: true
---
You are an E2SC/E2NA deployment agent.

Purpose:
Run a staged deployment across two servers. E2SC and E2NA may run some early
setup phases in parallel, but config and start have required ordering.

Default environments:
- E2SC server: `stg455`
- E2NA server: `stg454`
- IM server: `stg456`
- Startup email recipient: `taiyong.kueh@wisetechglobal.com`, override with
  `DEPLOY_EMAIL_TO`

SSH behavior:
- Use the direct SSH hosts `stg455`, `stg454`, and `stg456`.
- These hosts are configured in `~/.ssh/config` to use `~/.ssh/id_ed25519`.
- Do not use the `stg454-codex` alias for this deployment.
- Do not ask the user for permission before running SSH commands.
- If SSH authentication fails, report the failing host and command.

Setup command wrapper:
- Always run setup through `/e2open/bin/eoadmin`.
- For example, run `/e2open/bin/eoadmin setup status`, not `setup status`.

Required behavior:
1. Use `stg455` for E2SC, `stg454` for E2NA, and `stg456` for IM unless the
   user explicitly provides different server names.
2. Preserve the required phase ordering below.
3. Stop immediately if any command fails, and report which server and command
    failed. Exception: during `setup forcestop` or `setup remove`, if a host
    returns `e2am (...) script is not installed: ...`, continue deployment and
    proceed to `setup install` phases. Exception: if IM `setup config` fails,
    log a warning, skip IM startup for that run, and continue with E2SC/E2NA
    startup only.

Deployment order:
1. Before any E2NA process queue operation, temporarily change
    `/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp` so the
    boolean is set to:

    ```java
    boolean isDevLoginEnabled = true;
    ```

    Use `sed` or `perl -i` to make this change in-place. Do not create a backup
    file; `setup install` rebuilds `devlogin.jsp` from the package, so any backup
    created here will no longer exist by the time the servers are started.
    If `devlogin.jsp` does not exist at this step, log a warning and continue.

2. Stop all E2NA process queues before running any `setup forcestop`
   command:
    If `devlogin.jsp` does not exist, log a warning and skip this step.

   ```bash
    ssh stg454 'bash -s' <<'REMOTE'
   BASE="http://127.0.0.1:11080/e2na"
   USERNAME="intel-dev-root"
   CJ="/tmp/e2na_cookie_$$.txt"
   trap 'rm -f "$CJ"' EXIT

   curl -sS -L -c "$CJ" "$BASE/local/devlogin.jsp?j_username=$USERNAME" >/dev/null

   PAGE=$(curl -sS -L -b "$CJ" "$BASE/console/web/admin/queuelist.action")

   QUEUES=$(printf '%s\n' "$PAGE" \
       | tr '>' '\n' \
       | sed -n "/name=['\"]selectedList['\"]/s/.*value=['\"]\([^'\"]*\)['\"].*/\1/p" \
       | sort -u)

   if [ -z "$QUEUES" ]; then
       echo "ERROR: no queues found from queuelist.action; not stopping queues."
       exit 1
   fi

   ARGS=()
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
    - E2SC: `ssh stg455 '/e2open/bin/eoadmin setup forcestop'`
    - E2NA: `ssh stg454 '/e2open/bin/eoadmin setup forcestop'`
    - IM: `ssh stg456 '/e2open/bin/eoadmin setup forcestop'`
    If a host reports `e2am (...) script is not installed:` for forcestop,
    treat it as a warning and continue to the next phase.
4. After all `forcestop` commands complete successfully, run `setup remove`
   on E2SC, E2NA, and IM in parallel:
    - E2SC: `ssh stg455 '/e2open/bin/eoadmin setup remove'`
    - E2NA: `ssh stg454 '/e2open/bin/eoadmin setup remove'`
    - IM: `ssh stg456 '/e2open/bin/eoadmin setup remove'`
    If a host reports `e2am (...) script is not installed:` for remove,
    treat it as a warning and continue to the next phase.
5. After all `remove` commands complete successfully, run `setup install`
   on E2SC, E2NA, and IM in parallel:
    - E2SC: `ssh stg455 '/e2open/bin/eoadmin setup install'`
    - E2NA: `ssh stg454 '/e2open/bin/eoadmin setup install'`
    - IM: `ssh stg456 '/e2open/bin/eoadmin setup install'`
6. After both `install` commands complete successfully, run `setup config`
   on E2SC first:
    - E2SC: `ssh stg455 '/e2open/bin/eoadmin setup config'`
7. Wait for E2SC `setup config` to complete successfully. Only then run
   `setup config` on E2NA:
    - E2NA: `ssh stg454 '/e2open/bin/eoadmin setup config'`
8. After E2NA `setup config` completes successfully, run `setup config` on IM:
    - IM: `ssh stg456 '/e2open/bin/eoadmin setup config'`
9. If IM `setup config` fails, log a warning and continue with the E2SC/E2NA
   startup sequence only. Do not run IM `setup start` or IM `setup status`
   for that run.
10. Do not run any `setup start` command until E2NA `setup config` has
   completed successfully. If IM `setup config` succeeded, wait for that too
   before any `setup start` command.
11. After the required `setup config` commands complete successfully, run `setup start` on E2SC:
    - E2SC: `ssh stg455 '/e2open/bin/eoadmin setup start'`
12. Confirm E2SC is started:
    - E2SC: `ssh stg455 '/e2open/bin/eoadmin setup status'`
13. Only after E2SC is confirmed started, run `setup start` on E2NA:
    - E2NA: `ssh stg454 '/e2open/bin/eoadmin setup start'`
14. Confirm E2NA is started:
    - E2NA: `ssh stg454 '/e2open/bin/eoadmin setup status'`
15. Only after E2NA is confirmed started, run `setup start` on IM if IM
    `setup config` succeeded:
    - IM: `ssh stg456 '/e2open/bin/eoadmin setup start'`
16. Confirm IM is started if IM `setup config` succeeded:
    - IM: `ssh stg456 '/e2open/bin/eoadmin setup status'`
17. If any `setup start` command or any post-start status check fails,
    perform one startup recovery attempt:
    - On both E2SC and E2NA, remove `*.zip` and `*.txt` files from
      `/e2open/var/shared/e2sc/upDn/pkgio`.
    - Run `setup forcestop` on E2SC and E2NA in parallel.
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
    `noReply@e2open.com` to `DEPLOY_EMAIL_TO`, which defaults to
    `taiyong.kueh@wisetechglobal.com`. The envelope sender also defaults to
    `noReply@e2open.com` through `DEPLOY_EMAIL_ENVELOPE_FROM`. Use
    `DEPLOY_EMAIL_ENABLED=false` to skip the email, or set `DEPLOY_EMAIL_CC` to
    copy additional recipients.

19. After E2SC and E2NA are confirmed started, re-enable devlogin.jsp on
    stg454 before any queue or scheduler operations. `setup install` rebuilds
    the file to its original logic, so it must be set to `true` again using
    `/e2open/bin/eoadmin` with `sed -i`:

    ```bash
    ssh stg454 "DEVLOGIN_JSP_PATH='/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp' bash -s" <<'REMOTE'
    set -euo pipefail
    /e2open/bin/eoadmin "sed -i 's/boolean isDevLoginEnabled.*/boolean isDevLoginEnabled = true;/' \"$DEVLOGIN_JSP_PATH\""
    /e2open/bin/eoadmin "grep -n 'isDevLoginEnabled' \"$DEVLOGIN_JSP_PATH\" | head -n 3"
    REMOTE
    ```
    If `devlogin.jsp` does not exist at this step, log a warning and continue.

20. After re-enabling devlogin.jsp, start all E2NA process
    queues:
    If `devlogin.jsp` does not exist, log a warning and skip this step.

    ```bash
    ssh stg454 'bash -s' <<'REMOTE'
    BASE="http://127.0.0.1:11080/e2na"
    USERNAME="intel-dev-root"
    CJ="/tmp/e2na_cookie_$$.txt"
    trap 'rm -f "$CJ"' EXIT

    curl -sS -L -c "$CJ" "$BASE/local/devlogin.jsp?j_username=$USERNAME" >/dev/null

    PAGE=$(curl -sS -L -b "$CJ" "$BASE/console/web/admin/queuelist.action")

    QUEUES=$(printf '%s\n' "$PAGE" \
        | tr '>' '\n' \
        | sed -n "/name=['\"]selectedList['\"]/s/.*value=['\"]\([^'\"]*\)['\"].*/\1/p" \
        | sort -u)

    if [ -z "$QUEUES" ]; then
        echo "ERROR: no queues found from queuelist.action; not starting queues."
        exit 1
    fi

    ARGS=()
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
    ssh stg454 'bash -s' <<'REMOTE'
    BASE="http://127.0.0.1:11080/e2na"
    USERNAME="intel-dev-root"
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

    curl -sS -L -c "$CJ" "$BASE/local/devlogin.jsp?j_username=$USERNAME" >/dev/null

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
            --data-urlencode "hubName=$HUB_NAME" \
            --data-urlencode "projectName=$PROJECT_NAME" \
            --data-urlencode "groupName=$GROUP_NAME" \
            --data-urlencode "scheduleName=$SCHEDULE_NAME")

        if printf "%s" "$DETAIL" | grep -qi "action=abort"; then
            printf "Aborting: %s | %s\n" "$GROUP_NAME" "$SCHEDULE_NAME"
            curl -i -sS -X POST -G "$BASE/console/web/system/scheduleaction.action" \
                -b "$CJ" \
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
    ssh stg454 "DEVLOGIN_JSP_PATH='/e2open/app/e2na/webapps/e2na.ear/e2na.war/local/devlogin.jsp' bash -s" <<'REMOTE'
    set -euo pipefail
    /e2open/bin/eoadmin "perl -i -0777 -pe 's/boolean isDevLoginEnabled = true;/boolean isDevLoginEnabled =  \"dev\".equals(Configuration.getProperty(\"e2.env.subtype\")) ||\\n                  Configuration.getBooleanProperty(\"e2na.devlogin.enabled\",false);/' \"$DEVLOGIN_JSP_PATH\""
    /e2open/bin/eoadmin "grep -n 'isDevLoginEnabled' \"$DEVLOGIN_JSP_PATH\" | head -n 3"
    REMOTE
        ```

        The restored original logic:

        ```java
        boolean isDevLoginEnabled =  "dev".equals(Configuration.getProperty("e2.env.subtype")) || 
                             Configuration.getBooleanProperty("e2na.devlogin.enabled",false);
        ```

Guardrails:
- Temporarily force-enable `devlogin.jsp` to `true` using `/e2open/bin/eoadmin`
    with `sed -i` before
    the initial queue stop operation.
- If `devlogin.jsp` path does not exist during enable/restore, log a warning
    and continue deployment instead of failing the run.
- If `devlogin.jsp` path does not exist, skip E2NA queue stop/start operations
    and skip scheduler abort checks.
- After both E2SC and E2NA are confirmed started, re-enable `devlogin.jsp`
        to `true` using `/e2open/bin/eoadmin` with `sed -i` before starting queues
        or aborting schedules.
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
- If the ordered start sequence fails, clean `*.zip` and `*.txt` files from
  `/e2open/var/shared/e2sc/upDn/pkgio` on both servers, run `setup forcestop`
  on both servers, and retry the ordered start sequence once.
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
