# **SE4CS - SpiderFoot - DevSecOps Pipeline Documentation**

Questo documento descrive l'architettura, gli strumenti, le scelte progettuali e le configurazioni della pipeline CI/CD implementata per il progetto SpiderFoot.  
La pipeline è stata progettata per integrare i controlli di sicurezza automatizzati in Continuous Integration  (DevSecOps) su codice, dipendenze e infrastruttura del repository, tramite GitHub Actions.

## **Architettura del Flusso**

La pipeline (se4cs.yml) segue un approccio **Fail-Safe** con esecuzione parallela per massimizzare la velocità di feedback.

1. **Check Secrets:** Verifica preliminare della presenza dei token necessari. Se un segreto manca, il job corrispondente viene saltato senza far fallire la pipeline.  
2. **Parallel Security Scans:** Esecuzione simultanea di 4 motori di analisi (GitGuardian, Snyk, Semgrep, SonarCloud).  
3. **Push on Dashboard web:** push dei report JSON nelle relative Dashboard Web dei tool di analisi.  
4. **Aggregation:** Raccolta normalizzata dei risultati in una dashboard unificata, generata a runtime.

## **Stack Tecnologico e Strumenti**

| Strumento | Tipo Analisi | Target | Obiettivo |
| :---- | :---- | :---- | :---- |
| **GitGuardian** | Secrets Detection | Git History | Rilevare credenziali hardcodate (API Key, Password) nella history del repo. |
| **Snyk Open Source** | SCA (Manifest) | requirements.txt | Identificare vulnerabilità note nelle dipendenze Python. |
| **Snyk Code** | SAST | Codice Sorgente | Rilevare pattern di codice insicuro (es. Path Traversal, XSS). |
| **Snyk Container** | SCA (Binary) | Docker Image (Build) | Analisi vulnerabilità OS (Alpine Linux) sull'immagine *buildata* a runtime. |
| **Semgrep** | SAST | Codice & Config | Analisi semantica veloce e controllo configurazioni (es. Docker Compose). |
| **SonarCloud** | Quality & SAST | Code \+ Coverage | Analisi Code Smells, Coverage dei test e Quality Gate.. |

## **Dettagli Implementativi e Ottimizzazioni**

### **1\. Least Privilege Permission**

I permessi del GITHUB\_TOKEN sono stati ristretti al minimo indispensabile (contents: read) e definiti a livello di singolo Job per massimizzare la sicurezza della pipeline stessa.

### **2\. Gestione Resiliente dei Test ("Zombie Processes")**

L'integrazione con SonarCloud richiede la generazione di un report di copertura (coverage.xml). Tuttavia, i test di integrazione di SpiderFoot tendono a bloccarsi (hang) indefinitamente a causa di chiamate di rete esterne fallite (es. errori SSL), creando processi "zombie" che bloccano la CI.

Soluzione Implementata:  
È stato adottato un approccio "Circuit Breaker" a livello di shell bash:

* **Unit Testing Selettivo:** Esclusione esplicita dei path problematici (--ignore=test/integration e \--ignore=test/unit/test\_spiderfoot.py);  
* **Timeout Forzato:** Utilizzo del comando timeout \-k 10s 2m per forzare la terminazione di eventuali processi o thread rimasti appesi, garantendo che la pipeline prosegua verso l'analisi di SonarCloud;  
* **Verifica Esito Test:** Lo step non fallisce in base all'exit code di pytest, ma verifica l'esistenza del file di output:  
```
  if \[ \-s coverage.xml \]; then exit 0; else exit 1; fi
```

### **3\. Global Dependency Caching (Green IT)**

Per ottimizzare i tempi di esecuzione e ridurre l'impatto ambientale (CPU/Network), è stato abilitato il caching nativo di pip in tutti i job Python:

``` uses: actions/setup-python@v5  
  with:  
    python-version: '3.11'  
    cache: 'pip'
```
L’utilizzo della cache ha rilevato nei test la diminuzione dei tempi di esecuzione della pipeline di circa il 40% complessivamente.

### **4\. Container Security: Statico vs Dinamico**

A differenza dell'analisi statica del Dockerfile (tipica delle integrazioni SCM), la pipeline esegue una **build reale** dell'immagine Docker prima della scansione.

* **Comando:** docker build \-t spiderfoot-target . seguito da snyk container test.  
* **Vantaggio:** Permette di rilevare vulnerabilità nei pacchetti di sistema installati dinamicamente (es. tramite apk add) che non sono visibili nel solo file di testo del Dockerfile.

### **5\. Decoupling Temporale (SonarCloud)**

Per garantire che i dati siano disponibili via API per il report JSON finale, è stato inserito uno step di attesa (sleep 20\) post-scansione. Questo permette al motore asincrono di SonarCloud di elaborare il report caricato prima che la pipeline tenti di scaricare i risultati.

## **Aggregazione e Reporting**

Il job finale aggregate-reports utilizza uno script Python personalizzato (merge\_reports.py) iniettato dinamicamente (per praticità di trasporto del file se4cs.ymltra altri repository di test).

**Funzionalità dello script:**

* **Normalizzazione:** Gestisce sia il formato standard JSON di Snyk (per le dipendenze) sia il formato **SARIF** (per Snyk Code);  
* **Consolidamento:** Somma i risultati di Snyk Deps, Code e Container in un unico punteggio nella dashboard;  
* **Output:** Genera una tabella Markdown visibile nel **GitHub Job Summary** per una valutazione immediata dello stato di sicurezza (CLEAN / REVIEW / FAIL).

### **Artefatti Generati**

La pipeline rende disponibili per il download i seguenti report raw:

1. gitguardian-report.zip (Contiene 1 JSON: *ggshield*)  
2. snyk-report.zip (Contiene 3 JSON: *deps*, *code*, *container*)  
3. semgrep-report.zip (Contiene 1 JSON: *semgrep*)  
4. sonarcloud-report.zip (Contiene 1 JSON: *sonar-issues*)
