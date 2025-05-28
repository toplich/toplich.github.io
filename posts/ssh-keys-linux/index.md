# How to Set Up SSH Keys on Linux


# 🔐 Setting Up SSH Keys on Linux (Step-by-Step Guide)

Using SSH keys is a safer and easier way to connect to Linux servers — no more typing your password every time. This guide will help you set up SSH keys, copy them to a remote server, and log in securely.

## ✅ Step 1: Check If You Already Have SSH Keys

Open your terminal and run:

```bash
ls ~/.ssh/id_*.pub
```

If you see a file like `id_ed25519.pub` or `id_rsa.pub`, it means you already have a key. You can use that one or create a new one.

## 🛠 Step 2: Create a New SSH Key Pair

To make a new SSH key, run:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

> On older systems, use `ssh-keygen -t rsa -b 4096` instead.

- Press **Enter** to accept the default location (`~/.ssh/id_ed25519`)
- You can choose to add a passphrase (or leave it empty)

This creates two files:
- `id_ed25519` → your **private key** (keep it secret!)
- `id_ed25519.pub` → your **public key** (you’ll share this)

## 📤 Step 3: Add Your Public Key to the Remote Server

### Option 1: Use `ssh-copy-id` (recommended)

```bash
ssh-copy-id username@remote_server_ip
```

This command adds your key to the remote server so you can log in without a password.

### Option 2: Manual method

1. Show your public key:

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

2. Copy the whole line

3. Connect to your server and paste it into:

   ```bash
   nano ~/.ssh/authorized_keys
   ```

## 🚀 Step 4: Log In Without a Password

Now try logging in:

```bash
ssh username@remote_server_ip
```

If everything is correct, you’ll connect **without entering a password** 🎉

## 🔒 Step 5 (Optional): Turn Off Password Login

This step is for extra security. Only do it if SSH keys are working!

1. Open SSH settings on the server:

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. Find and change these lines:

   ```text
   PasswordAuthentication no
   ChallengeResponseAuthentication no
   UsePAM no
   ```

3. Save and restart SSH:

   ```bash
   sudo systemctl restart sshd
   ```

## 💡 Helpful Tips

- Your **private key** must stay private (`~/.ssh/id_ed25519`)
- A **passphrase** adds extra protection
- Use an SSH config file to manage many servers:

Create or edit this file:

```bash
nano ~/.ssh/config
```

Add this block:

```text
Host myserver
  HostName 192.168.1.10
  User vitalii
  IdentityFile ~/.ssh/id_ed25519
```

Then you can connect by simply running:

```bash
ssh myserver
```

## 🧩 Summary

SSH keys help you:
- Log in faster
- Avoid typing passwords
- Stay secure

Now you're ready to manage your servers like a pro — safely and efficiently!


