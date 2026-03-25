---
name: notifiche
description: Gestisci le notifiche macOS di Claude Code. Attiva/disattiva le notifiche sonore quando Claude finisce di generare o ha bisogno di attenzione. Comandi - /notifiche on, /notifiche off, /notifiche sempre (anche se stai guardando), /notifiche smart (solo quando non guardi), /notifiche status.
---

# Notifiche macOS per Claude Code

Gestisci le notifiche sonore native di macOS che avvisano quando Claude finisce di generare o ha bisogno della tua attenzione.

## Comandi

### `/notifiche status`
Leggi `~/.claude/settings.json` e controlla se i hooks `Stop` e `Notification` esistono e sono attivi. Mostra lo stato attuale:
- Se gli hooks esistono e contengono il check "frontmost" -> modalita "smart" (solo quando non guardi)
- Se gli hooks esistono senza il check -> modalita "sempre"
- Se gli hooks non esistono -> notifiche disattivate

### `/notifiche on`
Alias di `/notifiche smart`. Attiva le notifiche in modalita smart (default).

### `/notifiche off`
Rimuovi i hooks `Stop` e `Notification` da `~/.claude/settings.json`. Preserva tutto il resto del file.

Leggi il file, rimuovi le chiavi `Stop` e `Notification` dall'oggetto `hooks`. Se `hooks` rimane vuoto, rimuovi anche la chiave `hooks`.

### `/notifiche sempre`
Configura gli hooks senza il check dell'app in primo piano. Le notifiche arrivano SEMPRE, anche se stai guardando VSCode.

Hooks da scrivere in `~/.claude/settings.json` (merge con contenuto esistente):

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Ho finito di generare!\" with title \"Claude Code\" sound name \"Glass\"'"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "osascript -e 'display notification \"Serve la tua attenzione!\" with title \"Claude Code\" sound name \"Submarine\"'"
          }
        ]
      }
    ]
  }
}
```

### `/notifiche smart`
Configura gli hooks CON il check dell'app in primo piano. Le notifiche arrivano solo quando VSCode/Electron NON e in primo piano (stai facendo altro).

Hooks da scrivere in `~/.claude/settings.json` (merge con contenuto esistente):

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ] && [ \"$FRONT\" != \"Code\" ]; then osascript -e 'display notification \"Ho finito di generare!\" with title \"Claude Code\" sound name \"Glass\"'; fi"
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "FRONT=$(osascript -e 'tell application \"System Events\" to get name of first application process whose frontmost is true'); if [ \"$FRONT\" != \"Electron\" ] && [ \"$FRONT\" != \"Code\" ]; then osascript -e 'display notification \"Serve la tua attenzione!\" with title \"Claude Code\" sound name \"Submarine\"'; fi"
          }
        ]
      }
    ]
  }
}
```

## Regole importanti

1. **Leggi SEMPRE** `~/.claude/settings.json` prima di modificarlo
2. **Merge** con il contenuto esistente — non sovrascrivere mai permissions, env, o altri hooks
3. **Valida** il JSON dopo ogni modifica con `jq -e . ~/.claude/settings.json`
4. Conferma all'utente lo stato finale dopo ogni operazione
