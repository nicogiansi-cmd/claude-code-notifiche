---
name: notifiche
description: Gestisci le notifiche macOS di Claude Code. Attiva/disattiva le notifiche sonore quando Claude finisce di generare o ha bisogno di attenzione. Comandi - /notifiche install (setup guidato), /notifiche on, /notifiche off, /notifiche sempre, /notifiche smart, /notifiche status. Also triggers on notifiche install or notifiche setup.
---

# Notifiche macOS per Claude Code

Notifiche sonore native macOS con logo Claude che avvisano quando Claude finisce di generare o ha bisogno della tua attenzione. Cliccando sulla notifica si torna all'app IDE.

## Comandi

### `/notifiche install` o `/notifiche setup`

Setup COMPLETAMENTE AUTOMATICO. Esegui tutto senza chiedere conferma all'utente, step dopo step. Chiedi solo se strettamente necessario (es. quale IDE usano se ce ne sono piu di uno).

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

**Step 3: Rileva l'IDE in uso (automatico)**
```bash
ps aux | grep -i "[E]lectron" | grep -oE '/Applications/[^/]+\.app' | sort -u
```
- Se trovi UNA sola app (escludi Claude.app, ClickUp.app e altre app non-IDE): usala automaticamente.
- Se trovi PIU app che potrebbero essere IDE: chiedi all'utente quale usano.
- Se non trovi niente: chiedi all'utente il nome della loro app IDE.
- Estrai il nome app dal path (es. `/Applications/Antigravity.app` → `Antigravity`, `/Applications/Cursor.app` → `Cursor`).

**Step 4: Configura hooks in modalita "sempre"**
Leggi `~/.claude/settings.json`, fai merge degli hooks (NON sovrascrivere permissions, env o altri hooks), e scrivi:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/bin/terminal-notifier -message 'Ho finito di generare!' -title 'Claude Code' -sound Glass -execute 'open -a IDE_APP_NAME'"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/bin/terminal-notifier -message 'Serve la tua attenzione!' -title 'Claude Code' -sound Submarine -execute 'open -a IDE_APP_NAME'"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/bin/terminal-notifier -message 'Serve la tua autorizzazione!' -title 'Claude Code - Permesso' -sound Submarine -execute 'open -a IDE_APP_NAME'"
          }
        ]
      }
    ]
  }
}
```

**Step 5: Invia notifica di test**
```bash
/opt/homebrew/bin/terminal-notifier -message 'Notifiche attivate!' -title 'Claude Code' -sound Glass -execute 'open -a IDE_APP_NAME'
```

**Step 6: Conferma finale**
Mostra:
```
✅ Notifiche attivate!

Riceverai un avviso ogni volta che finisco di rispondere.
Clicca sulla notifica per tornare all'IDE.

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

Rileva l'IDE automaticamente come nello step 3 dell'install. Usa `-execute 'open -a APP_NAME'` (MAI `-activate BUNDLE_ID`).

Scrivi gli hooks come nel template sopra, facendo merge con il contenuto esistente di `~/.claude/settings.json`.

### `/notifiche smart`
Come `/notifiche sempre` ma con il check dell'app in primo piano. Le notifiche arrivano solo quando l'IDE NON e in primo piano.

Stessa logica di rilevamento app. Wrappa ogni comando con:
```
FRONT=$(osascript -e 'tell application "System Events" to get name of first application process whose frontmost is true'); if [ "$FRONT" != "Electron" ] && [ "$FRONT" != "Code" ]; then COMANDO_NOTIFICA; fi
```

## Regole importanti

1. **Leggi SEMPRE** `~/.claude/settings.json` prima di modificarlo
2. **Merge** con il contenuto esistente — non sovrascrivere mai permissions, env, o altri hooks
3. **Valida** il JSON dopo ogni modifica con `jq -e . ~/.claude/settings.json`
4. Conferma all'utente lo stato finale dopo ogni operazione
5. **Usa `-execute 'open -a APP_NAME'`** e MAI `-activate BUNDLE_ID` per il click-to-open
6. Durante `/notifiche install` fai TUTTO automaticamente, non chiedere conferme inutili
