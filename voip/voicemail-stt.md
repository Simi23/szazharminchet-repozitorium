---
title: Voicemail with Speech-to-text
description: Asterisk voicemail with python-based speech-to-text email notification
published: true
date: 2025-03-18T15:06:27.873Z
tags: linux
editor: markdown
dateCreated: 2025-03-18T15:06:27.873Z
---

# Asterisk configuration

Edit <kbd>voicemail.conf</kbd>

```ini
[general]
format = wav
serveremail = voicemail@dmz.worldskills.org
attach = yes
mailcmd = /voicemails/voicemail/bin/python3 /voicemails/process.py
review = yes

[dmz]
6001 = 1001,Jamie,jamie.oliver@dmz.worldskills.org
6002 = 1002,Jamie,jamie.oliver@dmz.worldskills.org
6003 = 1003,Jamie,jamie.oliver@dmz.worldskills.org
```

For now, all voicemail-related emails will go to `jamie.oliver@dmz.worldskills.org`. The `process.py` script will have to be created.

Edit <kbd>extensions.conf</kbd> to include a **VoiceMail** application after the **Dial** app times out. For testing, the **Dial** app only has a timeout of 10 seconds. The functionality to listen to voice mails will be added, too.

```ini
[internal]
; 001: Listen voicemails
exten = 001,1,VoiceMailMain(${CALLERID(num)}@dmz)
same = n,Hangup()

; 60XX: Dial people directly
exten = _60XX,1,Dial(PJSIP/${EXTEN},10,TtKk)
same = n,VoiceMail(${EXTEN}@dmz)
same = n,PlayBack(vm-goodbye)
same = n,HangUp()
```

# The Python script

Create the directory, allow access to everyone, then initiate the Python virtual environment. The script will require an external module called **vosk**.

```bash
mkdir /voicemails && cd /voicemails
chmod 777 .

python3 -m venv voicemail
source voicemail/bin/activate

pip3 install vosk
```

The **Vosk** library **needs models to work**, which can be downloaded separately. Take a look at all the [models available](https://alphacephei.com/vosk/models).

We will use a small model because the large ones require **~16GB** of memory.

```bash
wget https://alphacephei.com/vosk/models/vosk-model-small-en-us-0.15.zip
unzip vosk-model-small-en-us-0.15.zip
mv vosk-model-small-en-us-0.15/ model/
```

Now, create the script file called <kbd>process.py</kbd>

```python
#!/voicemails/voicemail/bin/python3

import email, random, sys, os, base64, subprocess, json
from vosk import Model, KaldiRecognizer, SetLogLevel

# Generate a random ID for temporary files
mailid = str(random.randint(1,100000000))

# Read the email from STDIN
raw_msg = sys.stdin.read()
emailmsg = email.message_from_string(raw_msg)

# Get the attachment in ASCII encoding
att_ascii = ""
for attachment in emailmsg.walk():
    ct_dp = attachment.get_content_disposition()
    if ct_dp == "attachment":
        att_ascii = attachment.get_payload().replace("\n", "").encode("ascii")
        
filename = f"/voicemails/vm_{mailid}.wav"

# Decode the base64 attachment into binary and write it to a file
with open(filename, "wb") as f:
    decoded = base64.decodebytes(att_ascii)
    f.write(decoded)
    print(len(decoded))


SAMPLE_RATE = 16000

# Initialize the voice recognition model
model = Model("/voicemails/model", lang="en-us")
rec = KaldiRecognizer(model, SAMPLE_RATE)

results = []

# Create a subprocess and convert the waveform into text
with subprocess.Popen(["ffmpeg", "-loglevel", "quiet", "-i",
                            filename,
                            "-ar", str(SAMPLE_RATE) , "-ac", "1", "-f", "s16le", "-"],
                            stdout=subprocess.PIPE) as process:

    while True:
        data = process.stdout.read(4000)
        if len(data) == 0:
            break
        if rec.AcceptWaveform(data):
            results.append(json.loads(rec.Result())["text"])

# Delete the wave file which is no longer needed
os.system(f"rm {filename}")

# Extract the recipient from the original email
recipient_name = emailmsg["To"].split('<')[0].strip('" ')
recipient_email = emailmsg["To"].split('<')[1].strip('>" ')

# Construct the new mail body
newmail = [
  "Subject: Missed call transcript",
  "",
  "You have a new missed call. To listen to your voicemails, call extension 001 and input your mailbox code. The transcript reads:",
  ""
]
results = [s.capitalize() for s in results]
newmail.extend(results)


# Write the email body to a text file
textfile = f"/voicemails/vm_{mailid}.txt"
with open(textfile, "w") as f:
    f.writelines([l + "\n" for l in newmail])

sender_name = "Asterisk Voicemails"
sender_mail = "asterisk@dmz.worldskills.org"

# Send the mail
sendmail_command = f"/usr/sbin/sendmail -F '{sender_name}' -f '{sender_mail}' {recipient_email} < {textfile}"
os.system(sendmail_command)

# Delete the text file which is no longer needed
os.system(f"rm {textfile}")
```