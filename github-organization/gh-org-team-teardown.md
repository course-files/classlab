# GitHub Organization Team Audit and Cleanup

This workflow identifies GitHub teams that were created more than one year ago, saves them together with their members, confirms the saved copy, deletes the teams, and then optionally removes former students who no longer need access to the organization.

**Important:** Age is used only to identify candidates for review. Do not automatically delete a team solely because it is old. Creation date says when a team was made, not whether it is still in use.

---

## How the workflow is ordered

| Part | Step | Reversible? |
|------|------|-------------|
| A | 1. Prerequisites and variables | n/a |
| A | 2. List teams older than the cutoff | n/a (read only) |
| A | 3. Review and exclude teams | n/a (read only) |
| A | 4. Save teams with members | n/a (read only) |
| A | 5. Confirm the saved list | n/a (read only) |
| A | 6. Delete the old teams | Team can be recreated from the backup, but not restored |
| B | 7. Compute which members are safe to remove | n/a (read only) |
| B | 8. Remove those members from the organization | Re-invite possible within three months |

**Can the members also be deleted?** Yes, but this is a separate action. Deleting a team does **not** remove its members from the organization; they remain organization members and keep any access they hold outside that team. Removing a person from the organization requires the organization owner role and is done in Part B. Removal deletes the person's access to all organization repositories and removes them from every team.

---

## 1. Prerequisites and Variables

### 1.1 Requirements

- [GitHub CLI](https://cli.github.com/) (`gh`) and [`jq`](https://jqlang.github.io/jq/) installed.
- You must be an **organization owner** (team deletion also works for team maintainers, but removing organization members requires an owner).
- If the organization enforces SAML single sign-on, authorize your CLI token for the organization.

Authenticate and add the scopes needed to read and administer the organization:

```bash
gh auth login
gh auth refresh -h github.com -s read:org -s admin:org
gh auth status
```

### 1.2 Set the GitHub Organization

Replace `YOUR-ORG` with the name of your GitHub organization.

```bash
ORG="YOUR-ORG"
CUTOFF_DAYS=365   # Teams created more than this many days ago are candidates
```

### 1.3 Create a backup directory

```bash
BACKUP_DIR="github-cleanup-${ORG}-team-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$BACKUP_DIR"
cd "$BACKUP_DIR"
pwd
```

All later commands assume that your terminal is **inside this directory**. If you close the terminal, open a new one, run `cd` into the directory again, and set `ORG` again.

### 1.4 Windows line-ending safeguard (Git Bash, MSYS2, Cygwin)

A native Windows `jq.exe` ends every output line with a carriage return and line feed (CRLF) instead of a line feed alone. The invisible `\r` then becomes part of every slug and username read from the list files. This breaks API calls (`invalid control character in URL`) and, more seriously, makes `comm` and `grep` comparisons silently fail to match, so the protection lists in Part B would not protect anyone.

Define this wrapper **once per terminal session, before running any other step**. It is harmless on Linux and macOS.

```bash
jq() { command jq "$@" | tr -d '\r'; }
```

To check any list file, run `grep -l $'\r' FILE`. It must print nothing.

> **Data protection note:** The files created here contain student GitHub usernames, which are personal data. Do not commit this directory to a repository. Store it securely and delete it when your retention period ends.

---

## 2. List Teams Created More Than One Year Ago

The list endpoint (`GET /orgs/{org}/teams`) does **not** return `created_at`. The date is only present when each team is fetched individually, so the commands below first collect every team slug and then fetch each team's full record.

```bash
# 2.1 Collect every team slug in the organization
gh api --paginate "orgs/$ORG/teams?per_page=100" --jq '.[].slug' > all-team-slugs.txt
wc -l < all-team-slugs.txt

# 2.2 Fetch the full record of each team (includes created_at)
while read -r slug; do
  gh api "orgs/$ORG/teams/$slug"
done < all-team-slugs.txt | jq -s '.' > all-teams-full.json

# 2.3 Select teams older than the cutoff
jq --argjson days "$CUTOFF_DAYS" '
  [ .[]
    | select((.created_at | fromdateiso8601) < (now - ($days * 86400)))
    | { slug, name, id, created_at, privacy,
        parent: (.parent.slug // null),
        members_count, repos_count } ]
  | sort_by(.created_at)
' all-teams-full.json > candidate-teams.json

# 2.4 Display the candidates
echo "Candidate teams: $(jq 'length' candidate-teams.json)"
jq -r '.[] | [.created_at[0:10], .slug, ("members=" + (.members_count|tostring)), ("parent=" + (.parent // "-"))] | @tsv' \
  candidate-teams.json | column -t -s $'\t'
```

---

## 3. Review and Exclude Teams

Age alone does not prove that a team is unused. Review the list above and place the slug of every team you want to **keep** in `keep-teams.txt`, one per line. Typical exclusions are teams for courses that still run, teams used for staff, and teams that control access to shared repositories.

```bash
touch keep-teams.txt
nano keep-teams.txt      # or any editor; leave empty to keep nothing

jq -r '.[].slug' candidate-teams.json | grep -vxFf keep-teams.txt > teams-to-delete.txt
echo "Teams selected for deletion: $(wc -l < teams-to-delete.txt)"
cat teams-to-delete.txt
```

### 3.1 Guard against child teams

Deleting a parent team also deletes all of its child teams. The check below reports any child team that is **not** on your deletion list but would be deleted because its parent is.

```bash
jq -r '.[] | select(.parent != null) | [.slug, .parent.slug] | @tsv' all-teams-full.json |
while IFS=$'\t' read -r child parent; do
  if grep -qxF "$parent" teams-to-delete.txt && ! grep -qxF "$child" teams-to-delete.txt; then
    echo "WARNING: '$child' is a child of '$parent' and would be deleted implicitly."
  fi
done
```

If any warning appears, either add the child to `keep-teams.txt` and also add its parent, or move the child to a different parent before continuing.

---

## 4. Save the Teams Together with Their Members

For every team on the deletion list, the commands below save the team record, its members, its maintainers, and the repositories it can access. The repository list is included so that access can be restored if a team is deleted in error.

```bash
mkdir -p teams

while read -r slug; do
  d="teams/$slug"; mkdir -p "$d"

  jq --arg s "$slug" '.[] | select(.slug == $s)' all-teams-full.json > "$d/team.json"

  gh api --paginate "orgs/$ORG/teams/$slug/members?per_page=100" \
    | jq -s 'add // [] | map({login, id})' > "$d/members.json"

  gh api --paginate "orgs/$ORG/teams/$slug/members?role=maintainer&per_page=100" \
    | jq -s 'add // [] | map(.login)' > "$d/maintainers.json"

  gh api --paginate "orgs/$ORG/teams/$slug/repos?per_page=100" \
    | jq -s 'add // [] | map({full_name, role_name})' > "$d/repos.json"

  jq -n \
    --slurpfile t  "$d/team.json" \
    --slurpfile m  "$d/members.json" \
    --slurpfile mt "$d/maintainers.json" \
    --slurpfile r  "$d/repos.json" '
    { team: ($t[0] | { slug, name, id, created_at, privacy,
                       parent: (.parent.slug // null),
                       reported_members_count: .members_count }),
      members: $m[0], maintainers: $mt[0], repositories: $r[0] }
  ' > "$d/record.json"

  echo "saved $slug"
  sleep 0.5
done < teams-to-delete.txt

# Consolidated files
jq -s '.' teams/*/record.json > saved-teams-and-members.json

jq -r '
  ["team_slug","team_name","team_created_at","member_login","is_maintainer"],
  ( .[] as $r
    | ($r.members | if length == 0 then [{login: ""}] else . end)[]
    | [ $r.team.slug, $r.team.name, $r.team.created_at, .login,
        (.login as $l | ($r.maintainers | index($l)) != null) ] )
  | @csv
' saved-teams-and-members.json > saved-teams-and-members.csv
```

The two consolidated files are:

- `saved-teams-and-members.json` – complete record (the file to use for a restore).
- `saved-teams-and-members.csv` – one row per team and member, suitable for opening in a spreadsheet. A team with no members appears once with an empty member column.

---

## 5. Confirm the Saved List

Do not proceed until every check below passes.

```bash
# 5.1 Every team selected for deletion has been saved, and nothing extra was saved
diff <(sort teams-to-delete.txt) <(jq -r '.[].team.slug' saved-teams-and-members.json | sort) \
  && echo "PASS: saved teams match the deletion list exactly"

# 5.2 Counts
echo "Teams on deletion list : $(wc -l < teams-to-delete.txt | tr -d ' ')"
echo "Teams saved            : $(jq 'length' saved-teams-and-members.json)"
echo "Distinct members saved : $(jq -r '.[].members[].login' saved-teams-and-members.json | sort -u | wc -l | tr -d ' ')"
echo "CSV data rows          : $(($(wc -l < saved-teams-and-members.csv) - 1))"

# 5.3 Saved member count versus the count GitHub reported for each team
jq -r '.[] | [ .team.slug, ("saved=" + (.members|length|tostring)),
               ("reported=" + (.team.reported_members_count|tostring)),
               (if (.members|length) == .team.reported_members_count then "OK" else "CHECK" end) ] | @tsv' \
  saved-teams-and-members.json | column -t -s $'\t'
```

A `CHECK` row is not always an error (for example, the reported count and the member listing can differ for teams with child teams), but investigate each one before deleting. Finally, open `saved-teams-and-members.csv` and read it once. Keep a second copy of the backup directory somewhere outside this machine.

---

## 6. Delete the Old Teams

**This step cannot be undone.** A deleted team can be recreated from the backup, but it receives a new ID and a new creation date.

The first run is a dry run that only prints what would happen.

```bash
DRY_RUN=true    # change to false only after reviewing the dry-run output

read -r -p "Type the number of teams to delete ($(wc -l < teams-to-delete.txt | tr -d ' ')) to proceed: " ANSWER
[ "$ANSWER" = "$(wc -l < teams-to-delete.txt | tr -d ' ')" ] || { echo "Confirmation failed; the loop below will run as a dry run only."; DRY_RUN=true; }

while read -r slug; do
  if [ "$DRY_RUN" = true ]; then
    echo "[dry-run] would delete $ORG/$slug"
  else
    if gh api -X DELETE "orgs/$ORG/teams/$slug"; then
      echo "deleted $slug" | tee -a deleted-teams.log
    else
      echo "FAILED $slug" | tee -a deleted-teams.log
    fi
    sleep 1
  fi
done < teams-to-delete.txt
```

After reviewing the dry run, set `DRY_RUN=false` and run the same block again.

### 6.1 Verify the deletion

```bash
gh api --paginate "orgs/$ORG/teams?per_page=100" --jq '.[].slug' | sort > remaining-team-slugs.txt

echo "Deleted teams that still exist (should print nothing):"
comm -12 <(sort teams-to-delete.txt) remaining-team-slugs.txt

# Count-based second check (independent of string matching)
echo "Teams before: $(wc -l < all-team-slugs.txt | tr -d ' ')" \
     " after: $(wc -l < remaining-team-slugs.txt | tr -d ' ')" \
     " deleted per log: $(grep -c '^deleted' deleted-teams.log)"
```

On Windows, a carriage-return mismatch (see section 1.4) can make the `comm` check print nothing even though the teams still exist. Rely on the count line as well: before minus after must equal the number of teams deleted.

---

# Part B (Optional): Remove Former Students from the Organization

Complete Part A first. This part removes people who belonged to the deleted teams **and** to no team that was kept.

## What removal does

- The person is removed from the organization and from every team.
- The person loses access to all of the organization's repositories, including their coursework repositories if those live in the organization.
- Private forks of the organization's private repositories that the person owns are deleted.
- GitHub retains the person's previous privileges and settings for three months, so re-inviting them within that period can restore them.
- The person's personal GitHub account is not affected.

## 7. Compute Who Is Safe to Remove

A student may belong to a deleted team and also to a current team. The lists below protect anyone who is in a retained team, any organization owner, and anyone you name explicitly.

```bash
export LC_ALL=C   # consistent sort order for comm

# 7.0 Remove any carriage returns from earlier list files and confirm none remain
#     (requires the jq wrapper from section 1.4 to be defined in this terminal)
for f in *.txt; do tr -d '\r' < "$f" > "$f.tmp" && mv "$f.tmp" "$f"; done
grep -l $'\r' *.txt || echo "OK: no carriage returns found"

# 7.1 Everyone who was in the deleted teams (from the saved backup)
jq -r '.[].members[].login' saved-teams-and-members.json | tr '[:upper:]' '[:lower:]' | sort -u > members-of-deleted-teams.txt

# 7.2 Everyone in the teams that still exist
gh api --paginate "orgs/$ORG/teams?per_page=100" --jq '.[].slug' | while read -r slug; do
  gh api --paginate "orgs/$ORG/teams/$slug/members?per_page=100" --jq '.[].login'
done | tr '[:upper:]' '[:lower:]' | sort -u > members-of-retained-teams.txt

# 7.3 Organization owners
gh api --paginate "orgs/$ORG/members?role=admin&per_page=100" --jq '.[].login' \
  | tr '[:upper:]' '[:lower:]' | sort -u > org-owners.txt

# 7.4 People you always want to keep (staff, teaching assistants, yourself)
touch protect-users.txt
gh api user --jq .login >> protect-users.txt
tr '[:upper:]' '[:lower:]' < protect-users.txt | sort -u -o protect-users.txt

# 7.5 Final removal list
cat members-of-retained-teams.txt org-owners.txt protect-users.txt | sort -u > do-not-remove.txt
comm -23 members-of-deleted-teams.txt do-not-remove.txt > users-to-remove.txt

echo "Members of deleted teams : $(wc -l < members-of-deleted-teams.txt | tr -d ' ')"
echo "Protected (skipped)      : $(comm -12 members-of-deleted-teams.txt do-not-remove.txt | wc -l | tr -d ' ')"
echo "Selected for removal     : $(wc -l < users-to-remove.txt | tr -d ' ')"
cat users-to-remove.txt

# Integrity check: none of the list files may contain a carriage return
grep -l $'\r' *.txt || echo "OK: no carriage returns found"
```

Read `users-to-remove.txt` line by line. Add any username you wish to keep to `protect-users.txt` and rerun step 7.5.

## 8. Remove the Members from the Organization

```bash
DRY_RUN=true    # change to false only after reviewing the dry-run output

read -r -p "Type the number of members to remove ($(wc -l < users-to-remove.txt | tr -d ' ')) to proceed: " ANSWER
[ "$ANSWER" = "$(wc -l < users-to-remove.txt | tr -d ' ')" ] || { echo "Confirmation failed; the loop below will run as a dry run only."; DRY_RUN=true; }

while read -r user; do
  if [ "$DRY_RUN" = true ]; then
    echo "[dry-run] would remove $user from $ORG"
  else
    if gh api -X DELETE "orgs/$ORG/members/$user"; then
      echo "removed $user" | tee -a removed-members.log
    else
      echo "FAILED $user" | tee -a removed-members.log
    fi
    sleep 1
  fi
done < users-to-remove.txt
```

### 8.1 Verify the removal

```bash
echo "Users still in the organization (should print nothing):"
while read -r user; do
  if gh api "orgs/$ORG/members/$user" --silent 2>/dev/null; then echo "STILL A MEMBER: $user"; fi
done < users-to-remove.txt
```

### 8.2 Related cleanup (optional)

Team deletion and member removal do not touch these, so review them separately:

```bash
# Pending invitations
gh api --paginate "orgs/$ORG/invitations?per_page=100" --jq '.[] | [.login // .email, .created_at] | @tsv'

# Outside collaborators (people with repository access who are not members)
gh api --paginate "orgs/$ORG/outside_collaborators?per_page=100" --jq '.[].login'
```

---

## Restoring from the Backup

If a team was deleted in error, recreate it and re-add its members from `saved-teams-and-members.json`. Replace `TEAM-SLUG` with the slug to restore.

```bash
SLUG="TEAM-SLUG"

NAME=$(jq -r --arg s "$SLUG" '.[] | select(.team.slug==$s) | .team.name' saved-teams-and-members.json)
PRIV=$(jq -r --arg s "$SLUG" '.[] | select(.team.slug==$s) | .team.privacy' saved-teams-and-members.json)

gh api -X POST "orgs/$ORG/teams" -f name="$NAME" -f privacy="$PRIV"

jq -r --arg s "$SLUG" '.[] | select(.team.slug==$s) | .members[].login' saved-teams-and-members.json |
while read -r user; do
  gh api -X PUT "orgs/$ORG/teams/$SLUG/memberships/$user" -f role=member
done
```

Repository access is recorded in `teams/$SLUG/repos.json`. Restore it with `PUT orgs/$ORG/teams/$SLUG/repos/OWNER/REPO`, using the permission values `pull`, `triage`, `push`, `maintain`, or `admin` (the saved `role_name` values `read` and `write` correspond to `pull` and `push`). A person removed from the organization must first be re-invited (`PUT orgs/$ORG/memberships/USER`), and they must accept before they can be added to a team.

---

## Final Checklist

- [ ] `ORG` and `CUTOFF_DAYS` are correct.
- [ ] Candidate teams were reviewed by a person, and `keep-teams.txt` reflects that review.
- [ ] The child-team guard printed no warnings.
- [ ] Step 5 checks passed, and a second copy of the backup exists.
- [ ] Team deletion was run as a dry run first, then for real.
- [ ] `users-to-remove.txt` was read line by line before removal.
- [ ] Any coursework, assessment evidence, or student work that must be retained has been exported or archived before those students lose access.
- [ ] The backup directory is stored securely and is not committed to a repository.
