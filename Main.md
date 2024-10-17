# Automate Daily Anki Progress Emails with Clockify Integration on macOS

This guide will help you set up a system on macOS to automatically send daily emails with your Anki progress, including:

- **Anki card counts** by category (New, Learning, Young, Mature).
- **Total reviews** done today.
- **Time spent on projects** tracked via Clockify.

The system will:

1. **Close Anki** if it's running.
2. **Run a Python script** to gather data and send an email via Gmail.
3. **Reopen Anki** after the script runs.
4. **Schedule the script to run daily** at a specified time using a LaunchAgent.
5. **Wake your Mac** from sleep to run the script (optional).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setting Up the Python Script](#setting-up-the-python-script)
  - [1. Install Python 3](#1-install-python-3)
  - [2. Create a Project Directory](#2-create-a-project-directory)
  - [3. Set Up a Virtual Environment](#3-set-up-a-virtual-environment)
  - [4. Install Required Python Packages](#4-install-required-python-packages)
  - [5. Create the Python Script](#5-create-the-python-script)
- [Setting Up Gmail API Credentials](#setting-up-gmail-api-credentials)
  - [1. Enable Gmail API](#1-enable-gmail-api)
  - [2. Create OAuth Client ID Credentials](#2-create-oauth-client-id-credentials)
  - [3. Download `credentials.json`](#3-download-credentialsjson)
  - [4. Place `credentials.json` in Your Project Directory](#4-place-credentialsjson-in-your-project-directory)
- [Setting Up Clockify Integration](#setting-up-clockify-integration)
  - [1. Obtain Your Clockify API Key](#1-obtain-your-clockify-api-key)
  - [2. Set the API Key as an Environment Variable](#2-set-the-api-key-as-an-environment-variable)
- [Creating the Shell Script](#creating-the-shell-script)
- [Creating the LaunchAgent](#creating-the-launchagent)
- [Scheduling Your Mac to Wake from Sleep (Optional)](#scheduling-your-mac-to-wake-from-sleep-optional)
- [Testing the Setup](#testing-the-setup)
- [Troubleshooting](#troubleshooting)
- [Conclusion](#conclusion)

---

## Prerequisites

- **macOS** computer.
- **Anki** installed.
- **Python 3** installed (macOS comes with Python 2; you need to install Python 3).
- Basic familiarity with the command line.
- **Clockify** account (free tier is sufficient).

---

## Setting Up the Python Script

### 1. Install Python 3

If you don't have Python 3 installed, install it using [Homebrew](https://brew.sh/):

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Python 3
brew install python3
```

### 2. Create a Project Directory

Create a directory for your project:

```bash
mkdir ~/AnkiEmailSender
cd ~/AnkiEmailSender
```

### 3. Set Up a Virtual Environment

Set up a Python virtual environment to manage dependencies:

```bash
python3 -m venv venv
```

Activate the virtual environment:

```bash
source venv/bin/activate
```

### 4. Install Required Python Packages

Install the necessary Python packages:

```bash
pip install google-auth google-auth-oauthlib google-api-python-client isodate requests
```

### 5. Create the Python Script

Create a file named `main.py` in your project directory:

```bash
nano main.py
```

Copy and paste the following code into `main.py`:

```python
import os
import pickle
import sqlite3
import json
from datetime import datetime, timedelta, date, timezone
from email.mime.text import MIMEText
import base64

from google.auth.transport.requests import Request
from google_auth_oauthlib.flow import InstalledAppFlow
# from google.oauth2.credentials import Credentials  # Not used in this script
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError
import requests
import isodate

# If modifying these scopes, delete the file token.pickle
SCOPES = ['https://www.googleapis.com/auth/gmail.send']

def get_anki_db_path():
    # Replace 'User 1' with your actual Anki profile name
    anki_profile = 'User 1'
    anki_base_path = os.path.expanduser('~/Library/Application Support/Anki2')
    anki_db_path = os.path.join(anki_base_path, anki_profile, 'collection.anki2')
    return anki_db_path

def get_category_counts():
    db_path = get_anki_db_path()
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    YOUNG_THRESHOLD = 21  # days

    counts = {}

    # New Cards
    cursor.execute("SELECT COUNT(*) FROM cards WHERE type = 0")
    counts['new'] = cursor.fetchone()[0]

    # Learning Cards
    cursor.execute("SELECT COUNT(*) FROM cards WHERE queue IN (1, 3)")
    counts['learning'] = cursor.fetchone()[0]

    # Young Cards
    cursor.execute(f"SELECT COUNT(*) FROM cards WHERE queue = 2 AND ivl < {YOUNG_THRESHOLD}")
    counts['young'] = cursor.fetchone()[0]

    # Mature Cards
    cursor.execute(f"SELECT COUNT(*) FROM cards WHERE queue = 2 AND ivl >= {YOUNG_THRESHOLD}")
    counts['mature'] = cursor.fetchone()[0]

    # Total Reviews Today
    today_start = datetime.combine(date.today(), datetime.min.time())
    tomorrow_start = today_start + timedelta(days=1)
    start_timestamp = int(today_start.timestamp() * 1000)
    end_timestamp = int(tomorrow_start.timestamp() * 1000)

    cursor.execute("""
        SELECT COUNT(*)
        FROM revlog
        WHERE id BETWEEN ? AND ?
    """, (start_timestamp, end_timestamp))
    counts['total_reviews'] = cursor.fetchone()[0]

    conn.close()
    return counts

def save_daily_counts(counts):
    filename = 'counts.json'
    with open(filename, 'w') as f:
        json.dump(counts, f)

def load_counts():
    filename = 'counts.json'
    if os.path.exists(filename):
        with open(filename, 'r') as f:
            return json.load(f)
    else:
        return None

def calculate_differences(today_counts, yesterday_counts):
    differences = {}
    for category in today_counts:
        differences[category] = today_counts[category] - yesterday_counts.get(category, 0)
    return differences

def create_message(sender, to, subject, message_text):
    message = MIMEText(message_text)
    if isinstance(to, list):
        message['To'] = ', '.join(to)
    else:
        message['To'] = to
    message['From'] = sender
    message['Subject'] = subject
    raw_message = base64.urlsafe_b64encode(message.as_bytes()).decode()
    return {'raw': raw_message}

def send_email_via_gmail(service, user_id, message):
    try:
        sent_message = service.users().messages().send(userId=user_id, body=message).execute()
        print(f"Email sent successfully.")
        return sent_message
    except HttpError as error:
        print(f"An error occurred: {error}")
        return None

def get_daily_time_entries(api_key):
    headers = {'X-Api-Key': api_key}
    base_url = 'https://api.clockify.me/api/v1'

    # Get user info to obtain the user ID and default workspace ID
    user_response = requests.get(f'{base_url}/user', headers=headers)
    if user_response.status_code != 200:
        print('Error getting user info:', user_response.text)
        return None
    user = user_response.json()
    user_id = user['id']
    workspace_id = user['activeWorkspace']

    # Get the local date and time
    local_now = datetime.now()
    local_date = local_now.date()

    # Start and end of the day in local time
    start_of_day_local = datetime.combine(local_date, datetime.min.time())
    end_of_day_local = datetime.combine(local_date, datetime.max.time())

    # Get the UTC offset in hours
    utc_offset_timedelta = local_now - datetime.utcnow()
    utc_offset_seconds = utc_offset_timedelta.total_seconds()
    utc_offset = timedelta(seconds=utc_offset_seconds)
    local_timezone = timezone(utc_offset)

    # Attach timezone info to datetime objects
    start_of_day_local = start_of_day_local.replace(tzinfo=local_timezone)
    end_of_day_local = end_of_day_local.replace(tzinfo=local_timezone)

    # Convert to UTC
    start_of_day_utc = start_of_day_local.astimezone(timezone.utc)
    end_of_day_utc = end_of_day_local.astimezone(timezone.utc)

    # Format as ISO strings for API request
    start_of_day_utc_str = start_of_day_utc.isoformat().replace('+00:00', 'Z')
    end_of_day_utc_str = end_of_day_utc.isoformat().replace('+00:00', 'Z')

    # Endpoint to retrieve time entries
    time_entries_url = f'{base_url}/workspaces/{workspace_id}/user/{user_id}/time-entries'

    params = {
        'start': start_of_day_utc_str,
        'end': end_of_day_utc_str,
        'hydrated': 'true',
        'page-size': 500  # Adjust if you have more than 500 entries per day
    }

    time_entries_response = requests.get(time_entries_url, headers=headers, params=params)
    if time_entries_response.status_code != 200:
        print('Error getting time entries:', time_entries_response.text)
        return None
    time_entries = time_entries_response.json()

    # Handle the case where no time entries are found
    if not time_entries:
        print("No time entries found for the specified time range.")
        return {}

    # Aggregate time spent per project
    project_times = {}
    for entry in time_entries:
        project_name = entry.get('project', {}).get('name', 'No Project')
        duration = entry.get('timeInterval', {}).get('duration')
        if duration:
            duration_seconds = isodate.parse_duration(duration).total_seconds()
            project_times[project_name] = project_times.get(project_name, 0) + duration_seconds

    # Convert total seconds to hours and minutes
    for project in project_times:
        total_seconds = project_times[project]
        hours = int(total_seconds // 3600)
        minutes = int((total_seconds % 3600) // 60)
        project_times[project] = {'hours': hours, 'minutes': minutes}

    return project_times

def main():
    creds = None
    # The file token.pickle stores the user's access and refresh tokens
    if os.path.exists('token.pickle'):
        try:
            with open('token.pickle', 'rb') as token:
                creds = pickle.load(token)
        except (pickle.UnpicklingError, EOFError) as e:
            print("Error loading token.pickle. It may be corrupted. Deleting the file and re-authenticating.")
            os.remove('token.pickle')
            creds = None

    # If there are no valid credentials, let the user log in
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            try:
                creds.refresh(Request())
            except google.auth.exceptions.RefreshError:
                print("Failed to refresh token. It may have been revoked or expired.")
                os.remove('token.pickle')
                creds = None

        if not creds or not creds.valid:
            try:
                flow = InstalledAppFlow.from_client_secrets_file(
                    'credentials.json', SCOPES)
                creds = flow.run_local_server(port=0)
            except FileNotFoundError:
                print("credentials.json file not found. Please ensure it exists in the script directory.")
                return
            except Exception as e:
                print(f"An error occurred during authentication: {e}")
                return

        # Save the credentials for future use
        with open('token.pickle', 'wb') as token:
            pickle.dump(creds, token)

    try:
        # Build the Gmail service
        service = build('gmail', 'v1', credentials=creds)

        # Load yesterday's counts
        yesterday_counts = load_counts()

        # Get today's counts
        today_counts = get_category_counts()

        # Calculate differences
        if yesterday_counts:
            differences = calculate_differences(today_counts, yesterday_counts)
        else:
            differences = {category: 0 for category in today_counts}

        # Get Clockify API key from environment variable
        api_key = os.environ.get('CLOCKIFY_API_KEY')
        if not api_key:
            print("Clockify API key not found in environment variable 'CLOCKIFY_API_KEY'. Please set it and try again.")
            return

        # Get time entries from Clockify
        daily_project_times = get_daily_time_entries(api_key)

        # Prepare email content
        sender_email = 'your_email@gmail.com'  # Replace with your Gmail address
        recipient_emails = ['recipient1@example.com', 'recipient2@example.com']  # Replace with recipient emails
        subject = f"Anki Progress for {date.today().isoformat()}"

        body = f"Hi,\n\nHere's your Anki progress for {date.today().isoformat()}:\n"
        body += f"\nTotal reviews today: {today_counts['total_reviews']}"
        body += f"\n\nCategory counts and changes since last run:"
        body += f"\nNew cards: {today_counts['new']} ({differences['new']:+})"
        body += f"\nLearning cards: {today_counts['learning']} ({differences['learning']:+})"
        body += f"\nYoung cards: {today_counts['young']} ({differences['young']:+})"
        body += f"\nMature cards: {today_counts['mature']} ({differences['mature']:+})"

        # Include Clockify data
        if daily_project_times:
            body += "\n\nTime spent on projects today (from Clockify):"
            for project, time_spent in daily_project_times.items():
                body += f"\n{project}: {time_spent['hours']}h {time_spent['minutes']}m"
        else:
            body += "\n\nCould not retrieve Clockify data."

        # Send email to each recipient individually
        for recipient_email in recipient_emails:
            message = create_message(sender_email, recipient_email, subject, body)
            send_email_via_gmail(service, 'me', message)

        # Save today's counts for the next run
        save_daily_counts(today_counts)

    except Exception as e:
        print(f"An error occurred: {e}")

if __name__ == '__main__':
    main()
```

**Important Notes:**

- **Anki Profile Name:**
  - Replace `'User 1'` in `anki_profile = 'User 1'` with your actual Anki profile name if it's different.
- **Email Addresses:**
  - Replace `'your_email@gmail.com'` with your Gmail address.
  - Update `recipient_emails` with the email addresses you want to send the report to.
- **Clockify API Key:**
  - We'll set this up in a later section.

Save and exit the editor.

---

## Setting Up Gmail API Credentials

To send emails via Gmail, you'll need to set up Gmail API credentials.

### 1. Enable Gmail API

- Go to the [Google Cloud Console](https://console.cloud.google.com/).
- Create a new project or select an existing one.
- Navigate to **APIs & Services** > **Library**.
- Search for **Gmail API** and click on it.
- Click **Enable**.

### 2. Create OAuth Client ID Credentials

- Go to **APIs & Services** > **Credentials**.
- Click **Create Credentials** > **OAuth client ID**.
- If prompted to configure the consent screen, do so (you can set it to external and fill in the required fields).
- Select **Desktop app** as the application type.
- Click **Create**.

### 3. Download `credentials.json`

- After creating the OAuth client ID, click **Download JSON**.
- Rename the downloaded file to `credentials.json`.

### 4. Place `credentials.json` in Your Project Directory

- Move the `credentials.json` file to your project directory (`~/AnkiEmailSender`).

---

## Setting Up Clockify Integration

To fetch time entries from Clockify, you'll need to obtain your API key and set it as an environment variable.

### 1. Obtain Your Clockify API Key

1. Log in to your Clockify account.
2. Click on your profile avatar in the top-right corner.
3. Select **Profile Settings**.
4. Scroll down to the **API** section.
5. Copy your **API Key**.

### 2. Set the API Key as an Environment Variable

We'll set the API key in your shell's profile so that it's available when the script runs.

1. Open your shell profile in an editor (e.g., `.bash_profile`, `.bashrc`, or `.zshrc`):

   ```bash
   nano ~/.bash_profile
   ```

2. Add the following line to set the environment variable:

   ```bash
   export CLOCKIFY_API_KEY="your_clockify_api_key_here"
   ```

   - Replace `"your_clockify_api_key_here"` with the API key you copied.

3. Save and exit the editor.

4. Reload your shell profile to apply the changes:

   ```bash
   source ~/.bash_profile
   ```

**Note:** If you're using `zsh`, use `~/.zshrc` instead of `~/.bash_profile`.

---

## Creating the Shell Script

Create a shell script that will automate the process:

```bash
nano ~/run_anki_and_script.sh
```

Paste the following content:

```bash
#!/bin/bash

# Close Anki if it's running
pkill -f Anki

# Wait a few seconds to ensure Anki has closed
sleep 5

# Navigate to your script's directory
cd "/Users/your_username/AnkiEmailSender"

# Activate virtual environment
source "/Users/your_username/AnkiEmailSender/venv/bin/activate"

# Run the Python script
python3 main.py

# Deactivate virtual environment
deactivate

# Wait a few seconds to ensure the script finishes
sleep 5

# Open Anki
open "/Applications/Anki.app"
```

**Notes:**

- **Replace `/Users/your_username/AnkiEmailSender`** with the actual path to your project directory.
- **Make the script executable:**

  ```bash
  chmod +x ~/run_anki_and_script.sh
  ```

---

## Creating the LaunchAgent

Create a LaunchAgent to schedule the script to run daily at 9 PM.

1. **Create the LaunchAgents Directory (if it doesn't exist):**

   ```bash
   mkdir -p ~/Library/LaunchAgents
   ```

2. **Create the plist File:**

   ```bash
   nano ~/Library/LaunchAgents/com.yourusername.ankiemail.plist
   ```

3. **Paste the Following Content:**

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN"
     "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
     <dict>
       <key>Label</key>
       <string>com.yourusername.ankiemail</string>
       <key>ProgramArguments</key>
       <array>
         <string>/Users/your_username/run_anki_and_script.sh</string>
       </array>
       <key>StartCalendarInterval</key>
       <dict>
         <key>Hour</key>
         <integer>21</integer>
         <key>Minute</key>
         <integer>0</integer>
       </dict>
       <key>StandardOutPath</key>
       <string>/tmp/ankiemail.out</string>
       <key>StandardErrorPath</key>
       <string>/tmp/ankiemail.err</string>
       <key>EnvironmentVariables</key>
       <dict>
         <key>CLOCKIFY_API_KEY</key>
         <string>your_clockify_api_key_here</string>
       </dict>
       <key>WakeFromSleep</key>
       <true/>
     </dict>
   </plist>
   ```

   **Notes:**

   - **Replace `your_username`** with your macOS username.
   - **Adjust the Label:** `com.yourusername.ankiemail` should be unique.
   - **Set `CLOCKIFY_API_KEY`** with your actual Clockify API key in the plist file's `EnvironmentVariables` section. This ensures the API key is available when the script runs via LaunchAgent.

4. **Load the LaunchAgent:**

   ```bash
   launchctl load ~/Library/LaunchAgents/com.yourusername.ankiemail.plist
   ```

---

## Scheduling Your Mac to Wake from Sleep (Optional)

To ensure your Mac wakes up to run the script:

1. **Open Terminal.**

2. **Schedule Daily Wake:**

   ```bash
   sudo pmset repeat wakeorpoweron MTWRFSU 21:00:00
   ```

   - You'll be prompted for your administrator password.
   - Adjust `21:00:00` to the desired time if needed.

3. **Verify Scheduled Events:**

   ```bash
   pmset -g sched
   ```

---

## Testing the Setup

1. **Test the Python Script Manually:**

   ```bash
   cd ~/AnkiEmailSender
   source venv/bin/activate
   python3 main.py
   deactivate
   ```

   - Ensure the email is sent and there are no errors.

2. **Test the Shell Script Manually:**

   ```bash
   ~/run_anki_and_script.sh
   ```

   - Verify that Anki closes, the script runs, and Anki reopens.

3. **Check Logs:**

   - If the script runs via LaunchAgent, check logs if needed:

     ```bash
     cat /tmp/ankiemail.out
     cat /tmp/ankiemail.err
     ```

---

## Troubleshooting

- **Anki Database Access Errors:**
  - Ensure Anki is closed when the script runs.
  - Verify the `anki_profile` in `get_anki_db_path()` matches your Anki profile name.

- **Gmail API Errors:**
  - Ensure `credentials.json` is in your project directory.
  - If prompted, complete the OAuth authorization in your browser.
  - Delete `token.pickle` if it's corrupted and re-run the script to re-authenticate.

- **Clockify API Errors:**
  - Ensure the `CLOCKIFY_API_KEY` environment variable is set correctly.
  - Verify your API key is correct and hasn't expired or been revoked.
  - Check your internet connection.

- **Permission Errors:**
  - Make sure all scripts have the correct permissions (`chmod +x` for executables).
  - The `activate` script should have read permissions.

- **Module Not Found Errors:**
  - Activate the virtual environment before running the script.
  - Install missing packages using `pip install package_name`.

- **Time Zone Issues:**
  - If the times reported by Clockify are incorrect, check your system's time zone settings.

---

## Conclusion

By following this guide, you've set up an automated system that emails you daily with your Anki progress and time spent on projects tracked via Clockify. This setup leverages Python, the Gmail API, Clockify API, and macOS's scheduling capabilities to create a seamless experience.

Feel free to customize the scripts and settings to better suit your needs. Happy studying and tracking!

---

**Disclaimer:** Use this guide responsibly. Ensure you comply with Google's API usage policies, Clockify's API terms of service, and respect privacy considerations when sending emails.
