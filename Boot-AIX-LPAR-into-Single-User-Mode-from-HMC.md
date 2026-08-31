# Boot an AIX LPAR into Single-User Mode from HMC

This procedure boots an AIX logical partition from its existing `rootvg` into single-user mode. It does not require AIX installation media or a NIM server.

## When to use this procedure

Use single-user mode when normal AIX startup is blocked by a service or configuration problem, such as:

- TCP/IP or NIS startup failures
- NFS configuration problems
- Incorrect `/etc/inittab` entries
- Applications preventing the system from reaching a login prompt

> **Warning:** An immediate HMC shutdown is equivalent to a hard reset. Use a normal OS shutdown whenever the AIX shell is accessible.

## Placeholders

Replace these values throughout the procedure:

- `<managed-system>`: Power managed-system name on the HMC
- `<lpar-name>`: AIX partition name

## 1. Identify the managed system and LPAR

List managed systems:

```bash
lssyscfg -r sys -F name,state
```

List partitions on the appropriate managed system:

```bash
lssyscfg -r lpar -m <managed-system> -F name,lpar_id,state
```

## 2. Exit an existing HMC console

At the beginning of a new line, enter:

```text
~.
```

This closes only the HMC virtual-terminal session; it does not shut down the LPAR.

## 3. Stop the LPAR

If a normal AIX shutdown is unavailable because the boot process is stuck:

```bash
chsysstate -r lpar -m <managed-system> \
  -o shutdown --immed \
  -n <lpar-name>
```

Older HMC documentation may show `-o off --immed`. On newer HMC releases, `-o off` is deprecated for LPARs; use `-o shutdown --immed`.

Confirm the partition is stopped:

```bash
lssyscfg -r lpar -m <managed-system> \
  --filter "lpar_names=<lpar-name>" \
  -F name,lpar_id,state
```

Expected state:

```text
<lpar-name>,<lpar-id>,Not Activated
```

## 4. Activate in diagnostic mode

Activate the LPAR using the diagnostic stored boot list:

```bash
chsysstate -r lpar -m <managed-system> \
  -o on \
  -n <lpar-name> \
  -b ds
```

The `-b ds` option means **Diagnostic mode using the stored boot list**. AIX boots from the existing system disk.

## 5. Open the HMC console

```bash
mkvterm -m <managed-system> -p <lpar-name>
```

Wait for:

```text
DIAGNOSTIC OPERATING INSTRUCTIONS
```

Press **Enter**.

## 6. Select single-user mode

At the **FUNCTION SELECTION** menu, select:

```text
5. Single User Mode
```

On older diagnostic menus, select **Task Selection**, then locate **Single User Mode**.

AIX continues booting and displays:

```text
INIT: SINGLE-USER MODE
Password:
```

Enter the local `root` password. A successful login produces a root shell.

## 7. Enable 64-bit commands if necessary

Some commands may report that the 64-bit environment has not been configured. Enable it with:

```bash
/etc/methods/cfg64
```

## 8. Perform the repair

Back up every configuration file before editing it. After making changes, use an appropriate syntax check when available.

Example for a Korn shell startup script:

```bash
cp -p /path/to/file /path/to/file.before-change
ksh -n /path/to/file
echo $?
```

A result of `0` indicates valid shell syntax.

## 9. Shut down single-user mode cleanly

```bash
sync
sync
shutdown -F
```

Wait for:

```text
....Halt completed....
```

Exit the console:

```text
~.
```

## 10. Activate in normal mode

Confirm the LPAR is `Not Activated`, then run:

```bash
chsysstate -r lpar -m <managed-system> \
  -o on \
  -n <lpar-name> \
  -b norm
```

Reconnect to monitor startup:

```bash
mkvterm -m <managed-system> -p <lpar-name>
```

## Console escape note

When an interactive session such as `clogin` is nested inside an HMC virtual terminal:

- `~.` exits the outer HMC virtual terminal.
- `~~.` passes `~.` through the HMC session to the nested session.
- `exit` is preferred when a normal shell prompt is available.
