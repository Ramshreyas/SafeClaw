# Feed Granola Meetings to Your OpenClaw Bot

Your Granola app stores all meeting data locally on your Mac at:
```
~/Library/Application Support/Granola/cache-v3.json
```

Your OpenClaw bot runs in Docker and can read files from:
```
~/openclaw-secure/data/
```

A cron job runs every 20 minutes, calls a sync script, and writes a filtered JSON file containing:

- **Last 3 days and today** — title, time, and AI-generated notes if available
- **Next 2 days** — title and time, so the bot knows what's coming up

## Create the Sync Script

Run this to create the script — copy and paste the whole block:

```bash
cat > ~/openclaw-secure/granola-sync.py << 'EOF'
import json, os
from datetime import date, timedelta, datetime

p = os.path.expanduser
raw = open(p('~/Library/Application Support/Granola/cache-v3.json')).read()
state = json.loads(json.loads(raw)['cache'])['state']
docs = state.get('documents', {})
panels = state.get('documentPanels', {})
events = state.get('events', [])

def txt(node):
    if not isinstance(node, dict):
        return ''
    if node.get('type') == 'text':
        for mark in node.get('marks', []):
            if isinstance(mark, dict) and mark.get('type') == 'link':
                if 'notes.granola.ai/t/' in mark.get('attrs', {}).get('href', ''):
                    return ''
        return node.get('text', '')
    return '\n'.join(filter(None, [txt(c) for c in node.get('content', [])]))

panel_text = {
    doc_id: '\n'.join(filter(None, [txt(pn.get('content', {})) for pn in pd.values()]))
    for doc_id, pd in panels.items() if isinstance(pd, dict)
}

today = date.today()
ago = today - timedelta(days=3)
soon = today + timedelta(days=2)

out = []
seen_titles = set()

for doc_id, v in docs.items():
    title = v.get('title', '')
    if not title:
        continue
    g = v.get('google_calendar_event') or {}
    date_str = (
        g.get('start', {}).get('dateTime') or
        g.get('start', {}).get('date') or
        v.get('created_at', '')
    ) if isinstance(g, dict) else v.get('created_at', '')
    if not date_str or len(date_str) < 10:
        continue
    try:
        meeting_date = date.fromisoformat(date_str[:10])
    except Exception:
        continue
    if not (ago <= meeting_date <= soon):
        continue
    end_str = g.get('end', {}).get('dateTime') or g.get('end', {}).get('date', '')
    dt = date_str[:16].replace('T', ' ')
    if end_str and len(end_str) >= 16:
        dt += ' – ' + end_str[11:16]
    entry = {'title': title, 'datetime': dt}
    notes = panel_text.get(doc_id, '')
    if notes:
        entry['notes'] = notes
    out.append(entry)
    seen_titles.add(title)

for e in events:
    title = e.get('summary', e.get('title', ''))
    if not title or title in seen_titles:
        continue
    start = e.get('start', {})
    date_str = start.get('dateTime', start.get('date', ''))
    if not date_str or len(date_str) < 10:
        continue
    try:
        event_date = date.fromisoformat(date_str[:10])
    except Exception:
        continue
    if not (today <= event_date <= soon):
        continue
    end_str = e.get('end', {}).get('dateTime') or e.get('end', {}).get('date', '')
    dt = date_str[:16].replace('T', ' ')
    if end_str and len(end_str) >= 16:
        dt += ' – ' + end_str[11:16]
    out.append({'title': title, 'datetime': dt})

out.sort(key=lambda x: x['datetime'])

out_path = p('~/openclaw-secure/data/granola-cache.json')
json.dump(out, open(out_path, 'w'), indent=2)
print(f'[{datetime.now().strftime("%Y-%m-%d %H:%M")}] wrote {len(out)} meetings')
EOF
```

## Set Up the Cron Job

Run this to add the cron job without opening an editor:

```bash
(crontab -l 2>/dev/null; echo "*/20 * * * * /usr/bin/python3 \$HOME/openclaw-secure/granola-sync.py >> \$HOME/openclaw-secure/data/granola-sync.log 2>&1") | crontab -
```

Confirm it was added:
```bash
crontab -l
```


The cron job is active immediately. Your bot will now automatically receive fresh meeting data every 20 minutes — you're all set.

---

## Extra

### Managing Your Cron Jobs

```bash
# List all your cron jobs
crontab -l

# Edit cron jobs
EDITOR=nano crontab -e

# Remove ALL cron jobs (careful)
crontab -r
```

To **pause** a job without deleting it, edit and add `#` at the start of the line:
```
# */20 * * * * /usr/bin/python3 ...
```

### What's Happening End to End

```
Granola app records meetings
        ↓
Saves to ~/Library/Application Support/Granola/cache-v3.json
        ↓
Cron runs granola-sync.py every 20 minutes
        ↓
Writes filtered meetings to ~/openclaw-secure/data/granola-cache.json
        ↓
Docker mounts ~/openclaw-secure/data/ into the container
        ↓
Bot reads the file and sends you a Telegram summary
```

No auth tokens leave your Mac. No API calls. The bot only sees what you give it.
