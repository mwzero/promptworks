# Prompt
Sei un ETL Config Portal Generator. Genera un’app web per gestire i file yaml indicati sotto con moduli CRUD e validazioni:

- registry/datasets/<dataset>.fields.yaml: Registry dei campi canonici del dataset.
- registry/target_mapping/<datastet>.yaml: Mapping dei campi logici → colonne fisiche
- selects/sel_<dataset>.yaml: Definizione SELECT  per estrazione dataset dalla sorgente.
- transforms/sel_<dataset>.transform.yaml: Trasformazioni/alias su output della SELECT.
- jobs/job_<dataset>.yaml: Orchestrazione job estrazione → trasformazione → CSV/copy.
- ui/datasets/<dataset>.fields.yaml: Campi selezionabili nella UI (etichette, tipi, default).
- ui/filters/<dataset>.filters.yaml: Definizione dei filtri disponibili in UI.

Il frontend deve essere realizzato con React mentre il backend deve essere realizzato con API all'interno dello stesso frontend react.
Il database da utilizzare è il localStorage del browsere. 

La UI deve validare la coerenza tra questi file, permettere l’esecuzione di query Oracle e Snowflake in sola lettura, simulare le trasformazioni e mostrare anteprime CSV.
Ogni salvataggio va registrato in SQLite.

Backend e frontend sarano distribuiti insieme. 

# Requisiti funzionali
- UI per ogni file: editor tabellare o form-based.
- Validazioni cross-file (es.: alias SELECT ⊆ Registry, trasformazioni coerenti, mapping compatibile, filtri solo su campi esistenti).
- Anteprima query Oracle (read-only, limit 100 righe, parametri bindati).
- Simulatore Transform (eval su sample rows).
- Anteprima CSV generato da Job.
- Audit log per tutte le modifiche.
- Ruoli: admin, analyst, viewer (RBAC).
- CRUD per Dataset, Fields, Mapping, Select, Transform, Job, UI Fields, UI Filters.

# Output atteso
- Struttura cartelle progetto (frontend + backend + db).
- Schema DB (tabelle per i 7 file + audit log).
- Boilerplate React per le schermate principali (Registry, Mapping, Select, Transform, Job, UI Fields, UI Filters, Validation Dashboard).
- Boilerplate Express API con endpoint vuoti.
- Esempi di validazione cross-file.

# Esempio di files ipotizzando "customers" come dataset

## registry/datasets/<dataset>.fields.yaml
dataset: customers
version: 1.0.0
fields:
  - key: customer_id
    label: ID Cliente
    type: number
    nullable: false
    pii: none
  - key: birth_date
    label: Data di nascita
    type: date
    nullable: true
    pii: restricted

## registry/target_mapping/<dataset>.yaml
table: "DWH.CRM.CUSTOMERS"
columns:
  customer_id: { target: CUSTOMER_ID, type: "NUMBER(38,0)", nullable: false }
  birth_date:  { target: BIRTH_DATE, type: "DATE" }

## selects/sel_<dataset>.yaml
id: sel_customers
dataset: customers
dialect: oracle
sql: |
  SELECT c.CUSTOMER_ID AS customer_id,
         TO_DATE(c.BIRTH_DT,'YYYY-MM-DD') AS birth_date
  FROM CRM.CUSTOMERS c
  WHERE c.UPDATED_AT >= :from_ts
params:
  - { name: from_ts, type: timestamp }

## transforms/sel_<dataset>.transform.yaml
transform:
  mode: post-select
  fields:
    - { name: customer_id, expr: "customer_id", type: "NUMBER(38,0)" }
    - { name: age_years, expr: "TRUNC(MONTHS_BETWEEN(CURRENT_DATE(), birth_date)/12)", type: "NUMBER(3,0)" }

## jobs/job_<dataset>.yaml
id: job_customers_v1
select: selects/sel_customers.yaml
transform: transforms/sel_customers.transform.yaml
csv:
  header: true
  delimiter: ","
  encoding: UTF8
  compression: GZIP
staging:
  stage_name: '@data_stage'
  path: 'landing/customers/'
  filename_pattern: 'customers_{YYYYMMDD}.csv.gz'

## ui/datasets/<dataset>.fields.yaml
dataset: customers
version: 1.0.0
fields:
  - { key: customer_id, label: ID Cliente, type: number, sortable: true, defaultSelected: true }
  - { key: birth_date, label: Data di nascita, type: date, sortable: true }

## ui/filters/<dataset>.filters.yaml
dataset: customers
filters:
  - field: birth_date
    label: Data di nascita
    operators: [between, after, before]
    widget: daterange

# Stack Tecnologico
Lo stack è:
	•	Frontend: React + TypeScript (con Vite).
	•	Backend: Express.js + TypeScript.
	•	Database: SQLite (file data/portal.db, distribuito insieme al portale, senza installazione esterna).
	•	Packaging: Backend e frontend sono distribuiti insieme
