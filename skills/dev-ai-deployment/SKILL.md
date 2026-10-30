---
name: dev-ai-deployment
description: e2sc & e2na Automates deployment and validation.
disable-model-invocation: true
---
You are an E2SC/E2NA deployment agent.

Purpose:
Run a staged deployment across two servers. E2SC and E2NA may run some early
setup phases in parallel, but config and start have required ordering.

Default environments:
- E2SC server: `dev10925`
- E2NA server: `dev10924`
- IM server: `dev10926` by default. Skip IM
  steps only when the user explicitly opts out of IM for this run.
- Startup email recipients: `taiyong.kueh@wisetechglobal.com,weiliang.yee@wisetechglobal.com,asyraf.jasni@wisetechglobal.com,Ryan.Risser@wisetechglobal.com`, override with `DEPLOY_EMAIL_TO`

SSH behavior:
- Use the direct SSH hosts `dev10925`, `dev10924`, and `dev10926`.
- These hosts are configured in `~/.ssh/config` to use `~/.ssh/id_ed25519`.
- Do not use the `dev12115-codex` alias for this deployment.
- Do not ask the user for permission before running SSH commands.
- If SSH authentication fails, report the failing host and command.

Setup command wrapper:
- Always run setup through `/e2open/bin/eoadmin`.
- For example, run `/e2open/bin/eoadmin setup status`, not `setup status`.

Required behavior:
1. Use `dev10925` for E2SC, `dev10924` for E2NA, and `dev10926` for IM unless
   the user explicitly provides different server names.
2. Preserve the required phase ordering below.
3. Stop immediately if any command fails, and report which server and command
   failed. Exception: if IM `setup config` fails, log a warning, skip IM startup
   for that run, and continue with E2SC/E2NA startup only.
4. IM is optional. If the user opts out of IM, skip every IM command, IM
   check, and IM log lookup below, and run the deployment for E2SC and E2NA only.

Deployment order:
1. Stop all E2NA process queues before running any `setup forcestop`
   command:

   ```bash
   ssh dev10924 'bash -s' <<'REMOTE'
   BASE="http://dev10924.dev.e2open.com:11080/e2na"
   USERNAME="intel-dev-root"
   CJ="/tmp/e2na_cookie_$$.txt"
   trap 'rm -f "$CJ"' EXIT

   curl -sS -L -c "$CJ" -b "$CJ" -o /dev/null \
       --data-urlencode "j_username=$USERNAME" \
       --data-urlencode "login=Login" \
       "$BASE/local/devlogin.jsp"

   MAIN=$(curl -sS -L -b "$CJ" -c "$CJ" -w '\n%{url_effective}' "$BASE/console/web/main.action")
   FINAL_URL=$(printf '%s\n' "$MAIN" | tail -n 1)
   MAIN=$(printf '%s\n' "$MAIN" | sed '$d')
   case "$FINAL_URL" in
       *devlogin*|*csrfTokenerror*) echo "ERROR: E2NA login failed; final URL: $FINAL_URL"; exit 1 ;;
   esac
   CSRF=$(printf '%s\n' "$MAIN" | grep -oE 'csrfToken=[^&"<>[:space:]]+' | head -n 1 | sed 's/^csrfToken=//')
   if [ -z "$CSRF" ]; then echo "ERROR: csrfToken not found on main.action"; exit 1; fi

   PAGE=$(curl -sS -L -G -b "$CJ" -c "$CJ" --data-urlencode "csrfToken=$CSRF" "$BASE/console/web/admin/queuelist.action")

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
       --data-urlencode "csrfToken=$CSRF" \
       "${ARGS[@]}" | sed -n "1,20p"
   REMOTE
   ```

2. Run `setup forcestop` on E2SC, E2NA, and IM in parallel:
   - E2SC: `ssh dev10925 '/e2open/bin/eoadmin setup forcestop'`
   - E2NA: `ssh dev10924 '/e2open/bin/eoadmin setup forcestop'`
   - IM: `ssh dev10926 '/e2open/bin/eoadmin setup forcestop'`
3. After all `forcestop` commands complete successfully, run `setup remove`
   on E2SC, E2NA, and IM in parallel:
   - E2SC: `ssh dev10925 '/e2open/bin/eoadmin setup remove'`
   - E2NA: `ssh dev10924 '/e2open/bin/eoadmin setup remove'`
   - IM: `ssh dev10926 '/e2open/bin/eoadmin setup remove'`
4. After all `remove` commands complete successfully, run `setup install`
   on E2SC, E2NA, and IM in parallel:
   - E2SC: `ssh dev10925 '/e2open/bin/eoadmin setup install'`
   - E2NA: `ssh dev10924 '/e2open/bin/eoadmin setup install'`
   - IM: `ssh dev10926 '/e2open/bin/eoadmin setup install'`
5. After all `install` commands complete successfully, run `setup config`
   on E2SC first:
   - E2SC: `ssh dev10925 '/e2open/bin/eoadmin setup config'`
6. Wait for E2SC `setup config` to complete successfully. Only then run
   `setup config` on E2NA:
   - E2NA: `ssh dev10924 '/e2open/bin/eoadmin setup config'`
7. After E2NA `setup config` completes successfully, run `setup config` on IM:
   - IM: `ssh dev10926 '/e2open/bin/eoadmin setup config'`
8. If IM `setup config` fails, log a warning and continue with the E2SC/E2NA
   startup sequence only. Do not run IM `setup start` or IM `setup status`
   for that run.
9. Do not run any `setup start` command until E2NA `setup config` has completed
   successfully. If IM `setup config` succeeded, wait for that too before any
   `setup start` command.
10. After the required `setup config` commands complete successfully, run
    `setup start` on E2SC:
   - E2SC: `ssh dev10925 '/e2open/bin/eoadmin setup start'`
11. Confirm E2SC is started:
   - E2SC: `ssh dev10925 '/e2open/bin/eoadmin setup status'`
12. Only after E2SC is confirmed started, run `setup start` on E2NA:
   - E2NA: `ssh dev10924 '/e2open/bin/eoadmin setup start'`
13. Confirm E2NA is started:
   - E2NA: `ssh dev10924 '/e2open/bin/eoadmin setup status'`
14. Only after E2NA is confirmed started, run `setup start` on IM if IM
    `setup config` succeeded:
    - IM: `ssh dev10926 '/e2open/bin/eoadmin setup start'`
15. Confirm IM is started if IM `setup config` succeeded:
    - IM: `ssh dev10926 '/e2open/bin/eoadmin setup status'`
16. If any `setup start` command or any post-start status check fails,
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
17. After both E2SC and E2NA are confirmed started, send a startup email
    notification. IM is optional for this step. The deployment runner sends this from
    `noReply@e2open.com` to `DEPLOY_EMAIL_TO`, which defaults to
    `taiyong.kueh@wisetechglobal.com,weiliang.yee@wisetechglobal.com,asyraf.jasni@wisetechglobal.com,Ryan.Risser@wisetechglobal.com`. Mail is
    submitted to the relay `mailhost` on port 25; override with
    `DEPLOY_EMAIL_RELAY`. Use `DEPLOY_EMAIL_ENABLED=false` to skip the email.

    Build number: read it from the latest `ssp-ext` install on E2SC, then
    include it in the email subject and body:

    ```bash
    ssh dev10925 "grep -h 'install completed successfully: version' /e2open/var/log/install/pkglog/ssp-ext.install | tail -n 1 | sed 's/.*version //'"
    ```

    Example value: `inteltrunkiao1517`. If the output is empty, log a warning
    and send the email with build `unknown`.

    Setup status: include the `setup status` output in the email body, reusing
    the output from steps 11 and 13 (and 15 for IM if it was started). Put each
    block under a header line with the server name, for example:

    ```
    E2SC dev10925:
    <E2SC setup status output>

    E2NA dev10924:
    <E2NA setup status output>
    ```

    If IM was started, add an `IM dev10926:` block. If a status output is
    empty, log a warning and write `status unavailable` under that header.

18. After both E2SC and E2NA are confirmed started, start all E2NA process
    queues:

    ```bash
    ssh dev10924 'bash -s' <<'REMOTE'
    BASE="http://dev10924.dev.e2open.com:11080/e2na"
    USERNAME="intel-dev-root"
    CJ="/tmp/e2na_cookie_$$.txt"
    trap 'rm -f "$CJ"' EXIT

    curl -sS -L -c "$CJ" -b "$CJ" -o /dev/null \
        --data-urlencode "j_username=$USERNAME" \
        --data-urlencode "login=Login" \
        "$BASE/local/devlogin.jsp"

    MAIN=$(curl -sS -L -b "$CJ" -c "$CJ" -w '\n%{url_effective}' "$BASE/console/web/main.action")
    FINAL_URL=$(printf '%s\n' "$MAIN" | tail -n 1)
    MAIN=$(printf '%s\n' "$MAIN" | sed '$d')
    case "$FINAL_URL" in
        *devlogin*|*csrfTokenerror*) echo "ERROR: E2NA login failed; final URL: $FINAL_URL"; exit 1 ;;
    esac
    CSRF=$(printf '%s\n' "$MAIN" | grep -oE 'csrfToken=[^&"<>[:space:]]+' | head -n 1 | sed 's/^csrfToken=//')
    if [ -z "$CSRF" ]; then echo "ERROR: csrfToken not found on main.action"; exit 1; fi

    PAGE=$(curl -sS -L -G -b "$CJ" -c "$CJ" --data-urlencode "csrfToken=$CSRF" "$BASE/console/web/admin/queuelist.action")

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
        --data-urlencode "csrfToken=$CSRF" \
        "${ARGS[@]}" | sed -n "1,20p"
    REMOTE
    ```

19. After all E2NA process queues are started, abort any schedule that is
    currently abortable:

    ```bash
    ssh dev10924 'bash -s' <<'REMOTE'
    BASE="http://dev10924.dev.e2open.com:11080/e2na"
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

    curl -sS -L -c "$CJ" -b "$CJ" -o /dev/null \
        --data-urlencode "j_username=$USERNAME" \
        --data-urlencode "login=Login" \
        "$BASE/local/devlogin.jsp" || exit 1

    MAIN=$(curl -sS -L -b "$CJ" -c "$CJ" -w '\n%{url_effective}' "$BASE/console/web/main.action")
    FINAL_URL=$(printf '%s\n' "$MAIN" | tail -n 1)
    MAIN=$(printf '%s\n' "$MAIN" | sed '$d')
    case "$FINAL_URL" in
        *devlogin*|*csrfTokenerror*) echo "ERROR: E2NA login failed; final URL: $FINAL_URL"; exit 1 ;;
    esac
    CSRF=$(printf '%s\n' "$MAIN" | grep -oE 'csrfToken=[^&"<>[:space:]]+' | head -n 1 | sed 's/^csrfToken=//')
    if [ -z "$CSRF" ]; then echo "ERROR: csrfToken not found on main.action"; exit 1; fi

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
            --data-urlencode "csrfToken=$CSRF" \
            --data-urlencode "hubName=$HUB_NAME" \
            --data-urlencode "projectName=$PROJECT_NAME" \
            --data-urlencode "groupName=$GROUP_NAME" \
            --data-urlencode "scheduleName=$SCHEDULE_NAME")

        if printf "%s" "$DETAIL" | grep -qi "action=abort"; then
            printf "Aborting: %s | %s\n" "$GROUP_NAME" "$SCHEDULE_NAME"
            curl -i -sS -X POST -G "$BASE/console/web/system/scheduleaction.action" \
                -b "$CJ" \
                --data-urlencode "csrfToken=$CSRF" \
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

Guardrails:
- Stop E2NA process queues before running any `setup forcestop` command.
- Only `setup forcestop`, `setup remove`, and `setup install` may be run in
  parallel across E2SC, E2NA, and IM.
- Do not run E2NA `setup config` until E2SC `setup config` succeeds.
- Do not run IM `setup config` until E2NA `setup config` succeeds.
- Do not run E2SC `setup start` immediately after E2SC `setup config`.
- Do not run any `setup start` until both E2SC and E2NA `setup config`
  commands have completed successfully.
- If IM `setup config` succeeds, do not run any `setup start` until IM
  `setup config` has also completed successfully.
- If IM `setup config` fails, log a warning, skip IM startup, and continue
  with E2SC/E2NA startup only.
- Run E2SC `setup start` before E2NA `setup start`.
- Do not run IM `setup start` until E2NA is confirmed started and only if IM
  `setup config` succeeded.
- If the user opts out of IM, skip all IM guardrails and IM steps.
- Do not run E2NA `setup start` until E2SC is confirmed started.
- If the ordered start sequence fails, remove `*.zip` and `*.txt` files from
  `/e2open/var/shared/e2sc/upDn/pkgio` on both servers using eoadmin, then retry
  the ordered start sequence once.
- Do not retry startup more than once. If the recovery attempt fails, grep the
  failing server's install pkglog for `error`, write the matching lines into
  the deployment log, then stop and report the failed server and command.
- Start E2NA process queues only after both E2SC and E2NA are confirmed
  started.
- Send the startup email only after both E2SC and E2NA are confirmed started.
- Abort currently abortable E2NA schedules only after all E2NA process queues
  are started.
- If any command fails, stop and report which server and command failed.
- Every E2NA console request sends `csrfToken`. Fail the E2NA login if the final URL contains `devlogin` or `csrfTokenerror`, or if no token is found on `main.action`.
