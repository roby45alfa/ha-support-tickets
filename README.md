# Segnalazioni clienti per Home Assistant

Un pulsante nell'Home Assistant dei tuoi clienti per inviarti **bug** e **migliorie** della loro domotica.
Le segnalazioni arrivano su **Telegram** e finiscono in una to-do list **Vikunja** sul tuo server;
il cliente vede in dashboard lo stato di ogni richiesta (*Inviata → In lavorazione → Risolta*).
Se il tuo server è spento, la richiesta resta in coda sull'HA del cliente e parte da sola appena torna online.

```
 HA del cliente                         Il tuo server
┌────────────────────────┐   HTTPS    ┌──────────────────────────────┐
│ Pulsante → modulo      │──────────► │ n8n  /webhook/segnalazioni   │──► Telegram
│ (script con campi)     │  token     │       │                      │
│                        │            │       ▼                      │
│ Coda locale (to-do)    │            │ Vikunja (to-do, 1 progetto)  │
│ se il server è spento  │            │       ▲                      │
│                        │◄────────── │ n8n  /webhook/segnalazioni-  │
│ Pagina "Le mie         │  ogni 10'  │       stato                  │
│ richieste"             │            └──────────────────────────────┘
└────────────────────────┘
```

## Contenuto

| Percorso | Cosa contiene |
|---|---|
| `server/vikunja/docker-compose.yml` | Vikunja (to-do list self-hosted) |
| `server/n8n/segnalazioni-clienti.json` | Workflow n8n: ricezione, Telegram, Vikunja, stato |
| `homeassistant/packages/segnalazioni.yaml` | Tutto il lato cliente in un package |
| `homeassistant/secrets.example.yaml` | Valori da mettere nel `secrets.yaml` del cliente |
| `homeassistant/dashboard/` | Card pronte: nativa, pop-up Bubble Card, tablet Mushroom |
| `assets/avatar-bot.png` | Immagine per il bot Telegram |

## Requisiti

- Un server con **Docker** e **n8n** (già installato).
- n8n raggiungibile da internet in HTTPS (Cloudflare Tunnel, reverse proxy, …) — basta esporre il percorso `/webhook/`.
- Un **bot Telegram** dedicato.
- Lato cliente: **Home Assistant 2024.10** o successivo.
- Facoltativi per le card: Bubble Card, Mushroom, card-mod, layout-card (HACS).

---

## Installazione

### 1. Vikunja

```bash
mkdir -p ~/vikunja && cd ~/vikunja
# copia qui server/vikunja/docker-compose.yml
mkdir files db && sudo chown 1000 files db
```

Nel compose sostituisci `IP-SERVER` e `VIKUNJA_SERVICE_JWTSECRET` (`openssl rand -hex 32`), poi:

```bash
docker compose up -d
docker compose exec vikunja /app/vikunja/vikunja user create -u admin -e tua@email.it
```

Da `http://IP-SERVER:3456`:

1. crea un progetto (es. **Segnalazioni clienti**): il numero nell'URL `/projects/2/…` è l'**ID progetto**;
2. crea l'etichetta **`In lavorazione`** (esattamente così);
3. **Impostazioni → Token API**: preset **TASK MANAGEMENT** (o almeno *Tasks → Create, Read, Read All*), scadenza lunga. Copia il token.

### 2. Bot Telegram

1. **@BotFather** → `/newbot` → copia il token.
2. Apri il bot e premi **Avvia** (altrimenti non può scriverti).
3. Il tuo chat ID: scrivi a **@userinfobot**.
4. Facoltativo: `/setuserpic` con `assets/avatar-bot.png` (inviala come *foto*).

### 3. n8n

1. **Workflows → Import from file** → `server/n8n/segnalazioni-clienti.json`.
2. Credenziali:
   - **Header Auth** "Vikunja": Name `Authorization`, Value `Bearer TOKEN-VIKUNJA` (in modalità **Fixed**). Assegnala ai nodi *Crea task Vikunja* e *Leggi task Vikunja*.
   - **Telegram API** con il token del bot → nodo *Notifica Telegram*.
3. Sostituisci i segnaposto:

| Nodo | Cosa |
|---|---|
| Verifica cliente | un token per cliente (`openssl rand -hex 24`) e il suo nome |
| Crea task Vikunja | `IP-SERVER` e `ID-PROGETTO` nell'URL |
| Leggi task Vikunja | `IP-SERVER` nell'URL |
| Formatta stato | `const PROGETTO = 2;` (numero, senza virgolette) |
| Notifica Telegram | `TUO-CHAT-ID` e `IP-SERVER` nel testo |

4. **Salva** e **attiva / pubblica** il workflow.

Prova dalla LAN:

```bash
curl -X POST http://IP-SERVER:5678/webhook/segnalazioni \
  -H "X-Segnalazioni-Token: TOKEN-CLIENTE-1" -H "Content-Type: application/json" \
  -d '{"tipo":"Bug","stanza":"Test","descrizione":"Prova","inviata_il":"01/01/2026 12:00","in_coda":false}'
# → {"ok":true}, task in Vikunja, messaggio Telegram

curl http://IP-SERVER:5678/webhook/segnalazioni-stato -H "X-Segnalazioni-Token: TOKEN-CLIENTE-1"
# → JSON con la richiesta "Prova"
```

Poi elimina il task di prova in Vikunja.

### 4. Esposizione su internet

Pubblica n8n in HTTPS ed esponi **solo** `/webhook/`. Esempi:

- **Cloudflare Zero Trust**: un'applicazione Access su tutto il dominio di n8n e una seconda su `tuo-dominio/webhook/` con policy *Bypass → Everyone*.
- **Nginx Proxy Manager**, scheda *Advanced* del proxy host:
  ```nginx
  location ~ ^/(?!webhook/) { return 403; }
  ```

L'editor di n8n resta così accessibile solo da LAN/VPN (o dietro login Cloudflare).

### 5. Home Assistant del cliente

1. **Impostazioni → Dispositivi e servizi → Aggiungi integrazione → Local To-do**, nome **`Segnalazioni in attesa`**.
2. Aggiungi a `secrets.yaml` i valori di `homeassistant/secrets.example.yaml` (URL pubblici e token del cliente).
3. Attiva i package in `configuration.yaml` (se non l'hai già fatto):
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
4. Copia `homeassistant/packages/segnalazioni.yaml` in `/config/packages/`.
5. **Strumenti per sviluppatori → YAML → Verifica configurazione**, poi **Riavvia**.
6. Controlla in **Stati**: `sensor.segnalazioni_assistenza` = `0` e `binary_sensor.server_assistenza_online` = `on`.

> Lo script e l'automazione definiti nel package non sono modificabili dalla UI. Se preferisci gestirli dalla UI, copiali in *Script* e *Automazioni* e toglili dal package.

### 6. Dashboard

Scegli una delle card in `homeassistant/dashboard/`:

| File | Risultato | Plugin |
|---|---|---|
| `1-card-semplice.yaml` | pulsante + elenco richieste | nessuno |
| `2-popup-bubble-card.yaml` + `2-pulsante-tile.yaml` | pulsantino che apre un pop-up | Bubble Card |
| `3-tablet-mushroom-card.yaml` + `3-tablet-pulsante-barra.yaml` | pagina tablet + pulsante con badge nella barra | Mushroom, card-mod, layout-card |

---

## Uso quotidiano

In Vikunja, per ogni richiesta:

| Cosa fai | Cosa vede il cliente |
|---|---|
| niente | 📩 Inviata |
| aggiungi l'etichetta `In lavorazione` | 🔧 In lavorazione |
| segni il task come completato | ✅ Risolta |

La pagina del cliente si aggiorna ogni 10 minuti, subito dopo ogni invio e dopo lo svuotamento della coda.
Per forzarla: `homeassistant.update_entity` su `sensor.segnalazioni_assistenza`.

## Aggiungere un cliente

1. Genera un token: `openssl rand -hex 24`.
2. Nel nodo **Verifica cliente** di n8n aggiungi una riga `'TOKEN': 'Nome cliente',` e salva.
3. Sull'HA del cliente ripeti il punto 5 dell'installazione con il suo token.

Ogni cliente vede solo le proprie richieste: n8n ricava il nome dal token, non da un parametro modificabile.

## Server con spegnimento notturno

Il package presume che il server sia spento **dalle 03:00 alle 09:00**:

- l'automazione non riprova in quella fascia e svuota la coda alle **09:05**;
- la card mostra un avviso dedicato 🌙.

Se il tuo server è sempre acceso, togli la condizione `time` dall'automazione e il trigger delle 09:05; se ha orari diversi, cambiali nel package e nella card (`3 <= now().hour < 9`).

## Problemi noti

| Sintomo | Causa / soluzione |
|---|---|
| `webhook "POST segnalazioni" is not registered` | Workflow n8n non attivo/pubblicato. |
| `Bad request – Invalid model provided` sul nodo *Leggi task Vikunja* | Nelle versioni recenti di Vikunja `/tasks/all` è stato rimosso: usa `/api/v1/tasks?per_page=500` (già così nel workflow). |
| 401/403 dai nodi Vikunja | Token senza permessi: rifallo con il preset TASK MANAGEMENT. |
| `{"ok":false,"errore":"token non valido"}` | Token dell'HA diverso da quello nel nodo *Verifica cliente*. |
| La card non cambia quando metti "In lavorazione" | Il trigger del sensore template non deve avere `to`/`not_to` (altrimenti ignora i cambi di soli attributi). Il package è già corretto. |
| Le richieste completate spariscono invece di andare in *Risolte* | Se la tua versione di Vikunja esclude i task completati, nell'URL di *Leggi task Vikunja* aggiungi `&filter=done = true \|\| done = false`. |
| Il pulsante Mushroom esce dalla barra | In un `horizontal-stack` tutti i pulsanti devono avere la stessa larghezza. |

## Licenza

[MIT](LICENSE)
