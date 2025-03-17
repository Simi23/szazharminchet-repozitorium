---
title: Asterisk Offline TTS + Playback
description: 
published: true
date: 2025-03-17T10:24:25.707Z
tags: linux
editor: markdown
dateCreated: 2025-03-17T09:15:45.172Z
---

# Offline TTS + Playback
When getting an email from Postfix convert it with TTS to .wav and play it in a call.

## Postfix

You have to have a working postfix. You have to have the emailaddress what you want to use in your alias_maps. In this configuration we will use `emailspeak@company.com`.

Edit `/etc/postfix/transport`:
```
...

emailspeak@company.com emailspeak:
```
Map this to postfix:
```
postmap /etc/postfix/transport
```

And edit `main.cf`:
```
...

transport_maps = hash:/etc/postfix/transport

...
```

You have to edit `master.cf` too. Add this to the end:
```
...

emailspeak      unix - n n - - pipe
  flags=Rq user=asterisk argv=/etc/postfix/scripts/tts.py ${sender}
```

Restart the service
```
systemctl reload postfix
```

## Python

Install packages for TTS and audio converting.
```
apt install espeak sox python3
```

Code:
```
#!/usr/bin/python3
import sys
import email
import os
import shutil
import subprocess

def parse_email():
    # Read email from stdin
    raw_email = sys.stdin.read()
    msg = email.message_from_string(raw_email)

    subject = msg["Subject"].strip()
    sender = msg["From"]

    if msg.is_multipart():
        for part in msg.walk():
            if part.get_content_type() == "text/plain":
                body = part.get_payload(decode=True).decode("utf-8", errors="ignore").strip()
                break
    else:
        body = msg.get_payload(decode=True).decode("utf-8", errors="ignore").strip()
       
    # Call the TTS and call initatior definition
    initiate_call(sender, subject, body)

def initiate_call(sender, subject, message):
		# Filenames
    call_filename = f"/tmp/call_{subject}.call"
    call2_filename = f"/var/spool/asterisk/outgoing/call_{subject}.call"
    tts_file = "/tmp/playback.wav"
    tts2_file = "/tmp/playback2.wav"
    to_tts_file = "/var/lib/asterisk/sounds/en/playback.wav"
    fulltext_file = f"/tmp/text-{subject}"

		# Paste the text into a file for TTS
    subprocess.run(f"touch /tmp/text && chmod 777 /tmp/text", shell=True)
    with open(fulltext_file, 'w') as f:
        f.write(f"You got an email from {sender}. The email: {message}.")

    try:
    		# Create the audio file convert it and place it to the right place.
        subprocess.run(f'espeak -f {fulltext_file} -w {tts_file} -b 1', shell=True)
        subprocess.run(f'sox {tts_file} -r 8000 -c 1 -e signed-integer {tts2_file}', shell=True)
        subprocess.run(f"chmod 777 {tts2_file}", shell=True)
        subprocess.run(f"cp {tts2_file} {to_tts_file}", shell=True)
        subprocess.run(f"chmod 777 {to_tts_file}", shell=True)

				# Create the call file and MOVE it to /var/spool/asterisk/outgoing/
        with open(call_filename, 'w') as f:
            f.write(f"Channel: PJSIP/{subject}\nMaxRetries: 1\nRetryTime: 60\nWaitTime: 30\nContext: internal\nExtension: 1234\nPriority: 1\n")
            
        subprocess.run(f"chmod 777 {call_filename}", shell=True)
        subprocess.run(f"mv {call_filename} {call2_filename}", shell=True)


    except PermissionError as e:
        print(f"Permission error: {e}")
    except Exception as e:
        print(f"Unexpected error: {e}")

if __name__ == "__main__":
    parse_email()
```

## Asterisk

`extensions.conf`
```
exten = 1234,1,Answer()
same = n,Wait(1)
same = n,Playback(playback)
same = n,Hangup()
```

```
asterisk -rx "dialplan reload"
```

## End
Now your config is done. You can send an email to emailspeak@company.com and then it will call the subject with your messeage.