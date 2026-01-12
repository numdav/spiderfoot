# **Baseline Testing Pipeline (test-baseline.yml)**

## **Descrizione Generale**

La pipeline **Baseline** è configurata per agire come "Gruppo di Controllo" per il progetto di messa in sicurezza. Il suo scopo primario è eseguire test di integrazione e build sul codice originale (Legacy) presente nel branch master, indipendentemente dal branch su cui viene lanciata l'azione.

Questo permette di verificare lo stato di funzionamento dell'applicazione "AS-IS" prima dell'applicazione delle patch di sicurezza.

## **Struttura dei Job**

### **1\. Job runtest WEB / CLI / Module**

Questo job verifica l'eseguibilità dell'applicazione in un ambiente Python 3.11 nativo.

* **Setup:** Configura l'ambiente e installa le dipendenze definite nel requirements.txt originale.  
* **Web Server Test:**  
  * Avvia sf.py in background.  
  * Effettua una chiamata curl alla porta 5001 per verificare la risposta HTTP.  
  * Il test emette un messaggio di successo o warning basandosi sulla risposta.  
* **CLI Verification:**  
  * Esegue sfcli.py \--help per confermare che il modulo a riga di comando sia interpretabile e privo di errori di sintassi bloccanti.  
  * Tenta una connessione al server tramite sfcli.py \-d.
* **Module Verification - sfp_citadel (Vulnerability Confirmation):**
  * Questo step esegue uno script Python dinamico (`test_citadel_vuln.py`) creato a runtime.
  * **Obiettivo:** Simulare l'esecuzione del modulo `sfp_citadel.py` intercettando le chiamate di rete (mocking).
  * **Logica del Test:** Se il modulo tenta di inviare la chiave API hardcodata nota (`3edfb560...`), il test intercetta la richiesta e segnala la vulnerabilità confermata.
  * **Comportamento Pipeline:** Lo step è configurato con `continue-on-error: true` perché, nel contesto di baseline, ci aspettiamo che la vulnerabilità sia presente (e quindi che lo script esca con errore `sys.exit(1)`), confermando la necessità del fix.

### **2\. Job docker-test (Container Build)**

Questo job verifica la costruibilità dell'immagine Docker definita nel Dockerfile originale.

* **Build Process:** Esegue il comando docker build . \-t spiderfoot:baseline.  
* **Esito:** La build viene completata con successo, confermando che l'immagine legacy (basata su Alpine 3.13) è ancora tecnicamente costruibile nell'ambiente GitHub Actions attuale.
