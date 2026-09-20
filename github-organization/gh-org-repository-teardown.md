# GitHub Organization Repository Teardown Script

This workflow identifies repositories that have not received a push for more than one year, saves a verified backup of them outside any Git repository, archives them, and then lets you decide whether to delete the archived repositories.

**Important:** Inactivity is used only to identify candidates for review. Do not archive or delete a repository solely because it is old.

The commands assume a Bash shell (Git Bash on Windows, WSL, macOS, or Linux).

---

## How the workflow is ordered

| Part | Step | Reversible? |
|------|------|-------------|
| A | 1. Prerequisites, variables, and backup folder | n/a |
| A | 2. Inventory all repositories | n/a (read only) |
| A | 3. Identify repositories with no push for more than the cutoff | n/a (read only) |
| A | 4. Review and exclude repositories | n/a (read only) |
| A | 5. Save the backup (metadata and full Git history) | n/a (read only) |
| A | 6. Confirm the saved backup | n/a (read only) |
| A | 7. Archive the repositories | Yes (`unarchive`) |
| B | 8. Build the deletion list | n/a (read only) |
| B | 9. Pre-deletion checks | n/a (read only) |
| B | 10. Delete the archived repositories | **No** (see the restore notes) |
| B | 11. Verify the deletion | n/a |

Archiving makes a repository read-only. It remains visible, cloneable, and searchable. Deletion is a separate decision, and Part B is optional.

---

## 1. Prerequisites, Variables, and Backup Folder

### 1.1 Install the tools

```shell
winget install --id GitHub.cli
winget install jqlang.jq
```

Close and reopen the terminal afterwards so that the new programs are found.

### 1.2 Confirm that GitHub CLI is working and authenticated

```shell
gh --version
gh auth status
```

Log in only if you are not already logged in:

```shell
gh auth login
```

Grant the scopes needed to list and archive repositories, and let `git` reuse the CLI credentials (required to clone private repositories and wikis):

```shell
gh auth refresh -h github.com -s repo -s read:org
gh auth setup-git
```

```shell
gh auth refresh -h github.com -s read:org -s admin:org
gh auth status
```

The extra scope needed to delete repositories is requested later, in Part B.

### 1.3 Set the GitHub Organization and the cutoff

Replace `BI-course` with your organization if necessary.

```shell
ORG="BI-course"
CUTOFF_DAYS=365   # Repositories with no push for more than this many days are candidates
```

### 1.4 Windows line-ending safeguard (Git Bash, MSYS2, Cygwin)

A native Windows `jq.exe` ends every output line with a carriage return and line feed (CRLF) instead of a line feed alone. The invisible `\r` then becomes part of every repository name read from the list files. This breaks API calls and makes `comm` and `grep` comparisons silently fail to match.

Define this wrapper **once per terminal session, before running any other step**. It is harmless on Linux and macOS.

```shell
jq() { command jq "$@" | tr -d '\r'; }
```

### 1.5 Create the backup folder outside any Git repository

Everything the workflow creates (lists, logs, metadata, and full copies of the repositories) is written to a folder that is separate from the folder holding this markdown file. The check below stops you if the chosen location sits inside a Git repository.

```shell
BACKUP_DIR="github-cleanup-${ORG}-repos-$(date -u +%Y%m%dT%H%M%SZ)"
mkdir -p "$BACKUP_DIR"
cd "$BACKUP_DIR"
pwd
```

```shell
if [ "$(git -C "$BACKUP_DIR" rev-parse --is-inside-work-tree 2>/dev/null)" = "true" ]; then
  echo "STOP: $BACKUP_DIR is inside a Git repository. Choose a different BACKUP_ROOT and repeat this step."
else
  printf '*\n' > "$BACKUP_DIR/.gitignore"     # second line of defence against accidental commits
  cd "$BACKUP_DIR" && pwd
fi
```

If the message begins with `STOP`, do not continue until it is resolved.

All later commands assume that your terminal is **inside this backup folder**. If you close the terminal, open a new one, then set `ORG` and the `jq` wrapper again and resume with:

```shell
cd "$(ls -dt "$HOME/github-cleanup-backups/repos-${ORG}-"* | head -1)" && pwd
```

> **Data protection note:** The backup contains student work and usernames, which are personal data. It is not meant to be copied into a public repository.

---

## 2. Inventory All Repositories

```shell
gh repo list "$ORG" \
  --limit 5000 \
  --json name,nameWithOwner,description,visibility,isPrivate,isArchived,isFork,isEmpty,isTemplate,createdAt,pushedAt,updatedAt,diskUsage,defaultBranchRef,primaryLanguage,repositoryTopics,licenseInfo,forkCount,stargazerCount,hasIssuesEnabled,hasWikiEnabled,url \
  > all-repositories.json

# A simple tab-separated overview
jq -r '.[] | [.name, .pushedAt, .updatedAt, (.isArchived|tostring), (.isPrivate|tostring)] | @tsv' \
  all-repositories.json > all-repositories.tsv
```

Check that the listing is not truncated. The two numbers should be equal or very close (repositories you cannot see may explain a small difference):

```shell
echo "Listed by gh    : $(jq 'length' all-repositories.json)"
gh api "orgs/$ORG" --jq '"Reported by GitHub: " + ((.public_repos + (.total_private_repos // 0)) | tostring)'
```

---

## 3. Identify Repositories with No Push for More Than the Cutoff

The cutoff is calculated inside `jq`, so no date command is required and the result is the same on every operating system. Repositories that are already archived are excluded.

```shell
# 3.1 Candidates
jq --argjson days "$CUTOFF_DAYS" '
  [ .[]
    | select(.isArchived == false and .pushedAt != null)
    | select((.pushedAt | fromdateiso8601) < (now - ($days * 86400)))
    | { name, pushedAt, updatedAt, visibility, isFork, isTemplate, diskUsage, url } ]
  | sort_by(.pushedAt)
' all-repositories.json > candidate-repositories.json

# 3.2 Repositories that have never been pushed to (empty repositories) need a separate decision
jq '[ .[] | select(.isArchived == false and .pushedAt == null)
          | { name, createdAt, isEmpty, visibility } ]' \
  all-repositories.json > never-pushed-repositories.json

# 3.3 Display both lists
echo "Candidate repositories : $(jq 'length' candidate-repositories.json)"
jq -r '.[] | [ .pushedAt[0:10], .name, .visibility,
               ("updated=" + .updatedAt[0:10]),
               ("fork=" + (.isFork|tostring)),
               ("template=" + (.isTemplate|tostring)),
               ((.diskUsage / 1024 | floor | tostring) + " MB") ] | @tsv' \
  candidate-repositories.json | column -t -s $'\t'

echo "Never pushed (not archived by this workflow): $(jq 'length' never-pushed-repositories.json)"
jq -r '.[] | [.name, .createdAt[0:10], ("empty=" + (.isEmpty|tostring))] | @tsv' \
  never-pushed-repositories.json | column -t -s $'\t'
```

A push date reflects only pushes to Git. A repository with an old push date can still be in use through issues, pull requests, or GitHub Pages, so compare the `pushedAt` and `updated` columns while reviewing.

---

## 4. Review and Exclude Repositories

Place the name of every repository you want to **keep active** in `keep-repos.txt`, one per line. Typical exclusions are template repositories that are still used, shared course resources, and repositories that publish a GitHub Pages site.

```shell
touch keep-repos.txt
# nano keep-repos.txt      # or any editor; leave empty to keep nothing

jq -r '.[].name' candidate-repositories.json | grep -vxFf keep-repos.txt > repos-to-archive.txt
echo "Repositories selected for archiving: $(wc -l < repos-to-archive.txt | tr -d ' ')"
cat repos-to-archive.txt
```

---

## 5. Save the Backup

Archiving preserves the repositories on GitHub, but the saved copy protects you against later deletion, accidental changes by other owners, and loss of the organization. The backup has three parts: metadata, a complete mirror of the Git history (all branches and tags), and, optionally, issues and pull requests.

### 5.1 Save the metadata

```shell
jq -R . repos-to-archive.txt | jq -s . > selected-names.json

jq --slurpfile sel selected-names.json \
  '[ .[] | select(.name as $n | $sel[0] | index($n)) ]' \
  all-repositories.json > saved-repositories.json

jq -r '
  ["name","visibility","description","created_at","pushed_at","updated_at","disk_usage_kb",
   "default_branch","primary_language","is_fork","is_template","forks","stars","url"],
  ( .[] | [ .name, .visibility, (.description // ""), .createdAt, .pushedAt, .updatedAt, .diskUsage,
            (.defaultBranchRef.name // ""), (.primaryLanguage.name // ""),
            .isFork, .isTemplate, .forkCount, .stargazerCount, .url ] )
  | @csv
' saved-repositories.json > saved-repositories.csv
```

### 5.2 Save a full mirror of each repository

A mirror clone contains every branch and tag, so the complete commit history is preserved. Check the disk space first:

```shell
echo "Estimated size to clone: $(jq '[.[].diskUsage] | add / 1024 | floor' saved-repositories.json) MB"
df -h .
```

Then clone. The loop can be run again safely; repositories that were already cloned are skipped.

```shell
mkdir -p mirrors wikis

while IFS= read -r repo; do
  if [ -d "mirrors/$repo.git" ]; then echo "already saved: $repo"; continue; fi

  echo "Cloning $ORG/$repo"
  if ! gh repo clone "$ORG/$repo" "mirrors/$repo.git" -- --mirror; then
    printf 'FAILED\t%s\n' "$repo" | tee -a clone-failures.tsv
    rm -rf "mirrors/$repo.git"       # remove a partial clone so that a rerun tries again
    continue
  fi

  # Wiki: best effort, because most repositories have none
  GIT_TERMINAL_PROMPT=0 git clone --mirror "https://github.com/$ORG/$repo.wiki.git" \
    "wikis/$repo.wiki.git" >/dev/null 2>&1 && echo "  wiki saved"
done < repos-to-archive.txt
```

### 5.3 Optional: save issues and pull requests

A mirror clone does not contain issues, pull requests, or their discussions. Skip this step if the repositories hold no discussion that matters to you.

```shell
mkdir -p issues-prs

while IFS= read -r repo; do
  echo "Saving issues and pull requests: $repo"
  gh issue list -R "$ORG/$repo" --state all --limit 5000 \
    --json number,title,state,author,createdAt,closedAt,labels,body,comments \
    > "issues-prs/$repo.issues.json" 2>/dev/null || echo "  (issues disabled or unavailable)"
  gh pr list -R "$ORG/$repo" --state all --limit 5000 \
    --json number,title,state,author,createdAt,closedAt,mergedAt,body,comments,reviews \
    > "issues-prs/$repo.prs.json" 2>/dev/null || echo "  (pull requests unavailable)"
  sleep 0.5
done < repos-to-archive.txt
```

The backup does **not** include GitHub Actions history and artifacts, release attachments, GitHub Pages content, Projects, Discussions, or repository settings such as webhooks and branch protection.

---

## 6. Confirm the Saved Backup

Do not proceed until every check below passes.

```shell
# 6.1 Every selected repository has a mirror, and nothing extra was saved
diff <(sort repos-to-archive.txt) <(ls mirrors | sed 's/\.git$//' | sort) \
  && echo "PASS: saved mirrors match the archive list exactly"

# 6.2 Metadata counts
echo "Repositories on the archive list : $(wc -l < repos-to-archive.txt | tr -d ' ')"
echo "Repositories in saved metadata   : $(jq 'length' saved-repositories.json)"
echo "Rows in the CSV                  : $(($(wc -l < saved-repositories.csv) - 1))"
```

The CSV row count can exceed the repository count if a description contains line breaks. The JSON count is authoritative.

```shell
# 6.3 Compare the tip of each mirror with the tip currently on GitHub
: > backup-verification.tsv
while IFS= read -r repo; do
  m="mirrors/$repo.git"
  if [ ! -d "$m" ]; then printf 'MISSING\t%s\n' "$repo" >> backup-verification.tsv; continue; fi
  local_sha=$(git -C "$m" rev-parse HEAD 2>/dev/null)
  remote_sha=$(GIT_TERMINAL_PROMPT=0 git -C "$m" ls-remote origin HEAD 2>/dev/null | cut -f1)
  if   [ -z "$local_sha" ] && [ -z "$remote_sha" ]; then status="EMPTY"
  elif [ "$local_sha" = "$remote_sha" ];            then status="OK"
  else                                                   status="CHECK"; fi
  printf '%s\t%s\n' "$status" "$repo" >> backup-verification.tsv
done < repos-to-archive.txt

cut -f1 backup-verification.tsv | sort | uniq -c
grep -v '^OK' backup-verification.tsv     # lists every repository that is not OK
```

Every line must read `OK` (or `EMPTY` for a repository with no commits). Investigate each `CHECK` (someone pushed after the copy was made, or the copy is incomplete) and each `MISSING` before continuing, then rerun 5.2 and 6.3.

```shell
# 6.4 Record which repositories have a verified backup, and check the folder size
awk -F'\t' '$1=="OK" || $1=="EMPTY" {print $2}' backup-verification.tsv | sort -u > repos-backup-verified.txt
echo "Verified backups: $(wc -l < repos-backup-verified.txt | tr -d ' ')"
du -sh mirrors wikis 2>/dev/null

# 6.5 No list file may contain a carriage return
grep -l $'\r' *.txt || echo "OK: no carriage returns found"
```

Finally, open `saved-repositories.csv` and read it once, and keep a second copy of the backup folder on a different disk or storage service.

---

## 7. Archive the Repositories

Archiving is reversible with `gh repo unarchive`. The first run is a dry run that only prints what would happen.

```shell
DRY_RUN=true    # change to false only after reviewing the dry-run output
N=$(wc -l < repos-to-archive.txt | tr -d ' ')

if [ "$DRY_RUN" != true ]; then
  read -r -p "Type the number of repositories to archive ($N) to proceed: " ANSWER
  [ "$ANSWER" = "$N" ] || { echo "Confirmation failed; the loop below will run as a dry run only."; DRY_RUN=true; }
fi

while IFS= read -r repo; do
  if [ "$DRY_RUN" = true ]; then
    echo "[dry-run] would archive $ORG/$repo"
  else
    echo "Archiving $ORG/$repo"
    if gh repo archive "$ORG/$repo" --yes; then
      printf 'ARCHIVED\t%s\n' "$repo" | tee -a archive-log.tsv
    else
      printf 'FAILED\t%s\n' "$repo" | tee -a archive-log.tsv
    fi
    sleep 1
  fi
done < repos-to-archive.txt
```

After reviewing the dry run, set `DRY_RUN=false` and run the same block again. If you see `Not Found (HTTP 404)` with a message about a missing scope, repeat step 1.2.

### 7.1 Verify the archiving

```shell
gh repo list "$ORG" --limit 5000 --json name,isArchived \
  --jq '.[] | select(.isArchived == true) | .name' | sort > archived-now.txt

echo "Selected repositories that are still not archived (should print nothing):"
comm -23 <(sort repos-to-archive.txt) archived-now.txt

echo "Selected: $(wc -l < repos-to-archive.txt | tr -d ' ')" \
     " archived per log: $(grep -c '^ARCHIVED' archive-log.tsv)"
```

The count line is independent of string matching, so it also reveals a false pass caused by hidden characters in the lists.

---

# Part B (Optional): Delete the Archived Repositories

Complete Part A first. **Archiving already achieves read-only preservation, so deletion is a separate decision that mainly saves clutter and removes content permanently.** You may stop after step 7 and return to this part later.

## What deletion does

- The repository is removed from GitHub, including its issues, pull requests, wiki, Actions history, releases, and Pages site.
- **Deleting a private repository also deletes all of its forks.** Forks of a public repository continue to exist. If students submitted work as forks of a private repository, that work is deleted with it.
- Links and clones that point to the repository stop working.
- GitHub may allow an organization owner to restore a deleted repository within 90 days (organization Settings, Deleted repositories), with limits: release attachments and team permissions are not restored, and restoration is not possible in every case. Treat it as an emergency measure, not as a backup.

Unlike the original script, which deleted **every** archived repository in the organization, this part deletes only repositories that were archived by this run **and** have a verified backup from step 6.

## 8. Build the Deletion List

```shell
# 8.1 Request the extra scope needed to delete repositories
gh auth refresh -h github.com -s delete_repo
gh auth status

# 8.2 Archived in this run AND verified in step 6
awk -F'\t' '$1=="ARCHIVED" {print $2}' archive-log.tsv | sort -u > repos-archived-this-run.txt
sort -u repos-backup-verified.txt > repos-backup-verified.sorted.txt
comm -12 repos-archived-this-run.txt repos-backup-verified.sorted.txt > delete-candidates.txt

echo "Archived in this run but WITHOUT a verified backup (never deleted by this workflow):"
comm -23 repos-archived-this-run.txt repos-backup-verified.sorted.txt

# 8.3 Names to keep permanently (archived, but never deleted)
touch keep-forever-repos.txt
nano keep-forever-repos.txt      # one repository name per line; leave empty to keep nothing

grep -vxFf keep-forever-repos.txt delete-candidates.txt > repos-to-delete.txt
echo "Repositories selected for deletion: $(wc -l < repos-to-delete.txt | tr -d ' ')"
cat repos-to-delete.txt
```

Repositories that were archived before this run are deliberately excluded. To include one, add its name to `repos-to-archive.txt` before step 5 so that it receives a verified backup.

## 9. Pre-Deletion Checks

```shell
# 9.1 Forks, stars, and visibility of each repository selected for deletion
jq -R . repos-to-delete.txt | jq -s . > delete-names.json
jq -r --slurpfile sel delete-names.json '
  .[] | select(.name as $n | $sel[0] | index($n))
      | [ .name, .visibility, ("forks=" + (.forkCount|tostring)),
          ("stars=" + (.stargazerCount|tostring)), ("last_push=" + .pushedAt[0:10]) ] | @tsv
' saved-repositories.json | column -t -s $'\t'

# 9.2 Repositories that publish a GitHub Pages site
while IFS= read -r repo; do
  gh api "repos/$ORG/$repo/pages" --silent 2>/dev/null && echo "PAGES ENABLED: $repo"
done < repos-to-delete.txt
```

Read the output carefully. A private repository with `forks` above zero will take those forks with it, and a repository with Pages enabled will take the published site offline. Add such repositories to `keep-forever-repos.txt` and rerun step 8.3 if you are not certain.

## 10. Delete the Archived Repositories

**This step cannot be reliably undone.** The first run is a dry run. A real run requires you to type the organization name.

Confirm authentication status:

```shell
gh auth refresh -h github.com -s read:org -s admin:org
gh auth status
```

**Option 1: Delete all archived repositories (includes repos archived in the past)**

```shell
# Scope needed to delete repositories (skip if already granted)
gh auth refresh -h github.com -s delete_repo

# Every archived repository, whenever it was archived
gh repo list "$ORG" --limit 5000 --json name,isArchived \
  --jq '.[] | select(.isArchived == true) | .name' | sort -u > all-archived-repositories.txt

# Names to keep permanently (one per line; leave the file empty to keep nothing)
touch keep-forever-repos.txt
grep -vxFf keep-forever-repos.txt all-archived-repositories.txt > repos-to-delete.txt

echo "Archived repositories found : $(wc -l < all-archived-repositories.txt | tr -d ' ')"
echo "Selected for deletion       : $(wc -l < repos-to-delete.txt | tr -d ' ')"

# Forks, stars, and last push of each repository selected for deletion
jq -R . repos-to-delete.txt | jq -s . > delete-names.json
gh repo list "$ORG" --limit 5000 --json name,visibility,forkCount,stargazerCount,pushedAt \
  | jq -r --slurpfile sel delete-names.json '
      .[] | select(.name as $n | $sel[0] | index($n))
          | [ .name, .visibility, ("forks=" + (.forkCount|tostring)),
              ("stars=" + (.stargazerCount|tostring)),
              ("last_push=" + ((.pushedAt // "never")[0:10])) ] | @tsv
    ' | column -t -s $'\t'

# Repositories with NO saved mirror in this backup folder (their content will be lost)
echo "Selected repositories without a saved mirror:"
while IFS= read -r repo; do
  [ -d "mirrors/$repo.git" ] || echo "  $repo"
done < repos-to-delete.txt
```

Dry run (set to `false` after reviewing the output):

```shell
DRY_RUN=true    # change to false only after reviewing the dry-run output
N=$(wc -l < repos-to-delete.txt | tr -d ' ')

if [ "$DRY_RUN" != true ]; then
  read -r -p "Type the organization name ($ORG) to delete $N repositories: " ANSWER
  [ "$ANSWER" = "$ORG" ] || { echo "Confirmation failed; the loop below will run as a dry run only."; DRY_RUN=true; }
fi

while IFS= read -r repo; do
  if [ "$DRY_RUN" = true ]; then
    echo "[dry-run] WOULD DELETE: $ORG/$repo"
  else
    echo "Deleting $ORG/$repo"
    if gh repo delete "$ORG/$repo" --yes; then
      printf 'DELETED\t%s\n' "$repo" | tee -a delete-log.tsv
    else
      printf 'FAILED\t%s\n' "$repo" | tee -a delete-log.tsv
    fi
    sleep 1
  fi
done < repos-to-delete.txt
```

**Option 2:**

```shell
DRY_RUN=true    # change to false only after reviewing the dry-run output
N=$(wc -l < repos-to-delete.txt | tr -d ' ')

if [ "$DRY_RUN" != true ]; then
  read -r -p "Type the organization name ($ORG) to delete $N repositories: " ANSWER
  [ "$ANSWER" = "$ORG" ] || { echo "Confirmation failed; the loop below will run as a dry run only."; DRY_RUN=true; }
fi

while IFS= read -r repo; do
  if [ "$DRY_RUN" = true ]; then
    echo "[dry-run] WOULD DELETE: $ORG/$repo"
  else
    echo "Deleting $ORG/$repo"
    if gh repo delete "$ORG/$repo" --yes; then
      printf 'DELETED\t%s\n' "$repo" | tee -a delete-log.tsv
    else
      printf 'FAILED\t%s\n' "$repo" | tee -a delete-log.tsv
    fi
    sleep 1
  fi
done < repos-to-delete.txt
```

After reviewing the dry run, set `DRY_RUN=false` and run the same block again. A `404` with a scope message means the `delete_repo` scope is missing (repeat step 8.1). The organization may also restrict deletion to owners.

## 11. Verify the Deletion

```shell
gh repo list "$ORG" --limit 5000 --json name --jq '.[].name' | sort > remaining-repositories.txt

echo "Deleted repositories that still exist (should print nothing):"
comm -12 <(sort repos-to-delete.txt) remaining-repositories.txt

echo "Selected: $(wc -l < repos-to-delete.txt | tr -d ' ')" \
     " deleted per log: $(grep -c '^DELETED' delete-log.tsv)"
```

---

## Restoring from the Backup

**Unarchive a repository** (Part A only):

```shell
gh repo unarchive "$ORG/REPO-NAME" --yes
```

**Recreate a deleted repository from its mirror.** Replace `REPO-NAME`. This restores the branches and tags (the full commit history). It does not restore issues, pull requests, or settings. The saved JSON files in `issues-prs/` are a reference copy only.

```shell
REPO="REPO-NAME"
gh repo create "$ORG/$REPO" --private
git -C "mirrors/$REPO.git" push "https://github.com/$ORG/$REPO.git" --all
git -C "mirrors/$REPO.git" push "https://github.com/$ORG/$REPO.git" --tags
```

The mirror is pushed with `--all` and `--tags` rather than `--mirror`, because GitHub rejects the hidden pull-request references that a mirror clone contains.

---

## Final Checklist

- [ ] The backup folder is outside any Git repository (step 1.5 did not print `STOP`).
- [ ] The candidate list was reviewed by a person, and `keep-repos.txt` reflects that review.
- [ ] Repositories with recent issue or pull request activity, Pages sites, or template use were considered.
- [ ] Every line in `backup-verification.tsv` is `OK` or `EMPTY`, and a second copy of the backup exists.
- [ ] Archiving was run as a dry run first, then for real, and the verification counts agree.
- [ ] Before any deletion: forks of private repositories and Pages sites were reviewed (step 9).
- [ ] Deletion was run as a dry run first, and only repositories with a verified backup were included.
- [ ] Coursework or assessment evidence that must be retained under your institution's policy has been preserved before deletion.
- [ ] The backup folder is stored securely, is not committed to a repository, and has a deletion date.
