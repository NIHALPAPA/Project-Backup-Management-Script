Project Backup Management Script

Prerequisites are given with their installation command below


Step 1: Installing Prerequisites
First, open your terminal and make sure your system is up to date:

sudo apt update && sudo apt upgrade -y
 

1. Install Git:
Required to clone and pull the latest code from your GitHub repository.

sudo apt install git -y
 
Verify installation:

git --version


2. Install Google Drive CLI (rclone tool):
This tool will upload your backups to Google Drive.

Install rclone: Run the following command to install rclone on Ubuntu:

sudo su

curl https://rclone.org/install.sh | sudo bash
 

Configure rclone for Google Drive: To configure rclone with Google Drive, you need to run the following command:

rclone config
 

Select "New Remote" and enter a name (e.g., gdrive).
 
Select Google Drive as the storage provider.

When prompted with "Use auto config?", select No (since our machine is remote).
 
Use auto config?
 * Say Y if not sure
 * Say N if you are working on a remote or headless machine

y) Yes (default)
n) No

y/n> n (by this we got one link in CLI we have to run below cmd in our local machine)

rclone authorize "drive" "eyJzY29********


Since we are on a remote machine and can't access a browser, you'll need to perform the authorization on a local machine that does have a browser. Follow these steps carefully:

First we have to install rclone on our Local Machine 

https://rclone.org/downloads/

we have to configure rclone on local machine as well

rclone config 

Select "New Remote" and enter a name (e.g., gdrive).

Select Google Drive as the storage provider.

When prompted with "Use auto config?", select No (since our machine is remote).

Use auto config?
 * Say Y if not sure
 * Say N if you are working on a remote or headless machine

y) Yes (default)
n) No

y/n> yes (by this browser will get open automatically on local machine)

Authenticate by signing in google account 

rclone authorize "drive" "eyJzY29w********* (Now paste this code which we get from our remote machine)

you will get the Authentication token for rclone 

Paste the following into your remote machine --->
 

Verify Rclone is Connected 
 
rclone lsd gdrive:

Mount Google Drive:

mkdir ~/gdrive
chmod 755 ~/gdrive
rclone mount gdrive: ~/gdrive &


3. Install cURL:
cURL is required if you are using the optional webhook feature to notify you when a backup is complete.


sudo apt install curl -y


Verify installation:

curl --version

WEB-HOOK Setup 

Slack Webhook Setup
https://api.slack.com/messaging/webhooks

Click "Create an App" → Select "From Scratch"
Choose a name (e.g., "Backup Notifications") and select a Slack workspace.
Go to Incoming Webhooks → Activate Incoming Webhooks.
Click "Add New Webhook to Workspace" → Select a channel → Click Allow.
Copy the webhook URL (it looks like https://hooks.slack.com/services/T000/B000/XXXX).
Replace "your_webhook_url" in your script with this URL.

4. Install zip

sudo apt update && sudo apt install zip -y

Once installed, verify by running:

zip --version

Slack Webhook Setup 
 
To enable Slack notifications for backup status updates:

1. Go to [Slack API](https://api.slack.com/messaging/webhooks).
2. Click "Create an App" → Select "From Scratch".
3. Choose a name (e.g., "Backup Notifications") and select a Slack workspace.
4. Go to Incoming Webhooks → Activate Incoming Webhooks.
5. Click "Add New Webhook to Workspace" → Select a channel → Click Allow.
6. Copy the webhook URL (it looks like `https://hooks.slack.com/services/T000/B000/XXXX`).
7. Replace the `WEBHOOK_URL` variable in the script with this URL.


Step 2: Create and Set Up the Backup Script
Now let’s create and set up the backup script.

1. Create a new shell script: Open a terminal and create a new shell script file using your preferred text editor (e.g., nano, vim):


nano backup_script.sh


2. Copy and paste the script into the backup_script.sh file:

#!/bin/bash

# Configurable Variables
PROJECT_FOLDER="/home/ubuntu/PROJECT_FOLDER"
GITHUB_REPO_URL="https://github.com/NIHALPAPA/Project-Backup-Management-Script.git"
BACKUP_DIR="/home/ubuntu/gdrive/BACKUP_DIR"
PROJECT_NAME="Project_Backup_Management_Script"  # Use underscores or hyphens for file names
RCLONE_REMOTE="gdrive"          # Name of the rclone remote configured
RCLONE_FOLDER_PATH="/BACKUP_DIR" # Path in your Google Drive (no leading slash)
RETENTION_DAYS=7                # Keep daily backups for 7 days
RETENTION_WEEKS=4               # Keep weekly backups for 4 weeks
RETENTION_MONTHS=3              # Keep monthly backups for 3 months
ENABLE_CURL=true                # Enable or disable webhook notifications
WEBHOOK_URL="https://hooks.slack.com/services/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"

# Logging Function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Create Backup Directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# 1. Clone or Pull the Latest Code (Improved)
if [ ! -d "$PROJECT_FOLDER" ]; then
    log "Cloning repository..."
    git clone "$GITHUB_REPO_URL" "$PROJECT_FOLDER"
elif [[ -d "$PROJECT_FOLDER" && -d "$PROJECT_FOLDER/.git" ]]; then
    log "Pulling latest changes..."
    git -C "$PROJECT_FOLDER" pull origin main  # Or specify your branch
else
    log "Error: Project folder exists but is not a Git repository."
    exit 1
fi

# 2. Create a Timestamped Backup (Improved)
TIMESTAMP=$(date "+%Y-%m-%d_%H-%M-%S")
BACKUP_FILE="$BACKUP_DIR/${PROJECT_NAME}_backup_$TIMESTAMP.tar.gz"  # Using tar.gz as an alternative

# Exclude .git and other unwanted files/directories. Add more as needed.
EXCLUDE_PARAMS=("--exclude=.git" "--exclude=node_modules" "--exclude=*.tmp*")

# Check for Empty project directory (After exclusions)
if [ -z "$(find "$PROJECT_FOLDER" -mindepth 1 -not -path '*/\.git*' -not -path '*/\node_modules*' -print -quit)" ]; then
    log "Error: Project folder is empty (after exclusions), nothing to backup."
    exit 1
fi

# Create Backup using tar.gz (more reliable than zip)
log "Creating backup of $PROJECT_FOLDER..."
if tar -czf "$BACKUP_FILE" "${EXCLUDE_PARAMS[@]}" -C "$(dirname "$PROJECT_FOLDER")" "$(basename "$PROJECT_FOLDER")"; then
    log "Backup created successfully: $BACKUP_FILE"
else
    log "Error: Backup creation failed."
    exit 1
fi

# 3. Upload Backup to Google Drive using rclone (Improved)
log "Uploading backup to Google Drive..."
if rclone copy "$BACKUP_FILE" "$RCLONE_REMOTE:$RCLONE_FOLDER_PATH"; then
    log "Backup uploaded successfully to Google Drive."
else
    log "Error: Google Drive upload failed."
    exit 1
fi

# 4. Implement Rotational Backup Deletion (Improved)
delete_old_backups() {
    local days=$1
    log "Deleting backups older than $days days..."
    find "$BACKUP_DIR" -name "${PROJECT_NAME}_backup_*.tar.gz" -type f -mtime +$days -print0 | xargs -0 -I {} sh -c 'if [[ -f "{}" ]]; then rm -f "{}"; echo "Deleted: {}"; fi'
}

delete_old_backups "$RETENTION_DAYS"
delete_old_backups $((RETENTION_WEEKS * 7))
delete_old_backups $((RETENTION_MONTHS * 30))

# 5. Optional Webhook Notification (Improved)
if [ "$ENABLE_CURL" = true ]; then
    log "Sending Slack notification..."
    if curl -X POST -H "Content-Type: application/json" -d '{"project": "'$PROJECT_NAME'", "date": "'$TIMESTAMP'", "status": "Backup successful"}' "$WEBHOOK_URL"; then
        log "Slack notification sent successfully."
    else
        log "Warning: Webhook notification failed. Check your WEBHOOK_URL."
    fi
fi

log "Backup and cleanup completed successfully."



3. Save the script: After pasting, press CTRL+O ENTER CTR+X to save and exit nano.

4. Make the script executable: To run the script, it needs to be executable. Run the following command:


chmod +x backup_script.sh


Step 3: Run the Script Manually
Now you can run the script manually to test it. In the terminal, run:


./backup_script.sh
 

If everything works correctly, the following messages in your terminal will be generated:

The backup process starts.
The project is cloned or updated.
A backup is created and uploaded to Google Drive.
Old backups are deleted based on your retention policy.


Step 4: Automate the Script with Cron

To schedule this backup to run automatically, set up a cron job.

1. Open the crontab file for editing:

crontab -e


2. Add a cron job to run the script at your desired schedule. For example, to run the backup every day at midnight:

0 0 * * * /home/ubuntu/backup_script.sh
 

This will run the script every day at midnight. You can adjust the time and frequency based on your needs.



Troubleshooting
1. Permission Issues
Ensure the script has execute permissions:

chmod +x backup_script.sh


2. Google Drive Issues
Verify that rclone is correctly configured and authenticated.

Ensure the RCLONE_FOLDER_PATH exists in your Google Drive.


3. cURL/Webhook Issues
Test the webhook URL manually:

curl -X POST -H "Content-Type: application/json" -d '{"text": "Test message"}' "https://hooks.slack.com/services/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"


THANK YOU :)
