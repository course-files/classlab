# GitHub Organization Semester Setup: Create Teams and Invite Students

This workflow creates GitHub teams for a course, and invites each student to the organization directly into the correct team. It reports every invitation that succeeded and every one that did not, and it is intended to be run once at the start of each semester.

**Input:** one file, `roster.csv`, with two columns: `team,username`. The list of teams is derived from this file automatically, so a student's row both defines their team and their invitation.

**Design rules used throughout this guide, so it never strands you:**
- **No command ever runs `cd`.** `RUN_DIR` is a fixed, absolute path, set once in step 1. Every command that reads or writes a file spells out `"$RUN_DIR/filename"`, so it works no matter where your terminal's current directory is.
- **No command ever runs `exit`.** A failed precondition prints a line starting with `STOP` and skips the rest of that block with an ordinary `if`/`else`; it never terminates your shell.

---

## How the workflow is ordered

| Step | What it does | Reversible? |
|------|---------------|-------------|
| 1 | Prerequisites, variables, and run folder | n/a |
| 2 | Prepare `roster.csv` | n/a |
| 3 | Validate the roster before anything is created | n/a (read only) |
| 4 | Create the teams | Yes (`gh api -X DELETE`) |
| 5 | Confirm the created teams | n/a (read only) |
| 6 | Invite students to their teams | Yes (cancel invite / remove member) |
| 6a | Redistribute students who change teams (optional, later re-runs) | Yes |
| 7 | Confirm invitations: successes and failures | n/a (read only) |
| 8 | End-of-run summary | n/a (read only) |

---

## 1. Prerequisites, Variables, and Run Folder

### 1.1 Install the tools (skip if already installed)

```shell
winget install --id GitHub.cli
winget install jqlang.jq
```

### 1.2 Confirm GitHub CLI is working and authenticated

```shell
gh --version
gh auth status
```

Log in only if you are not already logged in, and request the scope needed to create teams and send invitations:

```shell
gh auth login
gh auth refresh -h github.com -s admin:org
```

You must be an organization **owner**, or a **team maintainer** with permission to create teams, for this to succeed. If the organization enforces SAML single sign-on, authorize your CLI token for the organization.

### 1.3 Windows line-ending safeguard (Git Bash, MSYS2, Cygwin)

A native Windows `jq.exe` ends every output line with a carriage return, which breaks comparisons and API calls later. Define this wrapper once per terminal session; it is harmless on macOS and Linux.

```shell
jq() { command jq "$@" | tr -d '\r'; }
```

### 1.4 Set the organization and the run folder

Run this from wherever you keep this markdown file. `RUN_DIR` is resolved to a full path immediately, so it stops mattering where your terminal is afterwards.

```shell
ORG="BI-course"
SEMESTER="20260907-20261202-BI-BBIT-4-EC"     # used only to label files; change per semester

RUN_DIR="$(pwd)/github-semester-setup-$SEMESTER"
mkdir -p "$RUN_DIR"
echo "Run folder: $RUN_DIR"
```

`ORG`, `SEMESTER`, `RUN_DIR`, and the `jq` wrapper are ordinary shell variables and a shell function. They live only in this terminal session. **If you close the terminal or open a new one, run steps 1.3 and 1.4 again there before continuing** — the value of `SEMESTER` must be identical to the one you used originally, since it is also the folder name.

> **Data protection note:** These files contain student GitHub usernames. Do not commit `$RUN_DIR` to a repository. Store it securely for as long as your retention policy requires, then delete it.

---

## 2. Prepare `roster.csv`

Create `roster.csv` **inside `$RUN_DIR`** (for example, `"$RUN_DIR/roster.csv"`) with a header row, one line per student, using the GitHub **username**, not the student's real name or email:

```csv
team,username
bbt4106-group-a,octocat
bbt4106-group-a,defunkt
bbt4106-group-b,mojombo
mcs8104-thesis-cohort,pjhyett
```

A team slug **must** use only lowercase letters, numbers, and hyphens. GitHub always lowercases a team's display name to build its slug (`BBT4106 Group A` becomes `bbt4106-group-a`), and every later step in this guide uses the value from `roster.csv` directly as that slug in API URLs. If the roster contains uppercase letters, the slug GitHub actually creates will not match what the script expects, and every membership call for that team will fail with a confusing `404`. Step 3.5 checks for this, so do not weaken it.

Optionally, add a second file, `"$RUN_DIR/team-descriptions.csv"`, if you want a team to have a description or a non-default visibility. Any team without an entry here is created as a `closed` (visible to all organization members) team with no description, which is the recommended setting for a course team.

```csv
team,description,privacy
bbt4106-group-a,"BBT 4106 – Group A – Trimester 1 2027",closed
mcs8104-thesis-cohort,"MCS 8104 thesis supervision cohort",secret
```

`privacy` must be `closed` or `secret`. Use `secret` only for a team whose membership itself should not be visible to the rest of the organization, such as an examination or moderation panel; a normal course group should stay `closed`.

---

## 3. Validate the Roster

Do not proceed to step 4 until every check below reports `OK`.

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
elif [ ! -f "$RUN_DIR/roster.csv" ]; then
  echo "STOP: $RUN_DIR/roster.csv not found. Create it as shown in step 2."
else
  # 3.1 No carriage returns
  grep -l $'\r' "$RUN_DIR/roster.csv" && echo "STOP: fix line endings before continuing (see step 1.3)" || echo "OK: no carriage returns"

  # 3.2 Header is exactly as expected
  head -1 "$RUN_DIR/roster.csv"

  # 3.3 Extract the list of distinct teams and the list of rows, skipping the header
  tail -n +2 "$RUN_DIR/roster.csv" | awk -F',' '{print $1}' | sort -u > "$RUN_DIR/teams.txt"
  tail -n +2 "$RUN_DIR/roster.csv" > "$RUN_DIR/roster-rows.csv"
  echo "Distinct teams : $(wc -l < "$RUN_DIR/teams.txt" | tr -d ' ')"
  echo "Roster rows    : $(wc -l < "$RUN_DIR/roster-rows.csv" | tr -d ' ')"

  # 3.4 No duplicate team+username pairs
  sort "$RUN_DIR/roster-rows.csv" | uniq -d > "$RUN_DIR/roster-duplicates.txt"
  if [ -s "$RUN_DIR/roster-duplicates.txt" ]; then
    echo "STOP: duplicate rows found:"; cat "$RUN_DIR/roster-duplicates.txt"
  else
    echo "OK: no duplicate rows"
  fi

  # 3.5 Team slugs use ONLY lowercase letters, numbers, and hyphens (GitHub slugs are never uppercase)
  grep -Ev '^[a-z0-9-]+$' "$RUN_DIR/teams.txt" && echo "STOP: fix the team names above (lowercase, numbers, hyphens only)" || echo "OK: team names are valid slugs"
  # 3.6 A username must not appear twice for the SAME team (protects against copy-paste errors)
  awk -F',' '{print $1","$2}' "$RUN_DIR/roster-rows.csv" | sort | uniq -c | awk '$1 > 1 {print}' \
    > "$RUN_DIR/roster-repeat-in-team.txt"
  if [ -s "$RUN_DIR/roster-repeat-in-team.txt" ]; then
    echo "STOP: the same username is repeated within one team:"; cat "$RUN_DIR/roster-repeat-in-team.txt"
  else
    echo "OK: no repeats within a team"
  fi

  # 3.7 Confirm each GitHub username actually exists (catches typos before sending invitations)
  : > "$RUN_DIR/usernames.txt"
  : > "$RUN_DIR/invalid-usernames.txt"
  awk -F',' '{print $2}' "$RUN_DIR/roster-rows.csv" | sort -u > "$RUN_DIR/usernames.txt"
  while read -r user; do
    if gh api "users/$user" --silent 2>/dev/null; then
      :
    else
      echo "$user" | tee -a "$RUN_DIR/invalid-usernames.txt"
    fi
    sleep 0.2
  done < "$RUN_DIR/usernames.txt"

  if [ -s "$RUN_DIR/invalid-usernames.txt" ]; then
    echo "STOP: the usernames above do not exist on GitHub. Correct roster.csv and restart step 3."
  else
    echo "OK: every username exists ($(wc -l < "$RUN_DIR/usernames.txt" | tr -d ' ') students)"
  fi

  # 3.8 Which of these teams already exist (creating them again is harmless but worth knowing)
  gh api --paginate "orgs/$ORG/teams?per_page=100" --jq '.[].slug' | sort > "$RUN_DIR/existing-team-slugs.txt"
  echo "Teams in roster that already exist in $ORG:"
  comm -12 "$RUN_DIR/teams.txt" "$RUN_DIR/existing-team-slugs.txt"
fi
```

A misspelled GitHub username is the single most common reason an invitation fails, and step 3.7 catches it before any invitation is sent, not after.

---

## 4. Create the Teams

The first run is a dry run.

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
else
  DRY_RUN=true    # change to false only after reviewing the dry-run output

  : > "$RUN_DIR/teams-created.tsv"
  : > "$RUN_DIR/teams-create-failed.tsv"

  while read -r slug; do
    desc=$(awk -F',' -v s="$slug" '$1==s {print $2}' "$RUN_DIR/team-descriptions.csv" 2>/dev/null | tr -d '"')
    priv=$(awk -F',' -v s="$slug" '$1==s {print $3}' "$RUN_DIR/team-descriptions.csv" 2>/dev/null)
    priv=${priv:-closed}

    if grep -qxF "$slug" "$RUN_DIR/existing-team-slugs.txt"; then
      echo "already exists: $slug (skipped)"
      continue
    fi

    if [ "$DRY_RUN" = true ]; then
      echo "[dry-run] would create team '$slug' (privacy=$priv)${desc:+, description=\"$desc\"}"
    else
      if resp=$(gh api -X POST "orgs/$ORG/teams" -f name="$slug" -f privacy="$priv" \
                  ${desc:+-f description="$desc"} 2>&1); then
        created_slug=$(echo "$resp" | jq -r '.slug')
        if [ "$created_slug" != "$slug" ]; then
          printf 'WARN\t%s\tGitHub assigned a different slug: %s\n' "$slug" "$created_slug" | tee -a "$RUN_DIR/teams-created.tsv"
        fi
        printf 'CREATED\t%s\t%s\n' "$slug" "$created_slug" | tee -a "$RUN_DIR/teams-created.tsv"
      else
        printf 'FAILED\t%s\t%s\n' "$slug" "$resp" | tee -a "$RUN_DIR/teams-create-failed.tsv"
      fi
      sleep 0.5
    fi
  done < "$RUN_DIR/teams.txt"
fi
```

After reviewing the dry run, set `DRY_RUN=false` and run the block again. A `WARN` line means the slug GitHub created does not match `roster.csv`; this should not happen if step 3.5 passed, but if it does, fix the team name in `roster.csv` and step 5.1 will still flag it as missing.

---

## 5. Confirm the Created Teams

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
else
  # 5.1 Every team in the roster now exists, with the slug you expect
  gh api --paginate "orgs/$ORG/teams?per_page=100" --jq '.[].slug' | sort > "$RUN_DIR/existing-team-slugs.txt"
  echo "Teams in roster still missing from the organization (should print nothing):"
  comm -23 "$RUN_DIR/teams.txt" "$RUN_DIR/existing-team-slugs.txt"

  echo "Teams created this run : $(grep -c '^CREATED' "$RUN_DIR/teams-created.tsv" 2>/dev/null || echo 0)"
  echo "Team creations failed  : $(grep -c '^FAILED'  "$RUN_DIR/teams-create-failed.tsv" 2>/dev/null || echo 0)"
  [ -s "$RUN_DIR/teams-create-failed.tsv" ] && cat "$RUN_DIR/teams-create-failed.tsv"
fi
```

Resolve any failure (commonly a name collision with a team outside the roster, or a rate limit) before moving to step 6, since step 6 assumes every team from the roster exists.

---

## 6. Invite Students to Their Teams

Adding a username to a team membership both sends the organization invitation and pre-assigns the team, in a single call. A student who accepts the invitation lands directly in the right team with no separate step. A student who is **already** an organization member is added to the team immediately, with no invitation needed.

The first run is a dry run.

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
else
  # 6.0 Remove carriage returns from the roster files themselves, then confirm none remain
  for f in "$RUN_DIR/roster.csv" "$RUN_DIR/roster-rows.csv"; do
    [ -f "$f" ] && tr -d '\r' < "$f" > "$f.tmp" && mv "$f.tmp" "$f"
  done
  grep -l $'\r' "$RUN_DIR/roster.csv" "$RUN_DIR/roster-rows.csv" 2>/dev/null && echo "STOP: carriage returns remain" || echo "OK: no carriage returns found"

  DRY_RUN=true    # change to false only after reviewing the dry-run output
  N=$(wc -l < "$RUN_DIR/roster-rows.csv" | tr -d ' ')

  proceed=true
  if [ "$DRY_RUN" != true ]; then
    read -r -p "Type the number of invitations to send ($N) to proceed: " ANSWER
    if [ "$ANSWER" != "$N" ]; then
      echo "Confirmation failed; running as a dry run instead."
      DRY_RUN=true
    fi
  fi

  : > "$RUN_DIR/invites-sent.tsv"
  : > "$RUN_DIR/invites-failed.tsv"

  while IFS=',' read -r team user; do
    if [ "$DRY_RUN" = true ]; then
      echo "[dry-run] would add $user to team '$team' in $ORG"
    else
      if resp=$(gh api -X PUT "orgs/$ORG/teams/$team/memberships/$user" -f role=member 2>&1); then
        state=$(echo "$resp" | jq -r '.state')          # "pending" = invited, "active" = already a member
        printf 'OK\t%s\t%s\t%s\n' "$team" "$user" "$state" | tee -a "$RUN_DIR/invites-sent.tsv"
      else
        printf 'FAILED\t%s\t%s\t%s\n' "$team" "$user" "$resp" | tee -a "$RUN_DIR/invites-failed.tsv"
      fi
      sleep 0.5
    fi
  done < "$RUN_DIR/roster-rows.csv"
fi
```

After reviewing the dry run, set `DRY_RUN=false` and run the block again.

---

## 6a. Redistributing Students Who Change Teams (re-running with a corrected roster)

Step 6 only **adds** a student to the team named in `roster.csv`. It has no way to know that a student was previously in a different team, so re-running it with a corrected roster leaves a moved student in both the old team and the new one. This step closes that gap: it compares the roster you used last time against the one you are using now, and removes a student from their old team only when their new team is different.

This step is only needed when you are correcting an **existing** roster partway through a semester, not on the first run of step 6.

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
else
  # 6a.1 Point at the previous run's roster (a full path, or one relative to your current terminal location)
  PREVIOUS_ROSTER="$RUN_DIR/PREVIOUS_ROSTER/roster.csv"
  if [ -f "$PREVIOUS_ROSTER" ]; then
    echo "OK: found $PREVIOUS_ROSTER"

    # 6a.2 For every student, find their OLD team (if any) and compare it with the NEW team
    : > "$RUN_DIR/roster-changes.tsv"
    while IFS=',' read -r new_team user; do
      old_team=$(awk -F',' -v u="$user" '$2==u {print $1}' "$PREVIOUS_ROSTER" | tail -1)
      if [ -n "$old_team" ] && [ "$old_team" != "$new_team" ]; then
        printf 'MOVE\t%s\t%s\t%s\n' "$user" "$old_team" "$new_team" >> "$RUN_DIR/roster-changes.tsv"
      fi
    done < "$RUN_DIR/roster-rows.csv"

    echo "Students changing teams: $(wc -l < "$RUN_DIR/roster-changes.tsv" | tr -d ' ')"
    column -t -s $'\t' "$RUN_DIR/roster-changes.tsv"

    # 6a.3 Students who dropped out of the roster entirely (in the old roster, absent from the new one)
    awk -F',' '{print $2}' "$PREVIOUS_ROSTER" | sort -u > "$RUN_DIR/previous-usernames.txt"
    awk -F',' '{print $2}' "$RUN_DIR/roster-rows.csv" | sort -u > "$RUN_DIR/new-usernames.txt"
    comm -23 "$RUN_DIR/previous-usernames.txt" "$RUN_DIR/new-usernames.txt" > "$RUN_DIR/roster-dropped-users.txt"
    echo "Students no longer on any team in the new roster: $(wc -l < "$RUN_DIR/roster-dropped-users.txt" | tr -d ' ')"
    cat "$RUN_DIR/roster-dropped-users.txt"
  else
    echo "STOP: set PREVIOUS_ROSTER to the correct path before continuing."
  fi
fi
```

Review `roster-changes.tsv` and `roster-dropped-users.txt` before continuing. A name in `roster-dropped-users.txt` is not removed automatically; decide by hand whether that means the student withdrew from the unit (in which case you may want to remove them from the organization, using the team-audit-and-cleanup guide) or was simply omitted from the new file by mistake.

```shell
if [ -z "$RUN_DIR" ] || [ ! -f "$RUN_DIR/roster-changes.tsv" ]; then
  echo "STOP: run 6a.1–6a.3 above in this terminal first."
else
  # 6a.4 Remove each moved student from their OLD team only (dry run first)
  DRY_RUN=true
  N=$(wc -l < "$RUN_DIR/roster-changes.tsv" | tr -d ' ')

  if [ "$DRY_RUN" != true ]; then
    read -r -p "Type the number of students to move ($N) to proceed: " ANSWER
    if [ "$ANSWER" != "$N" ]; then
      echo "Confirmation failed; running as a dry run instead."
      DRY_RUN=true
    fi
  fi

  : > "$RUN_DIR/moves-done.tsv"
  : > "$RUN_DIR/moves-failed.tsv"

  while IFS=$'\t' read -r _ user old_team new_team; do
    if [ "$DRY_RUN" = true ]; then
      echo "[dry-run] would remove $user from '$old_team' (they will be added to '$new_team' in step 6)"
    else
      if errmsg=$(gh api -X DELETE "orgs/$ORG/teams/$old_team/memberships/$user" 2>&1); then
        printf 'REMOVED\t%s\t%s\n' "$user" "$old_team" | tee -a "$RUN_DIR/moves-done.tsv"
      else
        printf 'FAILED\t%s\t%s\t%s\n' "$user" "$old_team" "$errmsg" | tee -a "$RUN_DIR/moves-failed.tsv"
      fi
      sleep 0.5
    fi
  done < "$RUN_DIR/roster-changes.tsv"
fi
```

After this, run step 6 as normal with the current `roster.csv`; it will add every moved student to their new team. Removing a student from a team does not remove them from the organization or from any other team they belong to, so a student who is also, for example, a teaching assistant on a second team is unaffected.

---

## 7. Confirm Invitations: Successes and Failures

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
else
  echo "Roster rows processed   : $N"
  echo "Succeeded                : $(wc -l < "$RUN_DIR/invites-sent.tsv" 2>/dev/null | tr -d ' ')"
  echo "  of which pending       : $(awk -F'\t' '$3=="pending"' "$RUN_DIR/invites-sent.tsv" 2>/dev/null | wc -l | tr -d ' ')"
  echo "  of which already-active: $(awk -F'\t' '$3=="active"' "$RUN_DIR/invites-sent.tsv" 2>/dev/null | wc -l | tr -d ' ')"
  echo "Failed                   : $(wc -l < "$RUN_DIR/invites-failed.tsv" 2>/dev/null | tr -d ' ')"

  if [ -s "$RUN_DIR/invites-failed.tsv" ]; then
    echo
    echo "Failed invitations (team, username, reason):"
    column -t -s $'\t' "$RUN_DIR/invites-failed.tsv"
  fi
fi
```

A row in `invites-failed.tsv` is almost always one of:

| Message contains | Meaning | Action |
|---|---|---|
| `Not Found` and a note about `admin:org` scope | Your token lacks the scope | Repeat step 1.2 |
| `422` and `already has a pending invitation to a different role or team` | The student already has a different pending invitation | Cancel the existing invitation in the GitHub web UI, or wait for it to be resolved, then rerun |
| `403` and mentions organization SSO | The token is not authorized for this organization under SAML | Authorize the token from the message's URL, then rerun |
| `404` for the user | The username no longer exists (renamed or deleted), or the team slug is wrong (see step 2's note on uppercase slugs) | This should already have been caught in step 3.7; recheck the roster |

The invite loop can be safely rerun: a row already in `invites-sent.tsv` will show `active` on a second pass rather than sending a duplicate invitation, since GitHub treats the call as idempotent.

---

## 8. End-of-Run Summary

```shell
if [ -z "$RUN_DIR" ] || [ ! -d "$RUN_DIR" ]; then
  echo "STOP: RUN_DIR is not set or does not exist. Run step 1.4 in this terminal first."
else
  {
    echo "GitHub semester setup — $ORG — $SEMESTER"
    echo "Run folder: $RUN_DIR"
    echo "Date: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
    echo
    echo "Teams created this run   : $(grep -c '^CREATED' "$RUN_DIR/teams-created.tsv" 2>/dev/null || echo 0)"
    echo "Teams already existing   : $(comm -12 "$RUN_DIR/teams.txt" "$RUN_DIR/existing-team-slugs.txt" | wc -l | tr -d ' ')"
    echo "Team creation failures   : $(wc -l < "$RUN_DIR/teams-create-failed.tsv" 2>/dev/null | tr -d ' ')"
    echo "Invitations sent/updated : $(wc -l < "$RUN_DIR/invites-sent.tsv" 2>/dev/null | tr -d ' ')"
    echo "Invitations failed       : $(wc -l < "$RUN_DIR/invites-failed.tsv" 2>/dev/null | tr -d ' ')"
  } | tee "$RUN_DIR/run-summary.txt"

  # Pending invitations still awaiting a response, for reference later in the semester
  gh api --paginate "orgs/$ORG/invitations?per_page=100" \
    --jq '.[] | [.login // .email, .created_at, (.team_count|tostring)] | @tsv' \
    > "$RUN_DIR/pending-invitations.txt"
  echo "Total pending invitations in the organization: $(wc -l < "$RUN_DIR/pending-invitations.txt" | tr -d ' ')"
fi
```

Keep `run-summary.txt`, `invites-failed.tsv`, and `roster.csv` for this semester's records, and follow up on `invites-failed.tsv` individually, since a failed row means that student has no access.

---

## Recommendations

1. **Chase pending invitations after a week.** An invitation that a student never accepts leaves them outside every team indefinitely. Re-run the query in step 8 partway through the first week of term; anyone still listed has not accepted, and a reminder email resolves most of these.

2. **Keep `roster.csv` as your source of truth, in version control, but not in the organization's own repositories.** A private repository outside `$ORG` (for example, on your personal account) gives you history of every semester's roster without exposing student usernames inside the course organization.

3. **Add students individually mid-semester with the same command, not a new roster.** For a late add, run only the `gh api -X PUT ".../memberships/$user"` line from step 6 for that one row; there is no need to re-run the whole workflow.

4. **Consider `parent_team_id` for large cohorts.** If a unit has several groups (Group A, Group B, ...), creating one parent team for the unit and the groups as child teams lets you grant repository access once, at the parent, instead of once per group. This is a change to step 4, not to the roster format.

5. **Pair this script with the semester-end cleanup guide you already have.** The team-audit-and-cleanup guide identifies teams older than a chosen cutoff; running it before this script each semester, with a short cutoff such as 300 days, prevents last semester's team slugs from colliding with this semester's roster in step 3.8.

6. **Store `team-descriptions.csv` under version control alongside `roster.csv`.** It is optional today, but a description recording the unit code and semester makes a team self-explanatory a year later, when you are deciding in the cleanup guide whether it is still needed.

7. **A GitHub organization invitation expires after seven days.** If a cohort will not check email until closer to the first lab session, send the invitations no more than a few days beforehand, or plan to re-run step 6 to resend any that lapsed.
