# SSH Host Key Changed After Server OS Reinstallation

## A Real-World Troubleshooting Story

> One day I was doing SSH into a server that had been working perfectly.
> A few days earlier, the server's OS had been changed/reinstalled.
> Suddenly SSH stopped and showed
> **`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`**.
>
> This document explains what happened, why SSH did it, how to
> troubleshoot it safely, and how to fix it without blindly disabling
> SSH security.

------------------------------------------------------------------------

## 1. The Story

I tried to connect to the server:

``` bash
ssh ubuntu@10.121.220.21
```

Previously, this had worked normally.

Then the server's operating system was changed/reinstalled.

When I tried SSH again, I received:

``` text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

SSH also warned that somebody could potentially be performing a
man-in-the-middle attack.

That sounded serious.

But there was an important clue:

**The server's OS had recently been changed.**

That meant the SSH server could have generated a **new host key**.

My computer still remembered the **old host key** for `10.121.220.21`.

So SSH effectively saw:

``` text
Previously remembered:
10.121.220.21 → OLD HOST KEY

Currently presented:
10.121.220.21 → NEW HOST KEY
```

The keys did not match.

SSH stopped the connection.

That is actually a security feature working correctly.

------------------------------------------------------------------------

# 2. What Is an SSH Host Key?

An SSH server has cryptographic **host keys**.

They identify the SSH server itself.

This is different from the credentials used to authenticate a user.

Think of SSH as asking two separate questions:

``` text
1. Am I talking to the correct server?
        ↓
   Server host key

2. Is this user allowed to log in?
        ↓
   User authentication
```

The host key is about **server identity**.

------------------------------------------------------------------------

# 3. Why Does SSH Remember the Host Key?

The SSH client stores known server identities in:

``` text
~/.ssh/known_hosts
```

For example, when SSH is being run as `root`:

``` text
/root/.ssh/known_hosts
```

For a normal user such as `prabhakar`, it would normally be:

``` text
/home/prabhakar/.ssh/known_hosts
```

The important distinction is:

> `known_hosts` belongs to the **SSH client**, not the remote SSH
> server.

It contains information that lets the client recognize servers it has
connected to before.

------------------------------------------------------------------------

# 4. What Changed When the OS Was Reinstalled?

Before the rebuild:

``` text
My Machine
    |
    | ssh ubuntu@10.121.220.21
    v
Server
    |
    +-- Old OS
    |
    +-- Old SSH host key
```

The local machine had remembered:

``` text
10.121.220.21 → OLD KEY
```

Then the server was rebuilt:

``` text
My Machine
    |
    | ssh ubuntu@10.121.220.21
    v
Server
    |
    +-- New OS
    |
    +-- New SSH host key
```

The server now presented a different cryptographic identity.

SSH compared:

``` text
Stored key != Received key
```

and stopped.

------------------------------------------------------------------------

# 5. Why Does SSH Stop Instead of Automatically Trusting the New Key?

Because the same behavior can indicate a real attack.

Imagine:

``` text
Your Machine
      |
      v
   Attacker
      |
      v
 Real Server
```

If the attacker impersonates the server, the SSH client could receive a
different host key.

If SSH automatically trusted every new key, an attacker could
potentially hide behind the changed identity.

Instead, SSH says:

> The server I am talking to does not have the identity I previously
> recorded. Investigate before continuing.

Therefore:

``` text
Changed host key
       ≠
Definitely hacked
```

Instead:

``` text
Changed host key
       ↓
Investigate the reason
       ↓
Legitimate change?
or
Potential security problem?
```

------------------------------------------------------------------------

# 6. Legitimate Reasons a Host Key Can Change

Common legitimate causes include:

-   OS reinstallation
-   Server rebuild
-   VM recreation
-   SSH host keys regenerated
-   Server replacement
-   IP address reassigned to another machine
-   SSH configuration changes involving host keys

A malicious cause can also exist, such as a **man-in-the-middle
attack**.

Therefore, never blindly ignore the warning.

------------------------------------------------------------------------

# 7. The Warning We Saw

The SSH output included information similar to:

``` text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

and:

``` text
Offending ECDSA key in /root/.ssh/known_hosts:3
```

This means SSH found a conflicting stored host key.

The `:3` means the relevant entry was on **line 3** of:

``` text
/root/.ssh/known_hosts
```

The warning may end with:

``` text
Host key verification failed.
```

That means SSH refused to continue because it could not safely verify
the remote server's identity.

------------------------------------------------------------------------

# 8. First Rule: Do Not Blindly Disable Verification

A tempting workaround is:

``` bash
ssh -o StrictHostKeyChecking=no ubuntu@10.121.220.21
```

Do not treat that as the normal solution.

The warning is telling you that something changed.

The correct approach is:

``` text
Read warning
    ↓
Understand why key changed
    ↓
Verify legitimate change
    ↓
Remove stale local key
    ↓
Reconnect
    ↓
Verify new fingerprint
```

Especially on company or production infrastructure, do not bypass the
security check just to make the connection work.

------------------------------------------------------------------------

# 9. Troubleshooting From the Beginning

## Step 1 --- Confirm Which Local User Is Running SSH

Run:

``` bash
whoami
```

Why?

Because the location of `known_hosts` depends on the local account.

For example:

``` text
root
```

means:

``` text
/root/.ssh/known_hosts
```

A normal user may have:

``` text
/home/prabhakar/.ssh/known_hosts
```

------------------------------------------------------------------------

## Step 2 --- Find the Existing Host-Key Entry

Run:

``` bash
ssh-keygen -F 10.121.220.21
```

### What does `-F` mean?

It searches the known_hosts database for a particular host.

So:

``` bash
ssh-keygen -F 10.121.220.21
```

means:

> Find the stored SSH host-key information for `10.121.220.21`.

This is a **read-only diagnostic command**.

It does not change the remote server.

------------------------------------------------------------------------

# 10. Understanding the `known_hosts` Entry

A simplified entry can look conceptually like:

``` text
10.121.220.21 ssh-ed25519 AAAA...
```

You may encounter different host-key algorithms, such as:

``` text
ssh-ed25519
ecdsa-sha2-nistp256
rsa-sha2-512
```

You do not need to memorize the algorithms for this troubleshooting
scenario.

The important mental model is:

``` text
Host/IP
   +
Server host key
   ↓
stored in known_hosts
```

------------------------------------------------------------------------

# 11. Step 3 --- Confirm Why the Key Changed

Before deleting anything, ask:

> Why did this server's host key change?

In our case, the answer was already known:

**The server's OS had been changed/reinstalled.**

That makes a new host key expected.

For a company or production server, ideally confirm the change with the
administrator or the person who performed the rebuild.

For example:

> The SSH host key for `10.121.220.21` has changed. Was this server
> recently rebuilt or reinstalled?

This is safer than simply accepting the new identity.

------------------------------------------------------------------------

# 12. Step 4 --- Remove the Stale Local Host Key

After confirming that the server was legitimately rebuilt, run:

``` bash
ssh-keygen -R 10.121.220.21
```

### What does `-R` mean?

`-R` removes keys associated with the specified host from the local
known_hosts file.

Breakdown:

``` text
ssh-keygen
    |
    +-- -R
    |     Remove host entry
    |
    +-- 10.121.220.21
          Target host
```

This changes the **local client's** SSH trust database.

It does not:

-   change the remote OS
-   remove SSH from the remote server
-   regenerate the server's key
-   modify the remote server's configuration

It removes the stale local record.

------------------------------------------------------------------------

# 13. The Command We Used in the Incident

The terminal showed an explicit version:

``` bash
ssh-keygen -f '/root/.ssh/known_hosts' -R '10.121.220.21'
```

Breakdown:

``` text
ssh-keygen
    |
    +-- -f '/root/.ssh/known_hosts'
    |       |
    |       +-- explicitly specify the known_hosts file
    |
    +-- -R '10.121.220.21'
            |
            +-- remove this host's stored key
```

The shorter command is normally enough when SSH is using the expected
configuration:

``` bash
ssh-keygen -R 10.121.220.21
```

------------------------------------------------------------------------

# 14. Step 5 --- Verify the Old Entry Is Gone

Run:

``` bash
ssh-keygen -F 10.121.220.21
```

The old stored entry should no longer be present.

Now the client is ready to encounter the server's new host key.

Conceptually:

``` text
Before:

known_hosts
     |
     +-- old key for 10.121.220.21


After ssh-keygen -R:

known_hosts
     |
     +-- no old key for 10.121.220.21
```

------------------------------------------------------------------------

# 15. Step 6 --- Connect Again

Now run:

``` bash
ssh ubuntu@10.121.220.21
```

SSH should now see the host as one whose current key is not known
locally.

You may see:

``` text
The authenticity of host '10.121.220.21' can't be established.

ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxx.

Are you sure you want to continue connecting
(yes/no/[fingerprint])?
```

This is different from the original situation.

### Original situation

``` text
Known host
    ↓
Old key stored
    ↓
New key received
    ↓
Mismatch
    ↓
BLOCK
```

### After removing the stale entry

``` text
No stored key
    ↓
New key received
    ↓
SSH asks whether to trust it
```

------------------------------------------------------------------------

# 16. Step 7 --- Verify the Fingerprint

The fingerprint is a compact representation of the server's host key.

For example:

``` text
SHA256:U0GkmvdelE1QhF+cC/xAR32u/D+06muey2n1cf...
```

Conceptually:

``` text
Long cryptographic host key
          ↓
      fingerprint
          ↓
Compact identity representation
```

For important company/production systems, verify the fingerprint through
a trusted source when possible.

Examples:

-   administrator
-   approved infrastructure documentation
-   trusted server console
-   server inventory
-   information supplied during the rebuild

If the fingerprint is verified, it is safe to accept the new key.

------------------------------------------------------------------------

# 17. Step 8 --- Accept the New Key

After verification:

``` text
yes
```

SSH will store the new host key locally.

The trust relationship becomes:

``` text
Server
  |
  +-- New host key
  |
  v
SSH client
  |
  +-- verified/trusted
  |
  v
known_hosts
```

Future connections should then work normally.

------------------------------------------------------------------------

# 18. Step 9 --- Verify the Rebuilt Server

Because the server was recently rebuilt, verify that you actually
reached the expected machine.

After login:

``` bash
hostname
```

Then:

``` bash
whoami
```

Then:

``` bash
cat /etc/os-release
```

These answer:

``` text
hostname
    ↓
Am I on the expected server?

whoami
    ↓
Which account am I using?

/etc/os-release
    ↓
Which operating system/version is installed?
```

This is particularly useful after an OS replacement.

------------------------------------------------------------------------

# 19. Complete Runbook

When you encounter:

``` text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

use this workflow:

### 1. Confirm local user

``` bash
whoami
```

### 2. Find the stored host key

``` bash
ssh-keygen -F 10.121.220.21
```

### 3. Confirm why the server key changed

In this incident:

``` text
Server OS was changed/reinstalled
```

### 4. Remove the stale local entry

``` bash
ssh-keygen -R 10.121.220.21
```

### 5. Verify removal

``` bash
ssh-keygen -F 10.121.220.21
```

### 6. Reconnect

``` bash
ssh ubuntu@10.121.220.21
```

### 7. Verify the new fingerprint

Use a trusted source where appropriate.

### 8. Accept the new key

``` text
yes
```

### 9. Verify the server

``` bash
hostname
whoami
cat /etc/os-release
```

------------------------------------------------------------------------

# 20. Command Safety Classification

  ---------------------------------------------------------------------------------------
  Command                                 Classification          Purpose
  --------------------------------------- ----------------------- -----------------------
  `whoami`                                🟢 Read-only            Shows the current local
                                                                  user

  `hostname`                              🟢 Read-only            Shows hostname

  `ssh-keygen -F <host>`                  🟢 Read-only            Finds stored host-key
                                                                  entries

  `ssh-keygen -R <host>`                  🟡 Local modification   Removes a stale local
                                                                  known_hosts entry

  `ssh user@host`                         🟢 Connection           Attempts an SSH
                                                                  connection

  `cat /etc/os-release`                   🟢 Read-only            Identifies the remote
                                                                  OS

  `ssh -o StrictHostKeyChecking=no ...`   ⚠️ Avoid as a blanket   Bypasses an important
                                          fix                     verification step
  ---------------------------------------------------------------------------------------

The important subtlety is that `ssh-keygen -R` modifies something, but
it modifies only the **local known_hosts file**.

------------------------------------------------------------------------

# 21. Common Mistakes

## Mistake 1 --- "The server must be hacked."

Not necessarily.

Correct reasoning:

``` text
Host key changed
      ↓
Investigate
      ↓
Could be:
- OS rebuild
- VM recreation
- key regeneration
- IP reassignment
- security issue
```

------------------------------------------------------------------------

## Mistake 2 --- Blindly typing `yes`

If you do not know why the key changed, investigate first.

------------------------------------------------------------------------

## Mistake 3 --- Disabling host-key checking

Do not use:

``` bash
-o StrictHostKeyChecking=no
```

as a general solution.

It can hide the exact security problem SSH is warning about.

------------------------------------------------------------------------

## Mistake 4 --- Deleting the entire `known_hosts` file

Avoid doing:

``` bash
rm ~/.ssh/known_hosts
```

just to fix one server.

That can remove trusted identities for many other servers.

Prefer:

``` bash
ssh-keygen -R 10.121.220.21
```

which targets the specific host.

------------------------------------------------------------------------

# 22. Production Mental Model

A server rebuild changes much more than just the operating system files.

Think:

``` text
Server rebuild
     |
     +-- OS
     +-- SSH host keys
     +-- packages
     +-- services
     +-- configuration
     +-- users
     +-- network configuration
     +-- application configuration
     +-- certificates
     +-- possibly data
```

Therefore, when a server has been rebuilt, old client-side trust
information can become stale.

SSH host-key mismatch is one example.

------------------------------------------------------------------------

# 23. Why This Matters for Your Linux Administration Work

This incident connects several Linux administration concepts:

``` text
SSH
 |
 +-- SSH client
 |
 +-- SSH server
 |
 +-- Server identity
 |
 +-- Host keys
 |
 +-- known_hosts
 |
 +-- Fingerprints
 |
 +-- Authentication
 |
 +-- Security verification
 |
 +-- Server rebuilds
```

This is exactly the kind of small operational issue that appears in real
infrastructure work.

The command itself is easy.

The important skill is **understanding why SSH stopped and knowing how
to fix it without weakening security**.

------------------------------------------------------------------------

# 24. Interview / Viva Questions

### Q1. What is `known_hosts`?

A local SSH client file that stores host-key information for remote SSH
servers the client has previously encountered.

### Q2. Why does SSH warn when a host key changes?

Because a changed host key can indicate either a legitimate
infrastructure change or a potential security problem such as a
man-in-the-middle attack.

### Q3. Why can reinstalling an OS change an SSH host key?

A fresh SSH installation can generate new server host keys.

### Q4. What does this do?

``` bash
ssh-keygen -R 10.121.220.21
```

It removes the locally stored known_hosts entries associated with that
host.

### Q5. Does `ssh-keygen -R` change the remote server?

No. It modifies the local SSH client's known_hosts database.

### Q6. What does this do?

``` bash
ssh-keygen -F 10.121.220.21
```

It searches the known_hosts database for that host.

### Q7. Is a changed host key proof of an attack?

No. It is a warning that requires investigation.

### Q8. Why should we verify the new fingerprint?

Because we want to make sure the new key belongs to the intended server
before trusting it.

### Q9. Why shouldn't we simply disable `StrictHostKeyChecking`?

Because host-key verification is an important defense against connecting
to an unexpected or impersonating server.

------------------------------------------------------------------------

# 25. The Story in One Diagram

``` text
                 ONE DAY...

       I tried to SSH into the server
                    |
                    v
       ssh ubuntu@10.121.220.21
                    |
                    v
        SSH checked known_hosts
                    |
                    v
       Old key was already stored
                    |
                    v
      Server presented a NEW key
                    |
                    v
          OLD KEY != NEW KEY
                    |
                    v
      WARNING: REMOTE HOST
       IDENTIFICATION CHANGED
                    |
                    v
       "Why did the key change?"
                    |
                    v
       Server OS had been changed
                    |
                    v
       New SSH host key was created
                    |
                    v
       Legitimate change confirmed
                    |
                    v
     ssh-keygen -R 10.121.220.21
                    |
                    v
        Old local key removed
                    |
                    v
       SSH connection attempted again
                    |
                    v
       New fingerprint presented
                    |
                    v
        Fingerprint verified
                    |
                    v
                yes
                    |
                    v
              SSH LOGIN
                    |
                    v
        hostname / whoami / OS check
                    |
                    v
             CONNECTION FIXED
```

------------------------------------------------------------------------

# 26. Final Revision Card

``` text
PROBLEM
-------
SSH says:
REMOTE HOST IDENTIFICATION HAS CHANGED


LIKELY CAUSES
-------------
- OS reinstall
- Server rebuild
- VM recreation
- SSH host key regeneration
- IP reassignment
- Possible MITM attack


CHECK STORED KEY
----------------
ssh-keygen -F <host>


INVESTIGATE WHY IT CHANGED
--------------------------
Was the server rebuilt/reinstalled?


REMOVE STALE LOCAL ENTRY
------------------------
ssh-keygen -R <host>


RECONNECT
---------
ssh user@<host>


VERIFY NEW FINGERPRINT
----------------------
Compare with a trusted source when appropriate.


AFTER LOGIN
-----------
hostname
whoami
cat /etc/os-release


IMPORTANT
---------
Do NOT blindly disable SSH host-key verification.
```

------------------------------------------------------------------------

# 27. What We Just Demonstrated

This incident demonstrated an important Linux/SSH principle:

> **SSH verifies not only the user, but also the identity of the
> server.**

The `known_hosts` file lets the SSH client remember previously
encountered server identities.

When the server's OS was changed, the SSH host key changed.

The client detected:

``` text
Expected identity
       ≠
Presented identity
```

and stopped the connection.

After confirming that the server had legitimately been rebuilt, we
removed the stale local identity and established trust with the new host
key.

The key lesson is:

> **Do not treat the SSH warning as an obstacle to bypass. Treat it as a
> diagnostic and security signal.**

------------------------------------------------------------------------

# Current Knowledge

## Completed

-   SSH client vs SSH server
-   SSH host keys
-   `known_hosts`
-   Host-key fingerprints
-   Host-key mismatch
-   `ssh-keygen -F`
-   `ssh-keygen -R`
-   Safe host-key troubleshooting
-   Verifying a rebuilt server after login

## Future / Related Topics

-   SSH key-based user authentication
-   `authorized_keys`
-   `sshd_config`
-   SSH hardening
-   SSH troubleshooting in depth
-   Host-key algorithms
-   SSH certificates
