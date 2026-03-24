---
name: update-all
description: Dynamically discover all git repositories under a parent folder, checkout master/main, and pull latest code. Use when user wants to update all microservices, sync all repos, or says "update-all" or "pull all repos".
tools: Bash
---

# Update All Repos

You are helping a developer update all their microservice repositories at once.

## Goal

1. Ask the user for their parent folder (or use a default if provided)
2. Dynamically discover all git repos inside that folder
3. For each repo: checkout the default branch (master or main) and pull latest

---

## Step 1: Get the parent folder

If the user did not provide a path, ask:

```
Which folder contains all your microservice repos?
(e.g. ~/projects, ~/work, /Users/yourname/services)
```

If the user already provided a path in their message, use it directly.

---

## Step 2: Run the update

Once you have the path, run this command via Bash:

```bash
PARENT_DIR="<user-provided-path>"

for dir in "$PARENT_DIR"/*/; do
  if [ -d "$dir/.git" ]; then
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "📁 $dir"

    # Determine default branch (master or main)
    DEFAULT_BRANCH=$(git -C "$dir" symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')

    # Fallback: try master then main
    if [ -z "$DEFAULT_BRANCH" ]; then
      if git -C "$dir" show-ref --verify --quiet refs/heads/master; then
        DEFAULT_BRANCH="master"
      elif git -C "$dir" show-ref --verify --quiet refs/heads/main; then
        DEFAULT_BRANCH="main"
      else
        echo "⚠️  Could not determine default branch, skipping."
        continue
      fi
    fi

    echo "🔀 Switching to $DEFAULT_BRANCH..."
    git -C "$dir" checkout "$DEFAULT_BRANCH" 2>&1

    echo "⬇️  Pulling latest..."
    git -C "$dir" pull 2>&1
  fi
done

echo ""
echo "✅ Done updating all repos."
```

Replace `<user-provided-path>` with the actual path before running.

---

## Step 3: Report results

After running, summarize:
- How many repos were found
- Which ones updated successfully
- Any that had errors (conflicts, detached HEAD, no remote, etc.) — flag these clearly so the user knows what needs manual attention
