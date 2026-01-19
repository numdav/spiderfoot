# **Baseline Testing Pipeline (test-baseline.yml)**

## **Descrizione Generale**

La pipeline **Baseline** è configurata per agire come "Gruppo di Controllo" per il progetto di messa in sicurezza. Il suo scopo primario è eseguire test di integrazione e build sul codice originale (Legacy) presente nel branch master, indipendentemente dal branch su cui viene lanciata l'azione.

Questo permette di verificare lo stato di funzionamento dell'applicazione "AS-IS" e confermare la presenza delle vulnerabilità note prima dell'applicazione delle patch.

## **Struttura dei Job**

### **1. Job runtest WEB / CLI / Module**

Questo job verifica l'eseguibilità dell'applicazione in un ambiente Python 3.11 nativo e la presenza di falle di sicurezza attive.

* **Setup:** Configura l'ambiente e installa le dipendenze definite nel `requirements.txt` originale.
* **Web Server Test:**
  * Avvia `sf.py` in background.
  * Effettua una chiamata `curl` alla porta `5001` per verificare la risposta HTTP e confermare che il servizio si avvii correttamente.
* **CLI Verification:**
  * Esegue `sfcli.py --help` per confermare che il modulo a riga di comando sia interpretabile e privo di errori di sintassi bloccanti.
  * Tenta una connessione al server tramite `sfcli.py -d`.

#### **Security Verification Steps (Vulnerability Confirmation)**

Questi step sono progettati per **fallire** se l'applicazione fosse sicura. Il loro successo conferma che la vulnerabilità è presente ed sfruttabile.

* **SQL Injection Probe:**
  * **Obiettivo:** Verificare la suscettibilità dell'endpoint `/query` a iniezioni SQL arbitrarie.
  * **Logica:** Invia una richiesta HTTP GET con payload SQL grezzo (`SELECT 1`).
  * **Criterio di Successo (Vulnerabilità Confermata):** Se il server risponde con `HTTP 200` e un corpo JSON valido, significa che la query è stata eseguita direttamente sul database senza validazione.

* **Module Verification - sfp_citadel:**
  * **Obiettivo:** Confermare la presenza di Hardcoded Secrets (API Key) nel codice sorgente.
  * **Logica:** Esegue uno script Python dinamico (`test_citadel_vuln.py`) che simula l'esecuzione del modulo `sfp_citadel.py`, intercettando le chiamate di rete (mocking).
  * **Criterio di Successo (Vulnerabilità Confermata):** Se il modulo tenta di inviare la chiave API hardcodata nota (`3edfb560...`), il test intercetta la richiesta e segnala la vulnerabilità confermata.
  * **Nota:** Configurato con `continue-on-error: true` per permettere alla pipeline di proseguire anche se lo script esce con codice di errore (comportamento atteso quando trova la key).

### **2. Job docker-test (Container Build)**

Questo job verifica la costruibilità dell'immagine Docker definita nel Dockerfile originale.

* **Build Process:** Esegue il comando `docker build . -t spiderfoot:baseline`.
* **Esito:** La build viene completata con successo, confermando che l'immagine legacy (basata su Alpine 3.13) è ancora tecnicamente costruibile nell'ambiente GitHub Actions attuale, pur contenendo vulnerabilità sistemiche note.
