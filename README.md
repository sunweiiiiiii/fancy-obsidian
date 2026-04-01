# fancy-obsidian

Private markdown notes vault, synchronized across **iOS**, **Ubuntu**, and **Windows** via Git.

## How It Works

This vault uses the [Obsidian Git](https://github.com/Vinzent03/obsidian-git) community plugin to automatically commit and sync notes to this GitHub repository. Notes are kept private through the repository's access settings.

## Setup Instructions

### Prerequisites

- [Obsidian](https://obsidian.md/download) installed on your device
- A GitHub account with access to this repository
- Git installed on desktop platforms (Ubuntu/Windows)

---

### Windows

1. **Install Git** – Download and install from [git-scm.com](https://git-scm.com/download/win).
2. **Clone this repository:**
   ```powershell
   git clone https://github.com/sunweiiiiiii/fancy-obsidian.git
   ```
3. **Open the vault in Obsidian** – Launch Obsidian, choose *Open folder as vault*, and select the cloned `fancy-obsidian` folder.
4. **Install the Obsidian Git plugin** – Go to *Settings → Community Plugins → Browse*, search for **Obsidian Git**, install and enable it.
5. **Authenticate with GitHub** – The plugin will use Git's credential manager. On first push, Windows will prompt you to sign in with your GitHub credentials (or use a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-safe/managing-your-personal-access-tokens)).
6. **Sync** – The vault will auto-commit and push every 10 minutes. You can also trigger a manual sync via the command palette (*Ctrl+P → Obsidian Git: Create backup*).

---

### Ubuntu (Linux)

1. **Install Git:**
   ```bash
   sudo apt update && sudo apt install git
   ```
2. **Configure Git credentials** (recommended — stores credentials securely):
   ```bash
   git config --global credential.helper store
   ```
3. **Clone this repository:**
   ```bash
   git clone https://github.com/sunweiiiiiii/fancy-obsidian.git
   ```
4. **Open the vault in Obsidian** – Launch Obsidian, choose *Open folder as vault*, and select the `fancy-obsidian` folder.
5. **Install the Obsidian Git plugin** – Go to *Settings → Community Plugins → Browse*, search for **Obsidian Git**, install and enable it.
6. **Authenticate with GitHub** – Use a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-safe/managing-your-personal-access-tokens) as your password when prompted, or configure SSH:
   ```bash
   # Generate an SSH key and add the public key to your GitHub account
   ssh-keygen -t ed25519 -C "your_email@example.com"
   # Then use the SSH clone URL instead:
   git clone git@github.com:sunweiiiiiii/fancy-obsidian.git
   ```
7. **Sync** – The vault will auto-commit and push every 10 minutes.

---

### iOS

1. **Install Obsidian** from the [App Store](https://apps.apple.com/app/obsidian-connected-notes/id1557175442).
2. **Install the Obsidian Git plugin** – Go to *Settings → Community Plugins*, enable community plugins, browse for **Obsidian Git**, install and enable it.
3. **Create a GitHub Personal Access Token** – On GitHub, go to *Settings → Developer settings → Personal access tokens → Tokens (classic)* and generate a token with `repo` scope. Copy it somewhere safe.
4. **Clone this repository inside Obsidian:**
   - Open the command palette and run *Obsidian Git: Clone an existing remote repo*.
   - Enter the repository URL: `https://github.com/sunweiiiiiii/fancy-obsidian.git`
   - When prompted for credentials, use your GitHub **username** and the **Personal Access Token** as the password.
   - Choose a vault name/location and open the vault.
5. **Sync** – The vault will auto-pull on launch and auto-commit every 10 minutes.

> **Note for iOS:** Obsidian Git on iOS stores the vault inside Obsidian's local storage (iCloud is not required). Make sure *Settings → Community Plugins → Obsidian Git* shows the plugin is enabled after cloning.

---

## Sync Settings

The following auto-sync behaviour is pre-configured in `.obsidian/plugins/obsidian-git/data.json`:

| Setting | Value |
|---|---|
| Auto-commit interval | 10 minutes |
| Auto-pull interval | 10 minutes |
| Pull on startup | ✅ Enabled |
| Pull before push | ✅ Enabled |
| Conflict strategy | Merge |

You can adjust these in *Settings → Obsidian Git* at any time.

## What Is Synced

| Path | Synced? |
|---|---|
| `*.md` notes | ✅ Yes |
| `attachments/` | ✅ Yes |
| `.obsidian/app.json` | ✅ Yes |
| `.obsidian/appearance.json` | ✅ Yes |
| `.obsidian/community-plugins.json` | ✅ Yes |
| `.obsidian/core-plugins.json` | ✅ Yes |
| `.obsidian/plugins/obsidian-git/data.json` | ✅ Yes |
| `.obsidian/workspace.json` | ❌ No (device-specific) |
| `.obsidian/cache/` | ❌ No (device-specific) |
