# Linux Filesystem: 10 Beginner Exercises

**Phase 2: Linux basics** · Topic block: Filesystem · Sandbox: Ubuntu VM on Hyper-V

## Ground rules

- Work only inside `~/practice`. Nothing important can break there.
- Errors are the job, not a failure. Read the message, then fix it.
- Run `pwd` before every `rm`.
- Never put real passwords or company data into practice files.
- Pace: about 1 exercise per session, roughly 2 weeks in total.

## 1. Orient yourself on a new server

- **Scenario:** You SSH into a server for the first time. Where are you?
- **Outcome:** Run `pwd`, `ls`, `ls -la`, `cd ~`.
- **Done when:** You can say where you are and spot at least one hidden file (starts with `.`).

## 2. Explore the Linux map

- **Scenario:** A senior says "check the logs in /var and the config in /etc".
- **Outcome:** Visit `/`, `/etc`, `/var/log`, `/home`, `/tmp` with `cd` and `ls`.
- **Done when:** You wrote one sentence per folder on what it is for.

## 3. Build a project folder structure

- **Scenario:** Set up a workspace for a new app.
- **Outcome:** Create `~/practice/myapp/{config,logs,scripts,backup}` with `mkdir -p`.
- **Done when:** `ls -R ~/practice` shows the full tree.

## 4. Create and inspect files

- **Scenario:** Prepare a config file and a log file.
- **Outcome:** Create `app.conf` and `app.log` with `touch`, add text with `echo "..." > file`, read with `cat`.
- **Done when:** `cat` shows your text in both files.

## 5. Copy before you change (backup habit)

- **Scenario:** Never edit a config without a backup.
- **Outcome:** Run `cp config/app.conf backup/app.conf.bak`.
- **Done when:** `ls backup/` shows the backup file.

## 6. Move and rename

- **Scenario:** A log file is in the wrong folder and has a messy name.
- **Outcome:** Use `mv` to move it into `logs/` and rename it `app-2026-10-30.log`.
- **Done when:** `ls logs/` shows the new name and the old location is empty.

## 7. Find a lost file

- **Scenario:** "Where is that config file?" is a daily question.
- **Outcome:** Run `find ~/practice -name "*.conf"` and `find ~/practice -name "*.log"`.
- **Done when:** The output lists the correct paths.

## 8. Read logs without opening a whole file

- **Scenario:** A real log has thousands of lines. You only need the end.
- **Outcome:** Fill a file with 20 lines, then try `head -n 5`, `tail -n 5`, and `wc -l`.
- **Done when:** You can explain what each command showed.

## 9. Permissions: who can touch what

- **Scenario:** A script will not run: "Permission denied". Classic.
- **Outcome:** Create `scripts/hello.sh`, check it with `ls -l`, try to run it, then fix it with `chmod +x`.
- **Done when:** The script prints "Hello" after the fix.

## 10. Safe cleanup (delete with care)

- **Scenario:** Clear out old test data without wiping the wrong thing.
- **Outcome:** Delete one file with `rm`, then the whole `~/practice` folder with `rm -r`, after checking with `pwd` and `ls` first.
- **Done when:** `ls ~/practice` says "No such file or directory".

## Good-practice tips

- Check `pwd` before any `rm`.
- Never run `rm -rf` with `sudo` until you fully understand what it targets.
- Backup first, edit second.
- Keep practice data fake. Real secrets never go into files or Git.

## Mini-project idea

- **Set up, back up, and organize a mock app workspace.**
- Chains exercises 3 to 6.
- Commit your session files to `devops-learning/phase-2-linux/week-01/`.
