---
title: Hashing
last_reviewed: 2026-03-08
owner: Rognheim
---

# :fontawesome-solid-hashtag:


## **Hashing**

---

### **Introduction**

Hashing is used in this platform for two main tasks:

- Secure password storage (non-reversible hashes)
- Integrity verification for downloaded artifacts, images, and backups

Hashing is not encryption. Use hashing for verification and password derivation, and encryption for confidentiality.

---

### **Prerequisites and Scope**

Applies to:

- Identity/auth systems (Authelia user credentials)
- Debian VM image/bootstrap downloads
- Docker image and artifact verification workflows

Recommended defaults:

- Password hashing: Argon2id
- Integrity hashing: SHA-256

---

### **Configuration files**

- Authelia `configuration.yml` (password hashing algorithm settings)
- `user_database.yml` (hashed password values only)
- Download/bootstrap scripts where checksum verification is performed

---

### **Implementation**

#### 1. Password hashing policy (Argon2id)

Use Argon2id for all local credential stores that support it.  
Never store plaintext passwords in compose files, `.env`, or docs.

Generate a hash with Authelia tooling:

```shell
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'replace-with-strong-password'
```

Store only the hash output in `user_database.yml`.

#### 2. Integrity verification for downloads and images

When downloading cloud images or binaries, verify checksum before use:

```shell
sha256sum debian-13-generic-amd64.qcow2
```

Compare output against trusted upstream checksum files.

For generic file verification:

```shell
sha256sum -c SHA256SUMS
```

#### 3. Hash verification in provisioning flow

Before using artifacts in Proxmox/Cloud-Init bootstrap:

1. Download artifact and published checksum.
2. Verify checksum locally.
3. Import artifact only if checksum verification passes.
4. Record source URL and checksum in change notes.

#### 4. Avoid weak patterns

Do not use:

- MD5 for integrity security checks
- SHA-1 for security-sensitive integrity validation
- Unsalted fast hashes for passwords

Use strong password hashing (Argon2id, bcrypt, scrypt) depending on application support.

---

### **Operational Verification Checklist**

```shell
rg -n \"password:\\s*['\\\"]?[A-Za-z0-9!@#$%^&*()_+=-]+['\\\"]?$\" /opt/docker-appdata/compose-files -S
rg -n \"md5|sha1\" /opt/docker-appdata/compose-files /opt/docker-appdata/scripts -S
```

Expected outcome:

- No plaintext passwords in compose-managed stacks
- No weak hash algorithms in active security-sensitive workflows

---

### **Post-install tasks**

- Add checksum verification to all manual download procedures.
- Standardize password hash generation commands in runbooks.
- Periodically review auth config to confirm Argon2id remains enabled.
- Include integrity verification steps in incident response for suspected tampering.

---

### **Known issues**

- Verifying checksums from untrusted channels defeats the purpose of hashing.
- Legacy applications may only support weaker algorithms and need compensating controls.
