# **Functional Testing Pipeline (test-functional.yml)**

## **Descrizione Generale**

La pipeline **Functional Test** è stata progettata per operare come barriera di qualità ("Quality Gate") sul branch di fix (`fix-vuln-numda`) e sui branch principali (`master`, `main`).

A differenza della pipeline di *Baseline*, che serve a fotografare lo stato AS-IS, questa pipeline ha un duplice obiettivo critico:
1.  **Regression Testing:** Garantire che le correzioni infrastrutturali (aggiornamento librerie, OS Alpine 3.19) e del codice (Path Traversal fix) non abbiano compromesso le funzionalità core di SpiderFoot.
2.  **Security Validation (Pass/Fail):** Verificare *attivamente* che le vulnerabilità note (SQL Injection e Hardcoded Secrets) siano state effettivamente rimosse. Se una vulnerabilità viene ancora rilevata, la pipeline **DEVE fallire**.

## **Workflow Trigger**

La pipeline viene eseguita automaticamente su:
* Push diretti sui branch `fix-vuln-numda`, `master` e `main`.
* Apertura di Pull Request verso questi branch.
* Avvio manuale (`workflow_dispatch`) per debug on-demand.

## **Dettaglio dei Job**

### **1. Job `runtest` (Application Logic & Security)**

Questo job esegue l'applicazione in un ambiente Python 3.11 pulito (replicando l'ambiente di produzione su Alpine 3.19).

#### **Step Funzionali e di Regressione:**

* **Install Dependencies:**
    * Installa le dipendenze dal file `requirements.txt` *aggiornato* (inclusa `cryptography>=42.0.0`).
    * **Obiettivo:** Verificare l'assenza di conflitti di dipendenze (Dependency Hell) introdotti dalle nuove versioni sicure.

* **Run SpiderFoot (Web Server Test):**
    * Lancia `sf.py` in background.
    * Esegue `curl --fail http://127.0.0.1:5001`.
    * **Obiettivo:** Verificare che il web server si avvii correttamente e risponda con HTTP 200 OK.

* **Verify Core Functionality (Regression Test):**
    * Interroga l'endpoint API `/scanlist`.
    * **Obiettivo:** Confermare che l'applicazione riesca a connettersi al database SQLite e a restituire un JSON valido. Questo esclude regressioni causate dall'aggiornamento dei driver o delle librerie di sistema.

* **Run SFCLI (Command Line Interface):**
    * Esegue test sui comandi della CLI.
    * **Obiettivo:** Assicurare che il refactoring del file `sfcli.py` (necessario per il fix del Path Traversal) non abbia introdotto errori di sintassi o logica bloccanti.

#### **Security Verification Steps (Validation of Fixes)**

Questi step confermano che le azioni di remediation (SSVC: **ACT**) sono state efficaci.

* **Verify SQL Injection Fix:**
    * **Logica:** Invia nuovamente il payload malevolo (`SELECT 1`) all'endpoint `/query`.
    * **Criterio di Successo:** Il test passa **solo se** il server risponde con `HTTP 403 Forbidden` (o comunque un codice di errore che indica blocco), confermando che il meccanismo di input validation è attivo. Se il server risponde con `200 OK`, il test fallisce immediatamente (Exit 1), segnalando che la patch non è efficace.

* **Security Verification - sfp_citadel (Secret Removal):**
    * **Logica:** Esegue lo script di *Dynamic Analysis* con mocking di rete sul modulo `sfp_citadel`.
    * **Differenza dalla Baseline:** Non c'è `continue-on-error`.
    * **Criterio di Successo:** Lo script deve terminare con successo (Exit Code 0). Questo avviene solo se il modulo, durante l'esecuzione, **NON** tenta di inviare la vecchia chiave API hardcodata (`3edfb560...`). Questo dimostra che il segreto è stato rimosso dal codice sorgente.

### **2. Job `docker-test` (Hardened Image Build)**

Questo job verifica la costruibilità dell'immagine Docker post-hardening.

* **Dipendenza:** Esegue solo se `runtest` ha avuto successo (`needs: runtest`).
* **Build Process:** Esegue `docker build . -t spiderfoot:funcional`.
* **Obiettivo:** Certificare che l'aggiornamento della base image a `alpine:3.19` e le nuove istruzioni di pulizia nel Dockerfile producano un'immagine valida e funzionante.
