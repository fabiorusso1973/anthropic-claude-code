---
titolo: Allineamento inviti Outlook/Teams → calendario Google principale
origine: promemoria Google Calendar "Collegare inviti Outlook/Teams al calendario Google" (12/11/2026 09:30)
data_analisi: 2026-11-12
stato: analisi completata, configurazione da eseguire manualmente su Aruba
---

# Allineamento calendari Google / Outlook: esito dell'analisi e procedura

## 1. Sintesi

Il promemoria dava per scontato che gli inviti Teams/Outlook arrivassero su una casella **Microsoft 365** e proponeva:

1. lo sblocco dell'inoltro esterno in Defender;
2. una regola in Outlook sul web;
3. un test.

Le verifiche svolte oggi in parallelo da tre agenti, più un riscontro diretto, mostrano che **questa premessa non regge**:

| Verifica | Esito | Evidenza |
|---|---|---|
| Casella M365 `fabio.russo@consulentiassociati.cc` (tenant FABIORUSSO.onmicrosoft.com) | **Inattiva** | La Posta in arrivo contiene 6 elementi in tutto, l'ultimo del 05/02/2024. L'ultimo evento in calendario è del 19/08/2024. Negli ultimi 30 giorni non ci sono inviti e non ci sono eventi futuri. |
| Server che gestisce davvero la posta di consulentiassociati.cc | **Aruba** | Intestazioni del messaggio "Fw: Raccomandazioni" (Gmail, 01/11/2026): `Received: from smtpcmd14161.aruba.it`, `DKIM d=aruba.it`, `Message-ID …@hxcore.ol`, cioè il client Outlook mobile su una casella Aruba. |
| Record MX del dominio | **Da integrare** | Non verificabili da questa sessione: la lookup DNS è stata bloccata. |
| Connettore M365 per creare regole | Non adatto | `outlook_create_filter` dichiara: *"Forwarding, redirecting, and deleting actions are deliberately not supported"*. |
| Inoltri automatici già attivi verso Gmail | Nessuno | In Gmail risultano solo 2 inoltri manuali dell'01/11/2026, che non sono inviti. |
| Calendari Google sottoscritti (ICS) o servizi di sincronizzazione | Nessuno | Calendari presenti: principale, FINGECO, ST CONSULENTI ASSOCIATI, SALA RIUNIONI PROVALUE. |

**Conclusione.** I passi 1 e 2 del promemoria, cioè Defender e la regola OWA, **non servono**. L'inoltro va configurato **lato server Aruba** (Webmail Aruba). L'accesso alla casella Aruba non è disponibile da qui, quindi la configurazione va eseguita a mano seguendo il §3.

## 2. Vincolo di policy di gruppo (Allcore)

La mail "Utilizzo degli strumenti aziendali Google Workspace" (M. Passini, 22/10/2026), ripresa da L. Montessori il 03/11/2026, stabilisce:

- *"Google Calendar: per la pianificazione di appuntamenti e riunioni."*
- *"a partire da martedì 1° dicembre gli unici strumenti informatici autorizzati saranno quelli indicati nella mail sotto riportata."* L'unica eccezione è Excel.

Effetti sulle opzioni:

| Opzione | Coerenza con la policy |
|---|---|
| Inoltro server-side Aruba → `f.russo@soluzionetasse.com` (gli eventi entrano in Google Calendar) | **Coerente**: concentra la pianificazione in Google Calendar. |
| Servizio di sincronizzazione esterno (OneCal, CalendarBridge, Reclaim) | **Sconsigliato**: è un'app OAuth di terzi non autorizzata. Comporta inoltre i profili GDPR (DPA, trasferimento extra-UE). |
| Sottoscrizione ICS "Da URL" | Fuori obiettivo, perché crea un calendario separato. Inoltre non è applicabile ad Aruba (da verificare). |

Altri punti emersi dall'analisi e da valutare in vista del 01/12:

- vari eventi futuri usano **Zoom**, con host consulentiassociati.cc, invece di Meet;
- è attiva un'integrazione **Read.ai ↔ Teams** (mail di ri-autorizzazione del 05/11/2026).

## 3. Procedura operativa (Webmail Aruba)

> Le diciture dei menu Aruba sono ricavate dagli estratti di ricerca: le pagine complete non erano raggiungibili dalla sessione, vedi §6. Vanno confermate nell'interfaccia.

### Opzione A: regola solo per gli inviti (preferibile)

1. Accedere alla Webmail Aruba con `fabio.russo@consulentiassociati.cc`.
2. Aprire **Impostazioni → Messaggi e scrittura → Regole messaggi → Nuova regola**.
3. Condizioni disponibili nella Webmail, confermate a video il 12/11/2026: Destinatario, Oggetto, Contenuto, Nome allegato, Dimensione, Header. Si creano **due regole**, perché le condizioni dentro la stessa regola valgono tutte insieme (AND):
   - **Regola 1 "Inviti Outlook/Teams"**: **Header** con nome `x-ms-exchange-calendar-series-instance-id` che contiene `BAAAAIIA4AB0xbcQGoLgCA`.
     - *Riscontro*: l'invito Teams "Statuto RF Innova" del 15/01/2026, letto per intero, contiene questo header con un valore che inizia con `BAAAAIIA4AB0xbcQGoLgCA`.
     - *Formato del messaggio*: è `multipart/alternative`. La parte `text/calendar; method=REQUEST` è inserita nel corpo e **non ha un nome file**, e `X-MS-Has-Attach` è vuoto. Per questo un filtro su "Nome allegato" **non** intercetta gli inviti Outlook/Teams.
     - *Ripiego*: header `x-ms-traffictypediagnostic` che contiene `MeetingMessage`, presente nello stesso invito.
   - **Regola 2 "Inviti Google/altri (.ics)"**: **Nome allegato** che contiene `invite.ics`. È riscontrato sugli inviti Google Calendar presenti in Gmail.
   - L'**Oggetto** è inaffidabile: gli inviti Outlook hanno come oggetto solo il titolo della riunione, senza prefisso "Invito:".
4. Come azione di entrambe le regole impostare **Inoltra a** `f.russo@soluzionetasse.com`.
5. Verificare che resti una copia nella casella Aruba. Se la regola non lo consente, valutare l'Opzione B.
6. Salvare. Aruba invia una mail di conferma "La regola messaggi è stata impostata".

### Opzione B: inoltro automatico totale con copia

**Impostazioni → Messaggi e scrittura → Inoltro automatico → + Crea inoltro automatico** verso `f.russo@soluzionetasse.com`, con l'opzione **mantieni copia nella posta in arrivo** attiva.

- Pro: è semplice e lato server.
- Contro: porta in Workspace **tutta** la posta dello Studio (privacy e segreto professionale da valutare).

### Lato Google (account `f.russo@soluzionetasse.com`)

- Aprire Google Calendar → **Impostazioni → Generali → Impostazioni evento → "Aggiungi inviti al mio calendario"** e scegliere *Da tutti* oppure *Solo se il mittente è noto*. La funzione "Eventi da Gmail" è un'altra cosa: riguarda prenotazioni, voli e hotel.
- Se l'opzione è bloccata, la decide l'amministratore Workspace.

## 4. Test (passo 3 del promemoria)

1. Chiedere a un collega con Microsoft 365 di inviare un **invito Teams di prova** a `fabio.russo@consulentiassociati.cc`, poi una **modifica** e un **annullamento**.
2. In Gmail verificare che il messaggio inoltrato mostri la scheda invito, che l'evento compaia nel calendario principale e che il link Teams sia presente.
3. Controllare l'orario, perché c'è il rischio di sfasamento del fuso negli ICS Outlook.
4. **Rispondere all'invito dalla casella Aruba e non da Google.** La risposta da Google partirebbe da `f.russo@soluzionetasse.com`, e l'organizzatore la vedrebbe come un partecipante sconosciuto.
5. Se Google non aggiunge l'evento in automatico, perché il destinatario originale non è l'account Google, usare "Aggiungi al calendario" dalla scheda Gmail. Se anche questo non funziona, rivalutare la soluzione.

## 5. Punti aperti (da integrare)

- [ ] Record MX di consulentiassociati.cc, per confermare in modo definitivo che la posta è su Aruba.
- [ ] Condizioni disponibili nelle regole della Webmail Aruba (header sì/no) e conservazione della copia con "Inoltra a".
- [ ] Modalità di inoltro Aruba (redirect con originale intatto oppure nuovo messaggio): è decisiva perché Google riconosca l'invito.
- [ ] Contenuto dei PDF "Guida a Google Calendar" e "Guida a Google Meet" allegati alla mail del 03/11/2026.
- [ ] Impostazione "Aggiungi inviti al mio calendario" attualmente attiva nell'account Google.
- [ ] Destino della casella/tenant M365 inattivo (FABIORUSSO.onmicrosoft.com).

## 6. Fonti consultate

Le pagine ufficiali (guide.aruba.it, learn.microsoft.com, support.microsoft.com, support.google.com) **non erano raggiungibili** dalla rete della sessione. Le procedure al §3 derivano da estratti dei motori di ricerca e **non costituiscono lettura della fonte**: vanno confermate a video.

| Fonte | Letta integralmente | Uso |
|---|---|---|
| guide.aruba.it, "Regole e filtri per la posta in arrivo" | no (solo estratto) | Percorso Regole messaggi, azioni disponibili |
| guide.aruba.it, "Risposta automatica e inoltro automatico" | no (solo estratto) | Inoltro automatico con copia |
| support.google.com/calendar/answer/13159188 | no (solo estratto) | "Aggiungi inviti al mio calendario" |
| support.google.com/calendar/answer/6084018 | no (solo estratto) | Distinzione con "Eventi da Gmail" |
| Gmail: thread "Utilizzo degli strumenti aziendali Google Workspace" | **sì** | Policy di gruppo, citazioni §2 |
| Gmail: messaggio "Fw: Raccomandazioni" del 01/11/2026 (intestazioni RAW) | **sì** | Evidenza del server Aruba, §1 |
| Casella M365 e calendario Outlook (connettore Microsoft 365) | **sì** | Casella inattiva, §1 |
| Google Calendar: elenco calendari ed eventi | **sì** | §1 |
