---
title: Asterisk Online TTS from Email
description: Playback the body of an email in a call
published: true
date: 2025-03-17T09:57:54.054Z
tags: linux
editor: markdown
dateCreated: 2025-03-17T09:57:54.054Z
---

# Postfix configuration

This guide will use the address **spea<span>k@dmz.worldskills.o</span>rg** as the mailbox that will receive emails and "convert" them into TTS calls. This account already exists in LDAP.

Create the transport map for this address in <kbd>/etc/postfix/main.cf</kbd>.

```c
...

transport_maps = hash:/etc/postfix/transport
```

Edit <kbd>/etc/postfix/transport</kbd>.

```c
speak@dmz.worldskills.org   ttsreader:
```

Run `postmap /etc/postfix/transport` to create the hash file.

Create the transport definition in <kbd>/etc/postfix/master.cf</kbd>.

```c
ttsreader unix - n n - - pipe
  flags=Rq user=administrator
  argv=/etc/postfix/scripts/py/venv/bin/python3 /etc/postfix/scripts/py/tts.py
```

The long command is needed because we need a virtual environment to use external packages.

## Python script

Create the directory of the python script and initiate a virtual environment. Then, enter the virtual environment and install the required packages.

```bash
# Make sure python is installed
apt install python3-venv python3-pip

mkdir -p /etc/postfix/scripts/py
cd /etc/postfix/scripts/py

python3 -m venv venv

source venv/bin/activate
pip install gTTS
```

Create the following python script <kbd>/etc/postfix/scripts/py/tts.py</kbd>

```py
#!/etc/postfix/scripts/py/venv/bin/python3

from gtts import gTTS
import email, sys, os, random, time
from email.iterators import typed_subpart_iterator

callid = str(random.randint(1,1000000))

def create_callfile(filename):
  return f"""Channel: PJSIP/6001
Callerid: "Email TTS" <9999>
Context: internal
Extension: 9999
Priority: 1
Setvar: FILENAME={filename}
"""

def get_charset(message, default="ascii"):
  if message.get_content_charset():
    return message.get_content_charset()
  if message.get_charset():
    return message.get_charset()
  return default

def get_body(message):
  if message.is_multipart():
    text_parts = [part for part in typed_subpart_iterator(message, 'text', 'plain')]
    body = []
    for part in text_parts:
      charset = get_charset(part, get_charset(message))
      body.append(str(part.get_payload(decode=True), charset, 'replace'))
    return u"\n".join(body).strip()
  else:
    body = str(message.get_payload(decode=True), get_charset(message), 'replace')
    return body.strip()

raw_msg = sys.stdin.read()
emailmsg = email.message_from_string(raw_msg)

msgfrom = emailmsg['From'].split('<')[0].strip()
start = f"Email received from {msgfrom}. "

msgbody = start + get_body(emailmsg)

# For debugging purposes, you can write the message to be converted to speech into a txt file.
with open(f'/tmp/{callid}.txt', 'w') as f:
  f.write(msgbody)

tts = gTTS(msgbody)
tts.save(f'/tmp/{callid}.mp3')

filename = f'/tmp/wav{callid}'

convertcommand = f'ffmpeg -i /tmp/{callid}.mp3 -acodec pcm_s16le -ac 1 -ar 8000 {filename}.wav'
os.system(convertcommand)

modcommand = f'chmod 644 {filename}.wav'
os.system(modcommand)

callfile = f'/home/administrator/{callid}.call'

with open(callfile, 'w') as f:
  f.write(create_callfile(filename))

chmodcall = f'chmod 777 {callfile}'
os.system(chmodcall)

mvcommand = f'mv {callfile} /var/spool/asterisk/outgoing/'
os.system(mvcommand)
```

Make sure the files created belong to the administrator as that user will be running the script.

```bash
chown -R administrator:administrator /etc/postfix/scripts/py
```

# Asterisk configuration

Make sure the **pbx_spool** module is loaded. The application will call extension **6001**, from extension **9999**. Both extension are in the context **internal**.

```ini
[internal]
...

exten = 9999,1,Answer()
same = n,Wait(1)
same = n,Playback(${FILENAME})
same = n,Wait(1)
same = n,Hangup()
```

Now reload Postfix and Asterisk and watch the logs.

```bash
systemctl reload postfix

asterisk -rx "core restart now"
asterisk -rvvvvv
```

Whenever a mail is sent to **spea<span>k@dmz.worldskills.o</span>rg**, the extension **6001** will be dialed and the email message will be read out loud.