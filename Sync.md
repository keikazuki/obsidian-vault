
1. **Install the GitHub Sync Plugin**:
   - Open Obsidian.
   - Navigate to the **Settings**.
   - Click on **Community Plugins**.
   - Ensure **Safe Mode** is off.
   - Click on **Browse** and search for "GitHub Sync" by Kevin Chin.
   - Click **Install**, then **Enable** the plugin.

2. **Configure the Plugin**:
   - In the **Settings** under **Plugin Options**, select **GitHub Sync**.
   - Enter the **Repository URL** where you want to sync your vault ending with `.git`.

3. **Initialize a Git Repository in Your Obsidian Vault**:
   - First time?:
     - Open a terminal and run:
  ```bash
  cd /path/to/your/ObsidianVault
  git init
  git remote add origin https://github.com/YOUR_USERNAME/YOUR_OBSIDIAN_REPO.git
  git pull origin main  # If the repo already exists
  ```
   - Already done:
	- Open a terminal and run:
  ```bash
  cd /path/to/your/ObsidianVault
  git status
  git pull origin main  # If the repo already exists
  ```