---
name: notifiche
description: Gestisci le notifiche macOS di Claude Code. Attiva/disattiva le notifiche sonore quando Claude finisce di generare o ha bisogno di attenzione. Comandi - /notifiche install (setup guidato), /notifiche on, /notifiche off, /notifiche sempre, /notifiche smart, /notifiche status, /notifiche allow-on, /notifiche allow-auto, /notifiche allow-off. Also triggers on notifiche install or notifiche setup.
---

# Notifiche macOS per Claude Code

Notifiche sonore native macOS con logo Claude che avvisano quando Claude finisce di generare o ha bisogno della tua attenzione. Cliccando sulla notifica si torna all'app giusta (Antigravity o Claude.app) in base a da dove è stata avviata la chat.

## Comandi

### `/notifiche install` o `/notifiche setup`

Setup COMPLETAMENTE AUTOMATICO. Esegui tutto senza chiedere conferma all'utente, step dopo step.

**Step 1: Installa terminal-notifier (se manca)**
```bash
if ! command -v /opt/homebrew/bin/terminal-notifier &>/dev/null; then
  /opt/homebrew/bin/brew install terminal-notifier 2>&1
fi
```
Se anche brew manca, dillo all'utente: "Installa Homebrew prima: https://brew.sh" e fermati.

**Step 2: Sostituisci icona con logo Claude (silenzioso)**
```bash
if [ -f /Applications/Claude.app/Contents/Resources/electron.icns ]; then
  cp /Applications/Claude.app/Contents/Resources/electron.icns /opt/homebrew/Cellar/terminal-notifier/*/terminal-notifier.app/Contents/Resources/Terminal.icns 2>/dev/null
  touch /opt/homebrew/Cellar/terminal-notifier/*/terminal-notifier.app 2>/dev/null
  /System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -f /opt/homebrew/Cellar/terminal-notifier/*/terminal-notifier.app 2>/dev/null
  killall NotificationCenter Dock 2>/dev/null
fi
```
Se Claude.app non esiste, salta silenziosamente.

**Step 3: Rileva l'IDE di fallback**
```bash
ps aux | grep -i "[E]lectron" | grep -oE '/Applications/[^/]+\.app' | sort -u
```
- Escludi Claude.app, ClickUp.app e altre app non-IDE.
- Estrai il nome (es. `/Applications/Antigravity.app` → `Antigravity`).
- Serve solo come DEFAULT_APP nel prossimo step (fallback se il rilevamento runtime fallisce).
- Se non trovi niente, usa `Antigravity` come default.

**Step 4: Crea lo script helper `~/.claude/scripts/notify_open.sh`**

Crea/sovrascrivi il file con questo contenuto (sostituendo `DEFAULT_APP` con il nome trovato allo step 3):

```bash
#!/bin/bash
# Invia notifica e apre l'app giusta (Antigravity o Claude.app)
# in base all'albero dei processi — funziona sia da IDE che da Claude.app desktop
MESSAGE="${1:-Notifica}"
TITLE="${2:-Claude Code}"
SOUND="${3:-Glass}"

# Risali l'albero dei processi per trovare l'app sorgente
p=$$
APP=""
for i in 1 2 3 4 5 6 7 8 9 10; do
  p=$(ps -p "$p" -o ppid= 2>/dev/null | tr -d ' ')
  [ -z "$p" ] || [ "$p" = "1" ] && break
  args=$(ps -p "$p" -o args= 2>/dev/null)
  if echo "$args" | grep -qi "antigravity"; then
    APP="Antigravity"
    break
  elif echo "$args" | grep -qi "/Applications/Claude.app"; then
    APP="Claude"
    break
  fi
done
[ -z "$APP" ] && APP="DEFAULT_APP"

/opt/homebrew/bin/terminal-notifier \
  -message "$MESSAGE" \
  -title "$TITLE" \
  -sound "$SOUND" \
  -execute "open -a '$APP'"
```

Poi rendilo eseguibile:
```bash
chmod +x ~/.claude/scripts/notify_open.sh
```

**Step 5: Configura hooks in modalita "sempre"**
Leggi `~/.claude/settings.json`, fai merge degli hooks (NON sovrascrivere permissions, env o altri hooks), e scrivi:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/scripts/notify_open.sh 'Ho finito di generare!' 'Claude Code' 'Glass'"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/scripts/notify_open.sh 'Serve la tua attenzione!' 'Claude Code' 'Submarine'"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/scripts/notify_open.sh 'Serve la tua autorizzazione!' 'Claude Code - Permesso' 'Submarine'"
          }
        ]
      }
    ]
  }
}
```

**Step 6: Invia notifica di test**
```bash
~/.claude/scripts/notify_open.sh 'Notifiche attivate!' 'Claude Code' 'Glass'
```

**Step 7: Conferma finale**
Mostra:
```
✅ Notifiche attivate!

Riceverai un avviso ogni volta che finisco di rispondere.
La notifica apre automaticamente l'app giusta (Antigravity o Claude.app)
in base a da dove è stata avviata la chat.

Se NON hai visto la notifica di test, vai in:
Impostazioni di Sistema > Notifiche > terminal-notifier → abilita

Comandi disponibili:
- /notifiche sempre  → notifica sempre (attuale)
- /notifiche smart   → notifica solo quando non guardi l'IDE
- /notifiche off     → disattiva
- /notifiche status  → mostra stato
```

---

### `/notifiche status`
Leggi `~/.claude/settings.json` e controlla se i hooks `Stop`, `Notification` e `PermissionRequest` esistono e sono attivi. Mostra lo stato attuale:
- Se gli hooks esistono e contengono il check "frontmost" → modalita "smart"
- Se gli hooks esistono senza il check → modalita "sempre"
- Se gli hooks non esistono → notifiche disattivate

### `/notifiche on`
Alias di `/notifiche smart`. Attiva le notifiche in modalita smart (default).

### `/notifiche off`
Rimuovi i hooks `Stop`, `Notification` e `PermissionRequest` da `~/.claude/settings.json`. Preserva tutto il resto del file.

### `/notifiche sempre`
Configura gli hooks senza il check dell'app in primo piano. Le notifiche arrivano SEMPRE.

Assicurati che `~/.claude/scripts/notify_open.sh` esista (crealo come nello Step 4 dell'install se manca, usando `Antigravity` come DEFAULT_APP).

Scrivi gli hooks con questo template, facendo merge con il contenuto esistente di `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [{"hooks": [{"type": "command", "command": "~/.claude/scripts/notify_open.sh 'Ho finito di generare!' 'Claude Code' 'Glass'"}]}],
    "Notification": [{"hooks": [{"type": "command", "command": "~/.claude/scripts/notify_open.sh 'Serve la tua attenzione!' 'Claude Code' 'Submarine'"}]}],
    "PermissionRequest": [{"hooks": [{"type": "command", "command": "~/.claude/scripts/notify_open.sh 'Serve la tua autorizzazione!' 'Claude Code - Permesso' 'Submarine'"}]}]
  }
}
```

### `/notifiche smart`
Come `/notifiche sempre` ma con il check dell'app in primo piano. Le notifiche arrivano solo quando nessuna app Electron (Antigravity, Claude.app) è in primo piano.

Assicurati che `~/.claude/scripts/notify_open.sh` esista (crealo come nello Step 4 dell'install se manca).

Wrappa ogni comando con il check frontmost (entrambe le app sono Electron, quindi basta controllare "Electron"):
```
FRONT=$(osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true'); if [ "$FRONT" != "Electron" ]; then ~/.claude/scripts/notify_open.sh 'MESSAGGIO' 'TITOLO' 'SUONO'; fi
```

Template hooks smart:
```json
{
  "hooks": {
    "Stop": [{"hooks": [{"type": "command", "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ]; then ~/.claude/scripts/notify_open.sh 'Ho finito di generare!' 'Claude Code' 'Glass'; fi"}]}],
    "Notification": [{"hooks": [{"type": "command", "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ]; then ~/.claude/scripts/notify_open.sh 'Serve la tua attenzione!' 'Claude Code' 'Submarine'; fi"}]}],
    "PermissionRequest": [{"hooks": [{"type": "command", "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ]; then ~/.claude/scripts/notify_open.sh 'Serve la tua autorizzazione!' 'Claude Code - Permesso' 'Submarine'; fi"}]}]
  }
}
```

---

### `/notifiche allow-on`
Attiva l'**auto-allow da notifica**: quando Claude chiede un permesso, cliccando la notifica viene automaticamente approvato senza intervenire nella UI.

**Step 1: Crea `~/.claude/scripts/allow_permission.sh`**
```bash
#!/bin/bash
# Clicca automaticamente "Allow" in Claude Code quando invocato dalla notifica
APP_NAME="$1"
[ -z "$APP_NAME" ] && APP_NAME="Claude"

osascript <<EOF
tell application "System Events"
    tell process "$APP_NAME"
        set didClick to false
        try
            click button "Allow" of window 1
            set didClick to true
        end try
        if not didClick then
            try
                set allGroups to every UI element of window 1
                repeat with el in allGroups
                    try
                        click button "Allow" of el
                        set didClick to true
                        exit repeat
                    end try
                end repeat
            end try
        end if
        if not didClick then
            key code 36
        end if
    end tell
end tell
EOF
```
```bash
chmod +x ~/.claude/scripts/allow_permission.sh
```

**Step 2: Crea `~/.claude/scripts/notify_allow.sh`**
```bash
#!/bin/bash
# Notifica permesso — al clic approva automaticamente in Claude Code
MESSAGE="${1:-Permesso richiesto}"
TITLE="${2:-Claude Code - Permesso}"
SOUND="${3:-Submarine}"

# Risali l'albero dei processi per trovare l'app sorgente
p=$$
APP=""
for i in 1 2 3 4 5 6 7 8 9 10; do
  p=$(ps -p "$p" -o ppid= 2>/dev/null | tr -d ' ')
  [ -z "$p" ] || [ "$p" = "1" ] && break
  args=$(ps -p "$p" -o args= 2>/dev/null)
  if echo "$args" | grep -qi "antigravity"; then
    APP="Antigravity"
    break
  elif echo "$args" | grep -qi "/Applications/Claude.app"; then
    APP="Claude"
    break
  fi
done
[ -z "$APP" ] && APP="Antigravity"

/opt/homebrew/bin/terminal-notifier \
  -message "$MESSAGE" \
  -title "$TITLE" \
  -sound "$SOUND" \
  -execute "~/.claude/scripts/allow_permission.sh '$APP'"
```
```bash
chmod +x ~/.claude/scripts/notify_allow.sh
```

**Step 3: Aggiorna il hook `PermissionRequest` in `~/.claude/settings.json`**
Leggi il file, sostituisci SOLO il comando del hook `PermissionRequest` con:
```
~/.claude/scripts/notify_allow.sh 'Clicca per approvare' 'Claude Code - Permesso' 'Submarine'
```
Preserva tutto il resto invariato (Stop, Notification, permissions, ecc.).

**Step 4: Conferma**
```
✅ Auto-allow attivato!

Quando Claude chiede un permesso, clicca la notifica per approvare automaticamente.
```

---

### `/notifiche allow-auto`
Attiva l'**auto-approvazione silenziosa**: quando Claude chiede un permesso, viene approvato automaticamente via AppleScript senza notifica e senza che l'utente tocchi nulla.

**Step 1: Crea `~/.claude/scripts/auto_allow.sh`**
```bash
#!/bin/bash
# Auto-approva il permesso in Claude Code senza interazione utente

p=$$
APP=""
for i in 1 2 3 4 5 6 7 8 9 10; do
  p=$(ps -p "$p" -o ppid= 2>/dev/null | tr -d ' ')
  [ -z "$p" ] || [ "$p" = "1" ] && break
  args=$(ps -p "$p" -o args= 2>/dev/null)
  if echo "$args" | grep -qi "antigravity"; then
    APP="Antigravity"
    break
  elif echo "$args" | grep -qi "/Applications/Claude.app"; then
    APP="Claude"
    break
  fi
done
[ -z "$APP" ] && APP="Antigravity"

osascript <<EOF
tell application "System Events"
    tell process "$APP"
        set didClick to false
        try
            click button "Allow" of window 1
            set didClick to true
        end try
        if not didClick then
            try
                set allGroups to every UI element of window 1
                repeat with el in allGroups
                    try
                        click button "Allow" of el
                        set didClick to true
                        exit repeat
                    end try
                end repeat
            end try
        end if
        if not didClick then
            key code 36
        end if
    end tell
end tell
EOF
```
```bash
chmod +x ~/.claude/scripts/auto_allow.sh
```

**Step 2: Aggiorna il hook `PermissionRequest` in `~/.claude/settings.json`**
Sostituisci SOLO il comando del hook `PermissionRequest` con:
```
~/.claude/scripts/auto_allow.sh
```

**Step 3: Conferma**
```
✅ Auto-approvazione silenziosa attivata!

I permessi vengono approvati automaticamente senza nessuna interazione.
```

---

### `/notifiche allow-off`
Disattiva l'auto-allow: la notifica di permesso torna ad aprire solo l'app (comportamento normale).

Leggi `~/.claude/settings.json`, sostituisci SOLO il comando del hook `PermissionRequest` con:
```
~/.claude/scripts/notify_open.sh 'Serve la tua autorizzazione!' 'Claude Code - Permesso' 'Submarine'
```
Preserva tutto il resto invariato.

Conferma:
```
✅ Auto-allow disattivato.

La notifica di permesso ora apre Claude senza approvare automaticamente.
```

---

## Regole importanti

1. **Leggi SEMPRE** `~/.claude/settings.json` prima di modificarlo
2. **Merge** con il contenuto esistente — non sovrascrivere mai permissions, env, o altri hooks
3. **Valida** il JSON dopo ogni modifica con `jq -e . ~/.claude/settings.json`
4. Conferma all'utente lo stato finale dopo ogni operazione
5. **Usa `-execute 'open -a APP_NAME'`** e MAI `-activate BUNDLE_ID` per il click-to-open
6. Durante `/notifiche install` fai TUTTO automaticamente, non chiedere conferme inutili
7. Lo script `notify_open.sh` gestisce automaticamente il rilevamento dell'app — non servono più rilevamenti manuali dell'IDE nei hooks
