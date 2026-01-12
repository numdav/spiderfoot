# **Functional Testing Pipeline (test-functional.yml)**

## **Descrizione Generale**

La pipeline **Functional Test** è stata progettata per operare come barriera di qualità ("Quality Gate") sul branch di fix (`fix-vuln-numda`) e sui branch principali (`master`, `main`).

A differenza della pipeline di *Baseline*, che serve a fotografare lo stato AS-IS dei file di interesse, questa pipeline ha un duplice obiettivo critico:
1.  **Regression Testing:** Garantire che le correzioni infrastrutturali (aggiornamento librerie, OS Alpine 3.19) e del codice (Path Traversal fix) non abbiano "rotto" le funzionalità core di SpiderFoot.
2.  **Security Verification (Pass/Fail):** Verificare *attivamente* che le vulnerabilità note (es. Hardcoded Secret) siano state effettivamente rimosse. Se la vulnerabilità è ancora presente, la pipeline **DEVE fallire**.


## **Workflow Trigger**

La pipeline viene eseguita automaticamente su:
* Push diretti sui branch `fix-vuln-numda`, `master` e `main`.
* Apertura di Pull Request verso questi branch.
* Avvio manuale (`workflow_dispatch`) per debug on-demand.


## **Dettaglio dei Job**

### **1. Job `runtest` (Application Logic & Security)**

Questo job esegue l'applicazione in un ambiente Python 3.11 pulito (lo stesso usato in produzione su Alpine 3.19).

#### **Step Chiave:**

* **Install Dependencies:**
    * Installa le dipendenze dal file `requirements.txt` *aggiornato* (con `cryptography>=42.0.0`).
    * *Obiettivo:* Verificare che non ci siano conflitti di dipendenze (Dependency Hell) con le nuove versioni sicure.

* **Run SpiderFoot (Web Server Test):**
    * Lancia `sf.py` in background e attende l'inizializzazione.
    * Esegue `curl --fail http://127.0.0.1:5001`.
    * *Obiettivo:* Verificare che il web server si avvii correttamente e risponda con HTTP 200 OK. L'opzione `--fail` fa fallire il job se il server risponde con errori (es. 500 o 403).

* **Run SFCLI (Command Line Interface):**
    * Esegue comandi help e debug sulla CLI.
    * *Obiettivo:* Assicurare che il refactoring del file `sfcli.py` (necessario per il fix del Path Traversal) non abbia introdotto errori di sintassi o logica.

* **Security Verification - sfp_citadel (The "Fix" Test):**
    * Esegue lo stesso script di *Dynamic Analysis* usato nella Baseline, ma con una differenza fondamentale: **l'assenza di `continue-on-error`**.
    * **Logica:** Lo script inietta un mock di rete ed esegue il modulo `sfp_citadel`. Se il modulo tenta di inviare la vecchia chiave API hardcodata (`3edfb560...`), lo script esce con `sys.exit(1)`.
    * *Risultato Atteso:* La pipeline deve terminare con successo (Exit Code 0), confermando che il modulo **NON** ha inviato il segreto e ha gestito correttamente la mancanza di configurazione.

### **2. Job `docker-test` (Container Build)**

Questo job verifica la costruibilità dell'immagine Docker *dopo* l'hardening.
* **Dipendenza:** Richiede il successo del job `runtest` ( clausola `needs: runtest`).
* **Build Process:** Esegue `docker build . -t spiderfoot:funcional`.
* **Obiettivo:** Confermare che il passaggio da `alpine:3.13` a `alpine:3.19` e l'aggiunta dei comandi di pulizia (`rm -rf /lib/apk/db`) non abbiano introdotto errori di build (es. pacchetti di sistema mancanti o rinominati).
