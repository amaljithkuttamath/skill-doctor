---
name: rollback
description: Restore skills from the last skill-doctor backup. Use if upgraded skills cause problems after migration.
disable-model-invocation: true
allowed-tools: Read, Bash, Glob
---

## Find Backup

```bash
ls -d ~/.skill-doctor-backup-* 2>/dev/null | sort | tail -1
```

If no backups found: "No skill-doctor backups found. Nothing to roll back."

---

## Show Manifest

Read `manifest.json` from the backup directory.

Show the user:
```
Backup from [timestamp]

Skills that will be restored:
- [skill-name] -> [original-path]
- [skill-name] -> [original-path]
```

Ask: "Restore these skills to their pre-upgrade state?"

---

## Restore

On confirmation:

For each entry in `manifest.json`:
```bash
cp -r $BACKUP_DIR/<skill-name>/* <original-path>/
```

After restoring all skills:

"Rolled back [N] skills to their pre-upgrade state.
The backup is preserved at `$BACKUP_DIR` in case you need it again."

Do NOT delete the backup directory. The user may want to reference it.

---

## Multiple Backups

If the user asks to roll back to a specific backup (not the latest):

```bash
ls -d ~/.skill-doctor-backup-* 2>/dev/null | sort
```

Show all available backups with timestamps. Let the user pick.

---

## Rules

- Only restore skills listed in the manifest. Don't touch anything else.
- Don't delete the backup after restoring.
- If a skill's original path no longer exists (user deleted it), warn and skip that skill.
