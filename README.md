# Managing Multiple Cursor Instances on macOS

*Dante Bazaldua · 2026-01-14 · Source: https://stdout.mx/en/posts/cursor-sync-multiple-instances*

Tags: #cursor #macos #development #terminal #productivity

> A tale of two Cursors: How to manage separate work and personal Cursor instances while keeping them perfectly in sync

## TL;DR

Run multiple isolated Cursor instances (work/personal) on macOS with two shell functions: `picursor()` launches a separate Cursor instance, and `cursor-sync()` keeps both instances updated with the same app version, settings, and extensions. Complete isolation for accounts, but consistent configuration everywhere.

**What you get:**

- Separate Cursor sessions with different accounts
- Auto-sync: same extensions, settings, and keybindings
- One command to keep everything updated
- 5-minute setup

## Requirements

Before you begin, make sure you have:

- **macOS** (this guide is macOS-specific)
- **Cursor IDE** installed at `/Applications/Cursor.app`
- **Zsh shell** (default on modern macOS)
- **Basic terminal knowledge** (editing shell config files)
- **sudo access** (needed for copying the app)
- **rsync** (pre-installed on macOS)

## The Problem: A Developer's Dilemma

Picture this: You're a developer who loves using Cursor, the AI-powered code editor. You use it for work projects during the day, logged in with your company account. But when evening comes and you want to work on personal projects, you face a frustrating choice: log out and back in with your personal account, or compromise by mixing work and personal contexts.

Maybe you've tried switching between accounts manually. Maybe you've dealt with the hassle of losing your settings, extensions, and customizations every time you switch. Or perhaps you've simply given up and decided to keep everything in one account, sacrificing the clean separation between work and life.

There had to be a better way.

## The Journey to a Solution

The breakthrough came from [this Medium article](https://medium.com/@stasprigodich/how-to-set-up-multiple-cursor-accounts-on-macos-52f5f1fb65ec) that explored running multiple Cursor instances. But simply launching a separate instance wasn't enough. The real challenge was keeping both instances up to date without manually managing everything twice.

After some experimentation, I developed a two-function approach that solves both problems:

1. **`picursor()`** - Launch a completely isolated personal Cursor instance
2. **`cursor-sync()`** - Keep both instances in perfect sync

Let me walk you through how each piece works and why it matters.

## The Launch Function: picursor()

First, we need a way to launch a separate Cursor instance that won't interfere with the main one. Here's the function:

```bash
picursor() {
  # Launch with full isolation to prevent session sharing
  # Using --new-window ensures it opens in the correct isolated instance
  nohup "/Applications/Cursor-Personal.app/Contents/MacOS/Cursor" \
    --user-data-dir="$HOME/.cursor-personal" \
    --extensions-dir="$HOME/.cursor-personal/extensions" \
    --no-first-run \
    --new-window \
    --disable-dev-shm-usage \
    "$@" >/dev/null 2>&1 &
}
```

### Breaking It Down

Let's examine what each flag does and why it's necessary:

**`--user-data-dir="$HOME/.cursor-personal"`**

This is the heart of the isolation. By specifying a separate user data directory, we ensure that all settings, preferences, login sessions, and cache are stored completely separately from the main Cursor instance. Your work and personal accounts never cross paths.

**`--extensions-dir="$HOME/.cursor-personal/extensions"`**

Extensions are stored separately too. This means you could technically have different extensions in each instance, though we'll later sync them to keep things consistent.

**`--no-first-run`**

Skips the welcome screen and initial setup wizard. Since we're syncing settings from the main instance, we don't need to go through setup again.

**`--new-window`**

Forces the app to open a new window in the isolated instance rather than potentially opening in an existing window from the main Cursor instance.

**`--disable-dev-shm-usage`**

A memory management flag that prevents issues with shared memory, particularly useful when running multiple Electron-based apps.

**`nohup ... >/dev/null 2>&1 &`**

This shell technique does three things:

- `nohup` - Keeps the process running even if you close the terminal
- `>/dev/null 2>&1` - Redirects output and errors to /dev/null (keeps your terminal clean)
- `&` - Runs the process in the background

### Using picursor()

Once this function is in your `~/.zshrc`, using it is simple:

```bash
# Open Cursor-Personal in current directory
picursor .

# Open a specific project
picursor ~/Projects/my-side-project

# Just launch it
picursor
```

## The Sync Function: cursor-sync()

Launching a separate instance solves the isolation problem, but now you have a new challenge: keeping both instances updated and synchronized. When your main Cursor updates to a new version, your personal instance falls behind. When you install a helpful extension or tweak your keybindings, they don't carry over.

Enter `cursor-sync()` - a comprehensive sync solution that handles three critical aspects:

### Complete Function

Here's the full function (we'll break it down piece by piece below):

```bash
cursor-sync() {
  local main_app="/Applications/Cursor.app"
  local personal_app="/Applications/Cursor-Personal.app"
  local main_config="$HOME/Library/Application Support/Cursor"
  local personal_config="$HOME/.cursor-personal"

  echo "=== Cursor Sync ==="
  echo ""

  # Check if main Cursor exists
  if [ ! -d "$main_app" ]; then
    echo "❌ Main Cursor.app not found at $main_app"
    return 1
  fi

  # Step 1: Check and update app version
  local main_version=$(defaults read "$main_app/Contents/Info.plist" CFBundleShortVersionString 2>/dev/null)
  local personal_version=$(defaults read "$personal_app/Contents/Info.plist" CFBundleShortVersionString 2>/dev/null)

  echo "📦 App Versions:"
  echo "   Main Cursor: v$main_version"
  echo "   Cursor-Personal: v$personal_version"
  echo ""

  if [ "$main_version" != "$personal_version" ]; then
    echo "🔄 Updating Cursor-Personal app to v$main_version..."

    # Kill any running instances
    pkill -f "Cursor-Personal.app" 2>/dev/null
    sleep 2

    # Remove old version (need sudo since owned by root)
    sudo rm -rf "$personal_app"

    # Copy new version
    sudo cp -R "$main_app" "$personal_app"

    # Fix ownership
    sudo chown -R $(whoami):staff "$personal_app"

    echo "✓ App updated to v$main_version"
    echo ""
  else
    echo "✓ App version is up to date"
    echo ""
  fi

  # Step 2: Sync settings and extensions
  echo "⚙️  Syncing settings and extensions..."

  # Create backup
  if [ -d "$personal_config/User" ]; then
    cp -r "$personal_config/User" "$personal_config/User.backup.$(date +%Y%m%d_%H%M%S)" 2>/dev/null
  fi

  # Sync User settings (keybindings, settings.json, snippets, etc.)
  mkdir -p "$personal_config/User"
  if [ -d "$main_config/User" ]; then
    rsync -av --exclude='globalStorage' --exclude='workspaceStorage' --exclude='History' \
      "$main_config/User/" "$personal_config/User/" 2>&1 | grep -v "sending incremental" | grep -v "sent.*received" | grep -v "total size"
  fi

  # Sync extensions
  if [ -d "$main_config/extensions" ]; then
    mkdir -p "$personal_config/extensions"
    rsync -av "$main_config/extensions/" "$personal_config/extensions/" 2>&1 | grep -v "sending incremental" | grep -v "sent.*received" | grep -v "total size"
  fi

  echo ""
  echo "✓ Sync complete! Cursor-Personal is now fully up to date."
  echo "💡 Run 'picursor' to launch with all the latest features."
}
```

### 1. App Version Synchronization

```bash
# Check versions
local main_version=$(defaults read "$main_app/Contents/Info.plist" CFBundleShortVersionString 2>/dev/null)
local personal_version=$(defaults read "$personal_app/Contents/Info.plist" CFBundleShortVersionString 2>/dev/null)

if [ "$main_version" != "$personal_version" ]; then
  echo "🔄 Updating Cursor-Personal app to v$main_version..."

  # Kill running instances
  pkill -f "Cursor-Personal.app" 2>/dev/null
  sleep 2

  # Remove old version and copy new one
  sudo rm -rf "$personal_app"
  sudo cp -R "$main_app" "$personal_app"
  sudo chown -R $(whoami):staff "$personal_app"
fi
```

The function compares version numbers between your main Cursor and Cursor-Personal. When they differ, it:

1. **Gracefully stops** any running Cursor-Personal instances
2. **Removes** the outdated Cursor-Personal.app
3. **Copies** the entire updated app from the main Cursor
4. **Fixes ownership** so you can run it without permission issues

Note the `sudo` requirements - app files in `/Applications/` are typically owned by root, so we need elevated privileges to modify them.

### 2. Settings Synchronization

```bash
# Create safety backup
if [ -d "$personal_config/User" ]; then
  cp -r "$personal_config/User" "$personal_config/User.backup.$(date +%Y%m%d_%H%M%S)" 2>/dev/null
fi

# Sync User settings
mkdir -p "$personal_config/User"
if [ -d "$main_config/User" ]; then
  rsync -av --exclude='globalStorage' --exclude='workspaceStorage' --exclude='History' \
    "$main_config/User/" "$personal_config/User/"
fi
```

This section syncs all your settings while being smart about what to exclude:

**What Gets Synced:**

- `settings.json` - All your editor preferences
- `keybindings.json` - Custom keyboard shortcuts
- `snippets/` - Code snippets you've created
- Theme preferences
- UI state preferences

**What Gets Excluded:**

- `globalStorage` - Contains instance-specific data and tokens
- `workspaceStorage` - Project-specific settings and state
- `History` - Recently opened files and search history

Why exclude these? Because they're instance-specific. You don't want your work projects appearing in personal Cursor's recent files, and you definitely don't want to share authentication tokens between instances.

### 3. Extensions Synchronization

```bash
if [ -d "$main_config/extensions" ]; then
  mkdir -p "$personal_config/extensions"
  rsync -av "$main_config/extensions/" "$personal_config/extensions/"
fi
```

This ensures every extension installed in your main Cursor is available in Cursor-Personal. Install ESLint, Prettier, or any other extension once, and it appears in both instances.

### Safety First: Automatic Backups

Notice this line:

```bash
cp -r "$personal_config/User" "$personal_config/User.backup.$(date +%Y%m%d_%H%M%S)" 2>/dev/null
```

Before every sync, the function creates a timestamped backup of your personal instance's User folder. If something goes wrong, you can restore from `User.backup.20260114_143022` or whatever timestamp was created.

## Setting It All Up

Ready to implement this in your own workflow? Here's how:

### Step 1: Add Functions to Your Shell

Open your `~/.zshrc` file:

```bash
nano ~/.zshrc
# or
vim ~/.zshrc
```

Add both the `picursor()` and `cursor-sync()` functions to the end of the file. Copy the complete functions from earlier in this post:

- The `picursor()` function (from "The Launch Function" section)
- The `cursor-sync()` function (from "The Sync Function" section)

### Step 2: Reload Your Shell

```bash
source ~/.zshrc
```

### Step 3: First-Time Setup

Run the sync function to create your Cursor-Personal instance:

```bash
cursor-sync
```

This will:

- Copy Cursor.app to Cursor-Personal.app
- Create the `~/.cursor-personal` directory
- Sync all your settings and extensions

You'll need to enter your password when prompted (for the `sudo` commands).

### Step 4: Launch Your Personal Instance

```bash
# Replace with your actual project path
picursor ~/Projects/my-personal-project
```

The first time you launch, you'll need to log in with your personal Cursor account. After that, both instances maintain separate sessions.

**Note:** Replace `~/Projects/my-personal-project` with the actual path to your project directory.

## The Daily Workflow

Once set up, your workflow becomes beautifully simple:

**For work projects:**

```bash
# Just use Cursor normally
cursor .
# or open it from your dock/applications
```

**For personal projects:**

```bash
picursor ~/Projects/side-project
```

**After updates or installing new extensions:**

```bash
cursor-sync
```

Run `cursor-sync` whenever:

- Your main Cursor updates to a new version
- You install a new extension you want in both instances
- You modify settings or keybindings
- You want to ensure both instances are identical

## The Benefits

After using this setup for a while, several advantages became clear:

**Complete Isolation**: Work and personal accounts never interact. Different login sessions, different recent files, different workspace settings.

**Consistent Experience**: Both instances have the same extensions, keybindings, and settings. Muscle memory works the same everywhere.

**Easy Maintenance**: One command (`cursor-sync`) keeps everything updated. No manual copying, no forgetting which extensions you've installed where.

**Safe Experimentation**: Want to try a new extension or setting in your personal instance first? Go ahead - you control what syncs and when.

**Zero Context Switching Overhead**: No logging in and out, no checking which account you're using, no "oops, wrong account" moments.

## Tips and Best Practices

**Sync After Every Update**: Get in the habit of running `cursor-sync` whenever you see Cursor has updated. It takes seconds and ensures you won't forget.

**Bookmark Your Project Commands**: Add aliases for your common projects:

```bash
# Add these to your ~/.zshrc
alias workproject="cursor ~/Work/company-project"
alias sideproject="picursor ~/Projects/personal-project"
```

Replace the paths with your actual project directories.

**Check Version Numbers**: The sync function shows version numbers. If they differ significantly, you might want to sync before starting work.

**Understand What's Shared**: Remember that extensions and settings sync automatically, but project history, opened files, and authentication don't. That's by design.

## Conclusion

Managing multiple Cursor instances doesn't have to be painful. With these two functions, you get the best of both worlds: complete isolation between work and personal contexts, with effortless synchronization of the things that should be shared.

The setup takes five minutes. The benefits last forever.

Now, instead of choosing between convenience and separation, you have both. Your work projects stay in your work instance, your personal projects in your personal instance, and your productivity stays high in both.

Ready to give it a try? Add those functions to your `~/.zshrc` and reclaim control over your development environment.
