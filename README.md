# go beyond: automate your anki progress emails with clockify integration on macos

**this is your moment.**

baumol's cost disease is crippling the world you love. but if you step up, use accountability, and ignite a new heart within yourself—a culture hellbent on getting stuff done, creating value, and making things—you will do good.

you will make your grandfather proud.

you will craft a world your kids are happy to live in.

and you will begin chopping down the cancerous growth of baumol's cost disease. 🪓

let's get started.

---

## table of contents

- [introduction](#introduction)
- [prerequisites](#prerequisites)
- [setup overview](#setup-overview)
- [1. install python 3](#1-install-python-3)
- [2. create your project environment](#2-create-your-project-environment)
- [3. configure gmail api credentials](#3-configure-gmail-api-credentials)
- [4. integrate clockify](#4-integrate-clockify)
- [5. craft the shell script](#5-craft-the-shell-script)
- [6. schedule with launchagent](#6-schedule-with-launchagent)
- [7. wake your mac (optional)](#7-wake-your-mac-optional)
- [8. test and triumph](#8-test-and-triumph)
- [conclusion](#conclusion)

---

## introduction

we're building a system that:

- **closes anki** if it's running.
- **runs a python script** to gather your anki progress and clockify time entries.
- **sends you an email** summarizing your achievements.
- **reopens anki** after the script runs.
- **automates the process daily** at a specified time.
- **wakes your mac** to ensure nothing stops you.

it's time to take action, create value, and make things happen.

---

## prerequisites

- **macos** computer.
- **anki** installed.
- **python 3** installed.
- **clockify** account.
- **determination** to get stuff done.

---

## setup overview

we'll:

1. install python 3.
2. set up your project environment.
3. configure gmail api credentials.
4. integrate clockify for time tracking.
5. craft a shell script to automate tasks.
6. schedule the script with launchagent.
7. (optional) configure your mac to wake up.
8. test everything to ensure success.

---

## 1. install python 3

first things first—let's equip your mac with python 3.

```bash
# install homebrew if you haven't already
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# install python 3
brew install python3
```

---

## 2. create your project environment

time to set up a dedicated space for your automation project.

```bash
# create a directory for your project
mkdir ~/ankiautomation
cd ~/ankiautomation

# set up a virtual environment
python3 -m venv venv

# activate the virtual environment
source venv/bin/activate

# install required python packages
pip install google-auth google-auth-oauthlib google-api-python-client isodate requests
```

---

## 3. configure gmail api credentials

### **step up and enable the gmail api**

1. go to the [google cloud console](https://console.cloud.google.com/).
2. create or select a project.
3. navigate to **apis & services** > **library**.
4. search for **gmail api** and enable it.

### **craft your oauth credentials**

1. go to **apis & services** > **credentials**.
2. click **create credentials** > **oauth client id**.
3. configure the consent screen (set to external and fill required fields).
4. choose **desktop app** and create.
5. download the `credentials.json` file.

### **place credentials into your project**

move `credentials.json` into your project directory:

```bash
mv ~/Downloads/credentials.json ~/ankiautomation/
```

---

## 4. integrate clockify

### **obtain your clockify api key**

1. log in to your [clockify](https://clockify.me/) account.
2. click on your profile avatar > **profile settings**.
3. scroll to the **api** section and copy your api key.

### **set your api key as an environment variable**

edit your shell profile to include your clockify api key:

```bash
# for bash users
nano ~/.bash_profile

# for zsh users
nano ~/.zshrc
```

add the following line (replace with your actual api key):

```bash
export CLOCKIFY_API_KEY="your_clockify_api_key_here"
```

save and exit. then, reload your profile:

```bash
# for bash users
source ~/.bash_profile

# for zsh users
source ~/.zshrc
```

---

## 5. craft the shell script

this script is the heart of your automation—it's hellbent on getting stuff done.

create the script:

```bash
nano ~/run_anki_and_script.sh
```

paste the following (replace `/Users/your_username/ankiautomation` with your actual path):

```bash
#!/bin/bash

# close anki if it's running
pkill -f anki

# wait to ensure anki has closed
sleep 5

# navigate to your project directory
cd "/Users/your_username/ankiautomation"

# activate virtual environment
source "venv/bin/activate"

# run the python script
python3 main.py

# deactivate virtual environment
deactivate

# wait to ensure the script finishes
sleep 5

# open anki
open "/applications/anki.app"
```

make the script executable:

```bash
chmod +x ~/run_anki_and_script.sh
```

---

## 6. schedule with launchagent

let's automate this process daily, chopping down inefficiency.

create a launchagent plist file:

```bash
mkdir -p ~/library/launchagents
nano ~/library/launchagents/com.yourusername.ankiautomation.plist
```

paste the following content (replace placeholders accordingly):

```xml
<?xml version="1.0" encoding="utf-8"?>
<!doctype plist public "-//apple computer//dtd plist 1.0//en"
  "http://www.apple.com/dtds/propertylist-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>label</key>
    <string>com.yourusername.ankiautomation</string>
    <key>programarguments</key>
    <array>
      <string>/users/your_username/run_anki_and_script.sh</string>
    </array>
    <key>startcalendarinterval</key>
    <dict>
      <key>hour</key>
      <integer>21</integer>
      <key>minute</key>
      <integer>0</integer>
    </dict>
    <key>standardoutpath</key>
    <string>/tmp/ankiautomation.out</string>
    <key>standarderrorpath</key>
    <string>/tmp/ankiautomation.err</string>
    <key>environmentvariables</key>
    <dict>
      <key>CLOCKIFY_API_KEY</key>
      <string>your_clockify_api_key_here</string>
    </dict>
    <key>wakefromsleep</key>
    <true/>
  </dict>
</plist>
```

load the launchagent:

```bash
launchctl load ~/library/launchagents/com.yourusername.ankiautomation.plist
```

---

## 7. wake your mac (optional)

don't let sleep stop you from making progress.

```bash
sudo pmset repeat wakeorpoweron mtwrfsu 21:00:00
```

verify the schedule:

```bash
pmset -g sched
```

---

## 8. test and triumph

### **test the python script**

```bash
cd ~/ankiautomation
source venv/bin/activate
python3 main.py
deactivate
```

ensure the email is sent and data is accurate.

### **test the shell script**

```bash
~/run_anki_and_script.sh
```

verify anki closes, the script runs, and anki reopens.

### **check logs**

if issues arise, check:

```bash
cat /tmp/ankiautomation.out
cat /tmp/ankiautomation.err
```

---

## conclusion

you've stepped up. you've used accountability to create value and get things done. you've automated your anki progress tracking, integrated clockify, and set up a system that works tirelessly—even when you rest.

you've made your grandfather proud.

you've taken a swing at baumol's cost disease. 🪓

keep pushing forward. create. build. innovate. the world your kids inherit is shaped by the actions you take today.

---

**disclaimer:** use this guide responsibly. ensure you comply with all relevant api usage policies and respect privacy considerations when sending emails.
