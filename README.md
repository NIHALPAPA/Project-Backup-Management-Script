# Project-Backup-Management-Script

Prerequisites:
1. Git: To clone and pull the latest code from your GitHub repository.

2. Google Drive CLI (gdrive): This is required to upload the backup to Google Drive.

3. cURL: For making HTTP requests (needed for optional webhook notification).

4. Backup directory: You need a directory to store the backups temporarily.

5. Google Drive API setup: You need to configure your Google Drive account for CLI tool access.

Step 1: Installing Prerequisites
First, open your terminal and make sure your system is up to date:

sudo apt update && sudo apt upgrade -y


1. Install Git:
Git is needed to clone the GitHub repository.

sudo apt install git -y


Verify installation:

git --version


2. Install rclone CLI Tool:
This tool will upload your backups to Google Drive.

Install rclone: Run the following command to install rclone on Ubuntu:

sudo apt install rclone -y


Configure rclone for Google Drive: To configure rclone with Google Drive, you need to run the following command:

rclone config


Select n to create a new remote.
Name the remote (e.g., gdrive).
Choose 18 for Google Drive.
Follow the instructions to authenticate rclone with your Google Drive account. You'll get a link to paste in your browser for authentication.
Once it's authenticated, you'll have a configured remote (e.g., gdrive) that you can use for backups.

No remotes found, make a new one?
n) New remote
r) Rename remote
c) Copy remote
s) Set configuration password
q) Quit config
n/r/c/s/q> n
name> remote
Type of storage to configure.
Choose a number from below, or type in your own value
[snip]
XX / Google Drive
   \ "drive"
[snip]
Storage> drive
Google Application Client Id - leave blank normally.
client_id>
Google Application Client Secret - leave blank normally.
client_secret>
Scope that rclone should use when requesting access from drive.
Choose a number from below, or type in your own value
 1 / Full access all files, excluding Application Data Folder.
   \ "drive"
 2 / Read-only access to file metadata and file contents.
   \ "drive.readonly"
   / Access to files created by rclone only.
 3 | These are visible in the drive website.
   | File authorization is revoked when the user deauthorizes the app.
   \ "drive.file"
   / Allows read and write access to the Application Data folder.
 4 | This is not visible in the drive website.
   \ "drive.appfolder"
   / Allows read-only access to file metadata but
 5 | does not allow any access to read or download file content.
   \ "drive.metadata.readonly"
scope> 1
Service Account Credentials JSON file path - needed only if you want use SA instead of interactive login.
service_account_file>
Remote config
Use web browser to automatically authenticate rclone with remote?
 * Say Y if the machine running rclone has a web browser you can use
 * Say N if running rclone on a (remote) machine without web browser access
If not sure try Y. If Y failed, try N.
y) Yes
n) No
y/n> y
If your browser doesn't open automatically go to the following link: http://127.0.0.1:53682/auth
Log in and authorize rclone for access
Waiting for code...
Got code
Configure this as a Shared Drive (Team Drive)?
y) Yes
n) No
y/n> n
Configuration complete.
Options:
type: drive
- client_id:
- client_secret:
- scope: drive
- root_folder_id:
- service_account_file:
- token = {"access_token":"ya29.a0AcM612zRlOVOTzN-3ZmtSA0uKT8s5KJInQmiHMyHJ-2ywtZdZVVTP54gn6E2cGn3hGG7F4SL 2hDH2XQLUnlxFPeAP3C8HLSBsty3LHUsr8wVIUY3aQdkXOKSmrSUI-Ocv78mI53oz79KeVyqRxm3U2_fH8YwCQd-8Rq0lG1DaCgYKAYQ SARASFQHGX2MisNzXmWj0KG_VOdUR0wx3xg0175", "token_type": "Bearer", "refresh_token": "1//06G0s0-9nI8v9CgYIARAA GAYSNWF-L9IrLpCk0yTWOgMILQEtNFK1wvl521a2T561AZHXX-GAH3FUt4xpxFmdQ-w8wE3zWo38eeA", "expiry": "2025-02-02T10 
49:03.714268929+07:00"}

Keep this "remote" remote?
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y

To Access the content of google drive we have to mount it first 
mkdir gdrive

chmod 755 gdrive

rclone mount gdrive: ~/gdrive 


3. Install cURL:
cURL is required if you are using the optional webhook feature to notify you when a backup is complete.


sudo apt install curl -y


Verify installation:

curl --version


Step 2: Create and Set Up the Backup Script
Now let’s create and set up the backup script.

1. Create a new shell script: Open a terminal and create a new shell script file using your preferred text editor (e.g., nano, vim):


nano backup_script.sh


2. Copy and paste the script into the backup_script.sh file:

#!/bin/bash

# Configurable Variables
PROJECT_FOLDER="/home/ubuntu/PROJECT_FOLDER"
GITHUB_REPO_URL="https://github.com/NIHALPAPA/Project-Backup-Management-Script.git"
BACKUP_DIR="/home/ubuntu/BACKUP_DIR"
PROJECT_NAME="Project Backup Management Script"
RCLONE_REMOTE="gdrive"          # Name of the rclone remote configured
RCLONE_FOLDER_PATH="https://drive.google.com/drive/my-drive"   # Path in your Google Drive where backups will be uploaded
RETENTION_DAYS=7               # Keep daily backups for 7 days
RETENTION_WEEKS=4              # Keep weekly backups for 4 weeks
RETENTION_MONTHS=3             # Keep monthly backups for 3 months
ENABLE_CURL=true               # Enable or disable webhook notifications
WEBHOOK_URL="your_webhook_url"

# Step 1: Clone or Pull the Latest Code
if [ ! -d "$PROJECT_FOLDER" ]; then
  # If project folder does not exist, clone the repo
  git clone "$GITHUB_REPO_URL" "$PROJECT_FOLDER"
elif [[ -d "$PROJECT_FOLDER" && -d "$PROJECT_FOLDER/.git" ]]; then
  # If folder is a Git repo, pull the latest changes
  git -C "$PROJECT_FOLDER" pull origin main
else
  # Error if the folder exists but is not a Git repo
  echo "Error: Project folder exists but is not a Git repository."
  exit 1
fi

# Step 2: Create a Timestamped Backup
TIMESTAMP=$(date "+%Y-%m-%d_%H-%M-%S")
BACKUP_FILE="$BACKUP_DIR/${PROJECT_NAME}_backup_$TIMESTAMP.zip"

# Ensure the backup directory exists
mkdir -p "$BACKUP_DIR"

# Create a backup of the project directory
zip -r "$BACKUP_FILE" "$PROJECT_FOLDER"

# Check if the backup was successful
if [[ $? -ne 0 ]]; then
  echo "Error: Backup creation failed."
  exit 1
fi

# Step 3: Upload Backup to Google Drive using rclone
rclone copy "$BACKUP_FILE" "$RCLONE_REMOTE:$RCLONE_FOLDER_PATH"

# Check if the upload was successful
if [[ $? -ne 0 ]]; then
  echo "Error: Google Drive upload failed."
  exit 1
fi

# Step 4: Implement Rotational Backup Deletion
# Function to delete backups older than the specified number of days
delete_old_backups() {
  local days=$1
  find "$BACKUP_DIR" -name "${PROJECT_NAME}_backup_*.zip" -type f -mtime +$days -print0 | xargs -0 -I {} sh -c 'if [[ -f "{}" ]]; then rm -f "{}"; echo "Deleted: {}"; fi'
}

# Delete old backups based on retention periods
delete_old_backups "$RETENTION_DAYS"             # Daily backups
delete_old_backups $((RETENTION_WEEKS * 7))      # Weekly backups
delete_old_backups $((RETENTION_MONTHS * 30))    # Monthly backups

# Step 5: Optional Webhook Notification
if [ "$ENABLE_CURL" = true ]; then
  curl -X POST -H "Content-Type: application/json" -d '{"project": "'$PROJECT_NAME'", "date": "'$TIMESTAMP'", "status": "Backup successful"}' "$WEBHOOK_URL"
  if [[ $? -ne 0 ]]; then
    echo "Warning: Webhook notification failed. Check your WEBHOOK_URL."
  fi
fi

# Final success message
echo "Backup and cleanup completed successfully."



3. Save the script: After pasting, press CTRL+O ENTER CTR+X to save and exit nano.

4. Make the script executable: To run the script, it needs to be executable. Run the following command:


chmod +x backup_script.sh


Step 3: Configure the Script
Now, configure the script with your specific information.

1. Set the project folder path (PROJECT_FOLDER):

Update PROJECT_FOLDER="/path/to/project" to the directory where your project files are located on your local machine.

2. Set the GitHub repository URL (GITHUB_REPO_URL):

Replace https://github.com/username/repo.git with the URL of the GitHub repository you want to clone.

3. Set the backup directory (BACKUP_DIR):

Replace "/path/to/backups" with the path where you want to store your backup files.

4. Set the rclone Google Drive Remote and Folder Path (RCLONE_REMOTE and RCLONE_FOLDER_PATH): Replace the values of RCLONE_REMOTE and RCLONE_FOLDER_PATH with your rclone configuration.

RCLONE_REMOTE: This is the name of your rclone remote, which you configured during rclone config (e.g., gdrive).
RCLONE_FOLDER_PATH: This is the path to the folder in Google Drive where you want to upload backups (e.g., my_backups).

RCLONE_REMOTE="gdrive"              # Name of your rclone remote
RCLONE_FOLDER_PATH="backup_folder"  # Path in Google Drive (e.g., "my_backups" or "backup_folder")


5. Set the webhook URL (WEBHOOK_URL):

If you're using a webhook notification (e.g., Slack, Discord), replace the WEBHOOK_URL with your actual webhook URL.

6. Set retention days, weeks, and months:

Configure the RETENTION_DAYS, RETENTION_WEEKS, and RETENTION_MONTHS values according to how long you want to keep your backups.


Step 4: Run the Script Manually
Now you can run the script manually to test it. In the terminal, run:


./backup_script.sh


If everything works correctly, you should see the following messages in your terminal:

The backup process starts.
The project is cloned or updated.
A backup is created and uploaded to Google Drive.
Old backups are deleted based on your retention policy.


Step 5: Automate the Script with Cron

To schedule this backup to run automatically, set up a cron job.

1. Open the crontab file for editing:

crontab -e


2. Add a cron job to run the script at your desired schedule. For example, to run the backup every day at midnight:

0 0 * * * /path/to/backup_script.sh


This will run the script every day at midnight. You can adjust the time and frequency based on your needs.



Troubleshooting
1. Permission Issues:

If you encounter permission issues, make sure the script has execute permissions: 

chmod +x backup_script.sh


Google Drive Issues:

Make sure you've correctly authenticated gdrive and that the folder ID is correct.


cURL/WEBHOOK Issues:

Verify that the WEBHOOK_URL is working by testing it with a simple curl command manually.
