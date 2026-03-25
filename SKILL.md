---
name: notifiche
description: Gestisci le notifiche macOS di Claude Code. Attiva/disattiva le notifiche sonore quando Claude finisce di generare o ha bisogno di attenzione. Comandi - /notifiche on, /notifiche off, /notifiche sempre (anche se stai guardando), /notifiche smart (solo quando non guardi), /notifiche status.
---

# Notifiche macOS per Claude Code

Notifiche sonore native macOS che avvisano quando Claude finisce di generare o ha bisogno della tua attenzione. Cliccando sulla notifica si torna all'app IDE.

## Prerequisiti

- `terminal-notifier` installato: `brew install terminal-notifier`
- Abilitare le notifiche di terminal-notifier in **Impostazioni di Sistema > Notifiche > terminal-notifier**
- Conoscere il bundle ID della propria app IDE (es. `com.google.antigravity`, `com.microsoft.VSCode`). Per trovarlo: `defaults read /Applications/NOMEAPP.app/Contents/Info.plist CFBundleIdentifier`

## Comandi

### `/notifiche status`
Leggi `~/.claude/settings.json` e controlla se i hooks `Stop`, `Notification` e `PermissionRequest` esistono e sono attivi. Mostra lo stato attuale:
- Se gli hooks esistono e contengono il check "frontmost" -> modalita "smart" (solo quando non guardi)
- Se gli hooks esistono senza il check -> modalita "sempre"
- Se gli hooks non esistono -> notifiche disattivate

### `/notifiche on`
Alias di `/notifiche smart`. Attiva le notifiche in modalita smart (default).

### `/notifiche off`
Rimuovi i hooks `Stop`, `Notification` e `PermissionRequest` da `~/.claude/settings.json`. Preserva tutto il resto del file.

Leggi il file, rimuovi le chiavi `Stop`, `Notification` e `PermissionRequest` dall'oggetto `hooks`. Se `hooks` rimane vuoto, rimuovi anche la chiave `hooks`.

### `/notifiche sempre`
Configura gli hooks senza il check dell'app in primo piano. Le notifiche arrivano SEMPRE, anche se stai guardando l'IDE.

Hooks da scrivere in `~/.claude/settings.json` (merge con contenuto esistente):

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/bin/terminal-notifier -message 'Ho finito di generare!' -title 'Claude Code' -sound Glass -activate com.google.antigravity"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/bin/terminal-notifier -message 'Serve la tua attenzione!' -title 'Claude Code' -sound Submarine -activate com.google.antigravity"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/opt/homebrew/bin/terminal-notifier -message 'Serve la tua autorizzazione!' -title 'Claude Code - Permesso' -sound Submarine -activate com.google.antigravity"
          }
        ]
      }
    ]
  }
}
```

**Nota:** Sostituisci `com.google.antigravity` con il bundle ID della tua app IDE.

### `/notifiche smart`
Configura gli hooks CON il check dell'app in primo piano. Le notifiche arrivano solo quando l'IDE NON e in primo piano (stai facendo altro).

Hooks da scrivere in `~/.claude/settings.json` (merge con contenuto esistente):

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ] && [ \"$FRONT\" != \"Code\" ]; then /opt/homebrew/bin/terminal-notifier -message 'Ho finito di generare!' -title 'Claude Code' -sound Glass -activate com.google.antigravity; fi"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ] && [ \"$FRONT\" != \"Code\" ]; then /opt/homebrew/bin/terminal-notifier -message 'Serve la tua attenzione!' -title 'Claude Code' -sound Submarine -activate com.google.antigravity; fi"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ] && [ \"$FRONT\" != \"Code\" ]; then /opt/homebrew/bin/terminal-notifier -message 'Serve la tua autorizzazione!' -title 'Claude Code - Permesso' -sound Submarine -activate com.google.antigravity; fi"
          }
        ]
      }
    ]
  }
}
```

**Nota:** Sostituisci `com.google.antigravity` con il bundle ID della tua app IDE.

## Regole importanti

1. **Leggi SEMPRE** `~/.claude/settings.json` prima di modificarlo
2. **Merge** con il contenuto esistente — non sovrascrivere mai permissions, env, o altri hooks
3. **Valida** il JSON dopo ogni modifica con `jq -e . ~/.claude/settings.json`
4. Conferma all'utente lo stato finale dopo ogni operazione
