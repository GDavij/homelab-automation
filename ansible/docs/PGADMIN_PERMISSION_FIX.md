# pgAdmin Permission Fix - Technical Documentation

## Problem Analysis

### Root Cause
The pgAdmin container fails to start with permission errors:
```
ERROR: Failed to create the directory /var/lib/pgadmin/sessions:
       [Errno 13] Permission denied: '/var/lib/pgadmin/sessions'
```

### Why This Happens
1. **Container User**: pgAdmin runs as `uid=5050(pgadmin)` and `gid=0(root)`
2. **Host Directory**: The mounted volume `/cold-storage/database/pgadmin/data` was created with wrong ownership (likely `root:root`)
3. **Volume Mount**: Docker mounts the host directory into the container at `/var/lib/pgadmin`
4. **Permission Mismatch**: The pgAdmin process (uid 5050) cannot write to a directory owned by root

## Solution: Idempotent Permission Fix

### Design Principles
1. ✅ **Idempotent** - Can be run multiple times without side effects
2. ✅ **Conditional** - Only makes changes when needed
3. ✅ **Safe** - Checks before modifying
4. ✅ **Minimal Disruption** - Only restarts container if permissions were actually fixed
5. ✅ **Verifiable** - Provides clear feedback on what was done

## Implementation Breakdown

### Step 1: Create Directory with Correct Ownership (Idempotent)
```yaml
- name: Ensure pgAdmin data directory exists with correct ownership
  ansible.builtin.file:
    path: "{{ PGADMIN_DATA_PATH }}"
    state: directory
    owner: "5050"
    group: "0"
    mode: '0750'
  become: true
  register: pgadmin_dir_created
```

**What it does:**
- Creates directory if missing
- Sets ownership to `uid=5050, gid=0` (pgAdmin's user)
- Sets permissions to `0750` (owner: rwx, group: r-x, others: none)

**Why it's idempotent:**
- `ansible.builtin.file` module checks current state
- Only reports "changed" if it actually modifies something
- If directory exists with correct ownership → no change
- If directory exists with wrong ownership → only fixes the directory itself (not recursive)

**Registers:** `pgadmin_dir_created` - tells us if the directory was created

---

### Step 2: Fix Ownership of Existing Files (Smart Check)
```yaml
- name: Fix ownership of existing pgAdmin data (only subdirectories/files)
  ansible.builtin.shell: |
    # Only fix ownership if directory has wrong permissions
    current_owner=$(stat -c '%u' "{{ PGADMIN_DATA_PATH }}" 2>/dev/null || echo "unknown")
    if [ "$current_owner" != "5050" ] || [ -n "$(find {{ PGADMIN_DATA_PATH }} ! -user 5050 2>/dev/null)" ]; then
      chown -R 5050:0 "{{ PGADMIN_DATA_PATH }}"
      echo "changed"
    else
      echo "ok"
    fi
  register: pgadmin_ownership_fix
  changed_when: "'changed' in pgadmin_ownership_fix.stdout"
```

**What it does:**
1. Checks directory ownership: `stat -c '%u'` gets the user ID
2. Searches for files with wrong ownership: `find ! -user 5050`
3. **Only if** wrong ownership detected → runs `chown -R`
4. Outputs "changed" or "ok" based on whether changes were made

**Why it's idempotent:**
- Checks BEFORE making changes
- If ownership is already correct → outputs "ok" → Ansible reports "ok" (not changed)
- If ownership needs fixing → outputs "changed" → Ansible reports "changed"
- `changed_when` directive tells Ansible to only mark as changed if "changed" appears in output

**Registers:** `pgadmin_ownership_fix.changed` - tells us if ownership was fixed

---

### Step 3: Check Container Status (Read-Only)
```yaml
- name: Check if pgAdmin container exists and is running
  ansible.builtin.shell: |
    docker inspect -f '{{.State.Running}}' pgadmin 2>/dev/null || echo "not_found"
  register: pgadmin_container_status
  changed_when: false
```

**What it does:**
- Queries Docker for container state
- Returns: "true" (running), "false" (stopped), or "not_found" (doesn't exist)

**Why it's idempotent:**
- Pure read operation → `changed_when: false`
- Never modifies anything
- Gathers intelligence for decision-making

**Registers:** `pgadmin_container_status.stdout` - container state

---

### Step 4: Conditional Restart (Only When Needed)
```yaml
- name: Restart pgAdmin container only if permissions were fixed
  ansible.builtin.shell: |
    docker restart pgadmin
  when:
    - PGADMIN_ENABLED | default(false)
    - pgadmin_ownership_fix.changed | default(false)
    - pgadmin_container_status.stdout == "true"
  register: pgadmin_restart_result
```

**What it does:**
- Restarts the container ONLY if ALL conditions are true:
  1. pgAdmin is enabled
  2. Permissions were actually fixed (changed == true)
  3. Container exists and is running

**Why it's idempotent:**
- **Conditional execution** - only runs when needed
- First run with wrong permissions → fixes ownership → restarts → "changed"
- Second run with correct permissions → no ownership fix → no restart → "ok"
- Third run → same as second → "ok"

**Why this matters:**
- Avoids unnecessary service disruptions
- Clear audit trail (restart = permissions were fixed)
- No false "changed" status on subsequent runs

**Registers:** `pgadmin_restart_result` - tells us if restart happened

---

### Step 5: Wait for Stability (Only After Restart)
```yaml
- name: Wait for pgAdmin to be ready after restart
  ansible.builtin.pause:
    seconds: 5
  when:
    - PGADMIN_ENABLED | default(false)
    - pgadmin_restart_result is changed
```

**What it does:**
- Pauses for 5 seconds ONLY if container was restarted
- Allows pgAdmin to initialize before continuing

**Why it's idempotent:**
- Only runs when `pgadmin_restart_result is changed`
- If no restart → no pause
- Prevents race conditions with verification steps

---

## Idempotency Proof

### First Run (Permissions Wrong)
```
Task 1: Create directory → CHANGED (created)
Task 2: Fix ownership → CHANGED (fixed files)
Task 3: Check container → OK (read-only)
Task 4: Restart container → CHANGED (restarted)
Task 5: Wait → RUN (pause)
Result: 3 tasks changed
```

### Second Run (Permissions Correct)
```
Task 1: Create directory → OK (exists, correct ownership)
Task 2: Fix ownership → OK (already correct)
Task 3: Check container → OK (read-only)
Task 4: Restart container → SKIPPED (ownership_fix.changed == false)
Task 5: Wait → SKIPPED (no restart)
Result: 0 tasks changed ✅
```

### Third Run (Permissions Still Correct)
```
Same as second run → 0 tasks changed ✅
```

## Verification

### Check Ownership
```bash
ansible homelab -i inventory.ini -m shell -a "ls -ld /cold-storage/database/pgadmin/data" --become
```
Expected: `drwxr-x--- ... 5050 root ... /cold-storage/database/pgadmin/data`

### Check Container Logs
```bash
ansible homelab -i inventory.ini -m shell -a "docker logs pgadmin --tail 30" --become
```
Expected: No "Permission denied" errors

### Check Container Status
```bash
ansible homelab -i inventory.ini -m shell -a "docker ps --filter 'name=pgadmin'" --become
```
Expected: Status "Up" with no restarts

## Running the Fix

### Option 1: Via Main Playbook (Recommended)
```bash
ansible-playbook server_playbook.yml -i inventory.ini --tags pgadmin --ask-become-pass
```

### Option 2: Standalone Fix Script
```bash
ansible-playbook fix_pgadmin_permissions.yml -i inventory.ini --ask-become-pass
```

## Key Takeaways

1. **Always Check Before Changing** - Use stat, find, or docker inspect before making changes
2. **Use `changed_when`** - Control when Ansible reports tasks as changed
3. **Register Variables** - Store results for conditional logic
4. **Conditional Execution** - Use `when:` with multiple conditions
5. **Avoid Recursive Operations** - Separate directory creation from file ownership
6. **Read-Only Checks** - Mark queries as `changed_when: false`
7. **Test Idempotency** - Run playbook 3 times → only first should change

## Common Anti-Patterns (Avoided)

❌ **Bad:** `recurse: yes` in file module → always traverses all files
✅ **Good:** Conditional shell script that checks first

❌ **Bad:** `docker restart || true` → always runs
✅ **Good:** Conditional restart based on previous task results

❌ **Bad:** `ignore_errors: yes` → hides real problems
✅ **Good:** Proper conditionals that prevent errors

❌ **Bad:** No verification after changes
✅ **Good:** Log checks and status verification

## Monitoring

After running the fix, monitor for 24 hours:
```bash
# Check for crashes
watch -n 5 'ansible homelab -i inventory.ini -m shell -a "docker ps -a --filter name=pgadmin"'

# Monitor logs
ansible homelab -i inventory.ini -m shell -a "docker logs -f pgadmin"
```

## Success Criteria

✅ Container stays running (no crash loops)
✅ No "Permission denied" errors in logs
✅ pgAdmin web UI accessible at https://pg.homelab
✅ Can login with configured credentials
✅ Can add PostgreSQL server connections
✅ Playbook shows 0 changes on second run
