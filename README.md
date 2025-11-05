# mysql-backup – Backup automatico MySQL su S3

Il container **`mysql-backup`** gestisce il backup automatico del database MySQL.
Si basa sull’immagine `mysql/backup` (una versione personalizzata costruita da `./mysql-backup/Dockerfile`) e consente di eseguire dump periodici del database, inviandoli su un bucket **Amazon S3**.

---

## 🧩 Struttura generale

Il container è definito come servizio nel `docker-compose.yml`:

```yaml
sms-mysql-backup:
  image: mysql/backup
  container_name: mysql-backup
  hostname: mysql-backup
  restart: always
  build:
    context: ./mysql-backup
    dockerfile: Dockerfile
  environment:
    MYSQL_HOST: mysql-db
    MYSQL_USER: root
    MYSQL_DATABASE: root
    MYSQL_PWD: password
    AWS_S3_URI: s3-uri
    AWS_ACCESS_KEY_ID: s3-key-id
    AWS_SECRET_ACCESS_KEY: s3-key-secret
    AWS_DEFAULT_REGION: s3-region
    AWS_OTHER_OPTIONS: "--storage-class DEEP_ARCHIVE --progress-frequency 60"
  volumes:
    - ./mysql-backup/logs:/var/log
```

---

## ⚙️ Variabili d’ambiente

Le variabili d’ambiente controllano la connessione al database e la configurazione del backup remoto.

| Variabile                 | Descrizione                                                                               | Esempio                                                  |
| ------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **MYSQL_HOST**            | Nome host o container del database MySQL da cui eseguire il backup.                       | `mysql-db`                                               |
| **MYSQL_USER**            | Utente MySQL con privilegi di lettura sul database.                                       | `root`                                                   |
| **MYSQL_PWD**             | Password dell’utente MySQL.                                                               | `password`                                               |
| **MYSQL_DATABASE**        | Nome del database da esportare.                                                           | `password`                                               |
| **AWS_S3_URI**            | URI del bucket S3 dove salvare i backup.                                                  | `s3://my-bucket/mysql-backups/`                          |
| **AWS_ACCESS_KEY_ID**     | Access key AWS con permessi di scrittura sul bucket.                                      | `AKIA...`                                                |
| **AWS_SECRET_ACCESS_KEY** | Secret key AWS associata all’access key.                                                  | `wJalrXUtn...`                                           |
| **AWS_DEFAULT_REGION**    | Regione AWS in cui risiede il bucket.                                                     | `eu-central-1`                                           |
| **AWS_OTHER_OPTIONS**     | Parametri aggiuntivi passati al client S3 per personalizzare il comportamento del backup. | `"--storage-class DEEP_ARCHIVE --progress-frequency 60"` |

> ⚠️ Le virgolette non vanno incluse nel valore effettivo della variabile.
> Ogni opzione deve essere separata da uno spazio, come mostrato nell’esempio.

---

## 💾 Volumi

| Percorso nel container | Descrizione                                                  |
| ---------------------- | ------------------------------------------------------------ |
| `/var/log`             | Directory dove vengono salvati i log dei processi di backup. |

Esempio di montaggio:

```yaml
volumes:
  - ./mysql-backup/logs:/var/log
```

---

## 🕒 Configurazione del cron

Il container utilizza **cron** per eseguire automaticamente i backup in orari programmati.
Il file che definisce la pianificazione si trova in:

```
mysql-backup/crontab.txt
```

### Contenuto predefinito

```bash
SHELL=/bin/bash
BASH_ENV=/root/.bashrc
* 1 * * * root /app/backup.sh
```

Questa configurazione esegue il backup **ogni giorno all’1:00 di notte** (ora del container).
Lo script eseguito è `/app/backup.sh`, che si occupa del dump del database e del caricamento su S3.

### Come personalizzare la pianificazione

Per modificare la frequenza dei backup, basta **editare manualmente il file `crontab.txt`** prima di ricostruire o riavviare il container.

Esempi:

| Obiettivo             | Esempio di riga cron               | Descrizione                                     |
| --------------------- | ---------------------------------- | ----------------------------------------------- |
| Ogni giorno alle 3:00 | `0 3 * * * root /app/backup.sh`    | Esegue il backup alle 03:00 di ogni giorno      |
| Ogni 6 ore            | `0 */6 * * * root /app/backup.sh`  | Esegue il backup ogni 6 ore                     |
| Ogni lunedì alle 2:30 | `30 2 * * 1 root /app/backup.sh`   | Backup settimanale il lunedì alle 02:30         |
| Ogni 15 minuti        | `*/15 * * * * root /app/backup.sh` | Backup frequente ogni 15 minuti (solo per test) |

> 🔧 Dopo aver modificato `mysql-backup/crontab.txt`, è necessario **ricostruire il container** per applicare le modifiche:
>
> ```bash
> docker compose up --build -d sms-mysql-backup
> ```

---


## ▶️ Costruzione ed esecuzione dell’immagine

Per buildare e avviare il servizio di backup insieme al database MySQL:

```bash
docker compose up --build -d sms-mysql-backup
```

Il container inizierà automaticamente il processo di backup secondo la pianificazione definita in `crontab.txt`.

---

## 🧾 Log e troubleshooting

I log del processo di backup vengono salvati in:

```
./mysql-backup/logs/backup.log
```

Per visualizzarli in tempo reale:

```bash
docker logs -f mysql-backup
```

Se il backup non viene caricato su S3, verificare:

* ( eventuali errori nei logs )
* che le credenziali AWS siano corrette;
* che il bucket e il percorso esistano e siano accessibili;
* che `MYSQL_HOST` punti al container MySQL corretto;
* che la pianificazione in `crontab.txt` sia corretta e valida per cron.

---

## 📦 Note finali

* Il container `mysql-backup` è pensato per operazioni periodiche o manuali di dump su S3.
* È possibile personalizzare la frequenza dei backup tramite `mysql-backup/crontab.txt`.
* Può essere adattato facilmente per altri servizi compatibili con l’API S3 (MinIO, Wasabi, Backblaze B2, ecc.).
