# Lab 07 - Costruire e distribuire un'app di ticket con Docker e Kubernetes

## Il prodotto da realizzare

Il sito del lab 06 diventa un'applicazione: **Officina Ticket**. Dal browser si aprono richieste di manutenzione e si segnano come risolte. Le richieste vengono salvate in SQLite.

Costruirai l'immagine con un Dockerfile, la avvierai su EC2, proverai aggiornamento e persistenza, poi distribuirai la stessa applicazione su un vero Kubernetes a nodo singolo. Infine provocherai e riparerai due guasti di rete/disponibilità.

**Consegna:** app dimostrata dal vivo, tre file sorgente e un manifesto YAML. Bastano quattro screenshot prima/dopo e una breve checklist del cleanup. Non è richiesta una relazione Markdown. Il codice fornito è materiale di partenza: il lavoro consiste nel modificarlo, costruirlo, distribuirlo e verificarlo.

## Organizzazione - 170 minuti

Nelle 4 ore: 50 minuti di spiegazioni, 10 di pausa, 170 di pratica e 10 di discussione.

| Attività                               | Minuti | Risultato verificabile                   |
| -------------------------------------- | -----: | ---------------------------------------- |
| 1. Preparare la macchina               |     40 | Docker sulla propria EC2                 |
| 2. Costruire e usare l'app             |     25 | Ticket creati e risolti dal browser      |
| 3. Versioni e rete dei container       |     20 | V2 eseguita e client interno funzionante |
| 4. Conservare e recuperare i dati      |     15 | Ticket presenti dopo la sostituzione     |
| 5. Distribuire su Kubernetes           |     30 | App su NodePort, ConfigMap e PVC         |
| 6. Sostituire il Pod e riparare guasti |     15 | Dati conservati e servizio ripristinato  |
| 7. Dimostrazione e cleanup             |     25 | Risorse rimosse e verificate             |

I file lunghi si copiano **nell'editor**: non occorre digitarli. Ogni blocco terminale contiene una singola operazione. Eseguire un blocco e controllare il risultato prima del successivo.

**Scegliere i comandi del proprio ambiente:** le sezioni 2–4 riportano sia **Linux — terminale EC2** sia **Windows — PowerShell sul PC**. Eseguire soltanto la variante scelta; i contenuti di app.py, Dockerfile e .dockerignore sono comuni.

- **Linux:** costruire e provare i container sulla EC2.
- **Windows:** costruire e provare i container con Docker Desktop. Usare `http://127.0.0.1:8080` per queste prove, poi seguire il passaggio dei sorgenti alla EC2 prima della sezione 5.
- **Kubernetes su AWS, sezioni 5–6:** per entrambi i percorsi i comandi si eseguono nella EC2 Linux, tramite Session Manager nel browser. K3s non si installa direttamente in PowerShell. Dal browser del PC si verifica l'app remota all'IPv4 pubblico, porta 30080.

La sezione 1 prepara la EC2 per entrambi i percorsi. I container, i file e le variabili del PC non compaiono automaticamente sul server.

## Windows - preparare PowerShell sul PC

Per i blocchi **Windows — PowerShell sul PC** usare Docker Desktop con container Linux e PowerShell 7.3 o successivo. Se mancano, completarli una sola volta prima della lezione:

1. **PowerShell:** aprire la [pagina Microsoft](https://learn.microsoft.com/en-us/powershell/scripting/install/install-powershell-on-windows), scaricare l'installer MSI della versione stabile per il proprio PC, aprirlo → **Next → Install → Finish**. Dal menu Start aprire **PowerShell 7**.
2. **WSL 2:** aprire PowerShell **come amministratore** e verificare:

```powershell
wsl --status
```

Solo se WSL manca, installarlo:

```powershell
wsl --install --no-distribution
```

Riavviare Windows se richiesto, poi aggiornare WSL:

```powershell
wsl --update
```

3. **Docker Desktop:** aprire la [pagina ufficiale Windows](https://docs.docker.com/desktop/setup/install/windows-install/), scaricare l'installer adatto al PC e avviarlo. Se proposta, scegliere l'opzione **WSL 2**. Completare l'installazione, riavviare se richiesto e aprire **Docker Desktop** dal menu Start. In **Settings → General**, attivare **Use the WSL 2 based engine**, se l'opzione è disponibile, e applicare. Attendere che il motore sia avviato; dal menu Docker scegliere **Switch to Linux containers** solo se si stanno usando container Windows.
4. Se l'installazione richiede permessi amministrativi o la virtualizzazione risulta disabilitata, far completare la configurazione al supporto della postazione. Non serve aprire una shell Ubuntu: i comandi Windows del lab si eseguono in PowerShell.

Aprire una nuova finestra PowerShell dedicata al lab, senza privilegi amministrativi. Verificare la versione:

```powershell
$PSVersionTable.PSVersion
```

Se appare 5.1, aprire PowerShell 7 dal menu Start. Verificare il motore:

```powershell
docker version
```

Devono comparire Client e Server. Controllare il tipo di container:

```powershell
docker info --format '{{.OSType}}'
```

Atteso `linux`. Se il daemon non risponde, avviare Docker Desktop; se compare `windows`, passare a Linux containers dal menu Docker Desktop. Non eseguire `sudo`, `apt-get`, `dnf` o `systemctl` in PowerShell.

Controllare che il contesto punti al proprio Docker Desktop, non a un server remoto:

```powershell
docker context ls
```

I nomi del lab devono essere liberi anche nel Docker Desktop locale. Se esistono risorse di altri esercizi con gli stessi nomi, far verificare la postazione prima di continuare.

Preparare la cartella in Documenti, anche se reindirizzata su OneDrive:

```powershell
$LabDir = Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'lab07-windows'
```

Creare la cartella:

```powershell
New-Item -ItemType Directory -Path $LabDir -Force
```

Entrarvi:

```powershell
Set-Location $LabDir
```

Verificare il percorso prima di aprire i file o lanciare una build:

```powershell
Get-Location
```

Nei blocchi Windows, `notepad.exe` sostituisce `nano`. Incollare il contenuto nell'editor, salvare in **UTF-8** e scegliere **Tutti i file**: `Dockerfile`, `.dockerignore` e `app.py` non devono diventare file `.txt`. Per salvare usare Ctrl+S; Ctrl+O/Ctrl+X sono invece le scorciatoie descritte per nano su Linux.

Controllare i nomi, comprese le estensioni:

```powershell
Get-ChildItem -Force
```

Per HTTP usare `curl.exe`, non `curl`. Per leggere il codice di uscita di un comando Docker/AWS/curl, eseguire immediatamente:

```powershell
$LASTEXITCODE
```

## Preparazione dell’ambiente e confini

Verificare prima della lezione un'esecuzione completa sull'AMI scelta: Session Manager, installazione Docker/K3s, download immagini, storage local-path e accesso NodePort dalla rete dell'aula. Per rispettare i tempi, predisporre una macchina sostitutiva già pronta per i problemi di provisioning/download.

K3s è una distribuzione Kubernetes installata **sulla EC2 didattica**. Non si crea un cluster EKS, un load balancer AWS o un ruolo IAM. Il nodo singolo e il suo disco non offrono alta disponibilità: perdere la EC2 può perdere anche i dati. SQLite viene usato con **una replica**, senza esercizi di scalabilità orizzontale sullo stesso file.

L'app usa HTTP, non ha autenticazione ed è pensata soltanto per questo lab con dati sintetici e accesso limitato all'IP dell'aula. Il server Python è didattico. Non riutilizzarlo come servizio di produzione.

Se Academy non consente il provisioning, usare una VM Ubuntu di laboratorio con gli stessi strumenti e connettività controllata. La verifica richiede comunque un'app e un cluster reali: non basta compilare una descrizione.

## 1. Preparare una EC2 dedicata - 40 minuti

### 1.0 Creare la rete AWS - guida breve completa

Usare la **Console web**, su Windows o Linux. Sostituire `SIGLA` con una sigla minuscola, es. `ab12`. Annotare gli ID delle risorse create; lasciare invariate le reti esistenti.

1. **AWS:** Academy → **Start Lab** → indicatore verde → **AWS**. Regione: **US East (N. Virginia), us-east-1**. Aprire **VPC**.

2. **VPC:** **Your VPCs → Create VPC → VPC only**. Name tag: `lab07-SIGLA-vpc`; IPv4: **Manual input**, `10.70.0.0/16`; IPv6: **No IPv6 CIDR block**; Tenancy: **Default** → **Create VPC**.

3. **DNS:** selezionare la VPC → **Actions → Edit VPC settings**. Abilitare **DNS resolution** e **DNS hostnames** → **Save changes**. Mantenere il DHCP option set predefinito, con **AmazonProvidedDNS**.

4. **Subnet:** **Subnets → Create subnet**. VPC: quella appena creata; Name: `lab07-SIGLA-public`; AZ: **us-east-1a** (o altra AZ standard disponibile); VPC CIDR: `10.70.0.0/16`; Subnet CIDR: `10.70.1.0/24` → **Create subnet**.

5. **IPv4 pubblico:** selezionare la subnet → **Actions → Edit subnet settings** → abilitare **Enable auto-assign public IPv4 address** → **Save**.

6. **Gateway:** **Internet gateways → Create internet gateway**. Name: `lab07-SIGLA-igw` → **Create internet gateway**. Poi **Actions → Attach to VPC** → `lab07-SIGLA-vpc` → **Attach internet gateway**. Stato atteso: **Attached**.

7. **Route table:** **Route tables → Create route table**. Name: `lab07-SIGLA-rt-public`; VPC: `lab07-SIGLA-vpc` → **Create route table**. In **Routes → Edit routes → Add route**: Destination `0.0.0.0/0`; Target **Internet Gateway** → proprio `igw-...` → **Save changes**. Conservare `10.70.0.0/16 → local`.

8. **Associazione:** nella route table → **Subnet associations → Edit subnet associations** → selezionare solo `lab07-SIGLA-public` → **Save associations**. Lasciare **Main: No**.

9. **Security group:** **Security groups → Create security group**. Name: `lab07-SIGLA-sg`; Description: `Accesso web laboratorio 07`; VPC: `lab07-SIGLA-vpc`. **Inbound → Add rule:** tipo **Custom TCP**, TCP **8080** e TCP **30080**, come due regole separate; Source **My IP**, cioè IP pubblico del PC con `/32`. **Outbound:** **All traffic → 0.0.0.0/0** → **Create security group**. Nessun ingresso SSH o All traffic.

10. **Controllo:** **Subnets → `lab07-SIGLA-public` → Route table**: verificare `lab07-SIGLA-rt-public` e `0.0.0.0/0 → igw-...`, **Active**. Mantenere la **Network ACL predefinita**, con traffico consentito in ingresso e uscita.

**Per la EC2:** scegliere questa VPC, questa subnet e questo security group; **Auto-assign public IP: Enable**. Non servono NAT Gateway, Elastic IP o VPC endpoint. Se cambia l'IP del PC, aggiornare **My IP /32** nel gruppo.

**IAM:** **Advanced details → IAM instance profile → LabInstanceProfile**, fornito da Academy con `LabRole`. Se manca, verificare account/sessione con il supporto Academy; la policy studente non consente di creare un nuovo ruolo IAM.

### 1.1 Creare l'istanza

Nella Console **us-east-1**, aprire **EC2 → Instances → Launch instances**. In **Network settings → Edit**, scegliere VPC e subnet della tabella. In **Firewall (security groups)** scegliere **Select existing security group**, rimuovere eventuali selezioni automatiche e selezionare solo il proprio gruppo. Impostare:

| Campo                | Valore                                                                               |
| -------------------- | ------------------------------------------------------------------------------------ |
| Nome                 | `lab07-SIGLA`                                                                        |
| AMI                  | **Ubuntu Server 26.04 LTS ufficiale Canonical**, x86_64, con SSM Agent verificato    |
| Tipo                 | `t3.medium`, On-Demand, tenancy default                                              |
| Disco                | 20 GiB gp3, Delete on termination attivo                                             |
| Rete                 | VPC `lab07-SIGLA-vpc`, subnet `lab07-SIGLA-public`; Auto-assign public IP **Enable** |
| Key pair             | Nessuna: accesso con Session Manager                                                 |
| IAM instance profile | `LabInstanceProfile` già esistente                                                   |
| Metadata             | IMDSv2 richiesto, hop limit 1                                                        |
| Security group       | **Select existing security group** → `lab07-SIGLA-sg`                                |
| Ingresso Docker      | TCP **8080**, sorgente **My IP /32**                                                 |
| Ingresso Kubernetes  | TCP **30080**, sorgente **My IP /32**                                                |

Non aprire 22, 6443 o l'intero intervallo NodePort su Internet. Usare la VPC creata nella sezione 1.0; non aggiungere NAT Gateway o identità IAM. Questa VM sostituisce quella eliminata al termine del lab 06: non è necessario averla conservata. Si usa Ubuntu qui per uniformare l'ambiente K3s dell'aula.

**Prima di premere Launch instance:** espandere **Configure storage → Advanced** e verificare **Delete on termination** per il disco radice; in **Advanced details** selezionare `LabInstanceProfile`, **Metadata version: V2 only (token required)** e **Metadata response hop limit: 1**. Dopo l'avvio, in **Instances → lab07-SIGLA → Networking** verificare VPC/subnet corrette e un **Public IPv4 address** non vuoto. In **Security** deve esserci solo `lab07-SIGLA-sg`. Annotare ID istanza e volume radice.

Attendere i controlli di stato e aprire `Connect → Session Manager`. Annotare ID istanza, IPv4 pubblico, IPv4 privato, ID volume radice e security group per il cleanup.

**Se Session Manager non si connette:** attendere qualche minuto; verificare i controlli di stato, il profilo in **Actions → Security → Modify IAM role**, la route della subnet e l'uscita del security group. Se il profilo era assente, scegliere `LabInstanceProfile` e salvare. Usare l'AMI ufficiale indicata, con agente SSM; non aprire la porta 22 per sostituire questi controlli.

### 1.2 Preparare strumenti e cartella sulla EC2 - anche da un PC Windows

Entrare in Bash:

```bash
bash
```

Aggiornare l'indice dei pacchetti:

```bash
sudo apt-get update
```

Installare gli strumenti sulla propria VM:

```bash
sudo apt-get install -y docker.io nano curl
```

Avviare Docker:

```bash
sudo systemctl enable --now docker
```

Verificare Client e Server:

```bash
sudo docker version
```

Creare la cartella:

```bash
mkdir -p "$HOME/lab07"
```

Entrare nella cartella:

```bash
cd "$HOME/lab07"
```

I nomi `officina07`, `officina07-net`, `officina07-data` e il namespace `officina07` sono riservati alla propria macchina nuova. Non riutilizzare una VM condivisa con risorse omonime.

## 2. Costruire un'applicazione utilizzabile - 25 minuti

### 2.1 Creare il sorgente

Aprire l'editor:

**Linux — terminale EC2:**

```bash
nano app.py
```

**Windows — PowerShell sul PC:**

```powershell
notepad.exe .\app.py
```

**File app.py — incollare nell'editor.** Modificare soltanto il titolo predefinito con la propria sigla; non serve conoscere ogni riga Python per svolgere il deployment.

```python
import html
import json
import os
import sqlite3
from contextlib import closing
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from pathlib import Path
from urllib.parse import parse_qs, urlsplit

VERSION = "1.0"
TITLE = os.environ.get("APP_TITLE", "Officina Ticket - SIGLA")
DATA = Path(os.environ.get("DATA_DIR", "/data"))
DB = DATA / "tickets.db"
NOT_READY = Path("/tmp/officina-not-ready")


def query(sql, values=()):
    with closing(sqlite3.connect(DB, timeout=5)) as db:
        with db:
            return db.execute(sql, values).fetchall()


class App(BaseHTTPRequestHandler):
    def reply(self, status, body, content_type="text/html; charset=utf-8"):
        payload = body.encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(payload)))
        self.send_header("Cache-Control", "no-store")
        self.end_headers()
        self.wfile.write(payload)

    def do_GET(self):
        path = urlsplit(self.path).path
        if path == "/healthz":
            if NOT_READY.exists():
                return self.reply(503, "Temporaneamente non pronto", "text/plain")
            try:
                query("SELECT COUNT(*) FROM tickets")
            except sqlite3.Error:
                return self.reply(503, "Database non disponibile", "text/plain")
            return self.reply(200, "ok", "text/plain")

        if path not in ("/", "/api/tickets"):
            return self.reply(404, "Risorsa non trovata", "text/plain")

        rows = query("SELECT id, title, done FROM tickets ORDER BY id DESC")
        if path == "/api/tickets":
            data = [{"id": i, "title": t, "done": bool(d)} for i, t, d in rows]
            return self.reply(200, json.dumps(data), "application/json")

        items = []
        for ticket_id, title, done in rows:
            action = "<strong>Risolto</strong>" if done else (
                '<form method="post" action="/done">'
                f'<input type="hidden" name="id" value="{ticket_id}">'
                '<button>Segna risolto</button></form>'
            )
            items.append(
                f"<li><span>#{ticket_id} {html.escape(title)}</span>{action}</li>"
            )
        listing = "".join(items) or "<li>Nessun ticket: aprine uno.</li>"
        page = f"""<!doctype html>
<html lang="it"><meta charset="utf-8"><title>{html.escape(TITLE)}</title>
<style>
body {{ font: 18px system-ui; max-width: 850px; margin: 40px auto; padding: 20px; }}
h1 {{ color: #146c94; }} input {{ padding: 12px; width: 60%; }}
button {{ padding: 12px; cursor: pointer; }} li {{ padding: 16px; background: #edf6fa;
margin: 10px 0; display: flex; justify-content: space-between; gap: 16px; }}
ul {{ padding: 0; }} small {{ color: #555; }}
</style>
<h1>{html.escape(TITLE)}</h1><small>Versione {VERSION}</small>
<form method="post" action="/tickets">
<input name="title" maxlength="120" required placeholder="Nuova richiesta sintetica">
<button>Apri ticket</button></form><ul>{listing}</ul>
<p><a href="/api/tickets">Dati JSON</a> | <a href="/healthz">Controllo disponibilità</a></p>
</html>"""
        self.reply(200, page)

    def do_POST(self):
        path = urlsplit(self.path).path
        if path not in ("/tickets", "/done"):
            return self.reply(404, "Risorsa non trovata", "text/plain")
        try:
            length = int(self.headers.get("Content-Length", "0"))
        except ValueError:
            return self.reply(400, "Lunghezza non valida", "text/plain")
        if not 0 <= length <= 4096:
            return self.reply(413, "Richiesta troppo grande", "text/plain")
        fields = parse_qs(self.rfile.read(length).decode("utf-8", errors="replace"))
        if path == "/tickets":
            title = fields.get("title", [""])[0].strip()
            if not 1 <= len(title) <= 120:
                return self.reply(400, "Titolo richiesto, massimo 120 caratteri", "text/plain")
            query("INSERT INTO tickets(title, done) VALUES (?, 0)", (title,))
        else:
            try:
                ticket_id = int(fields.get("id", [""])[0])
            except ValueError:
                return self.reply(400, "ID non valido", "text/plain")
            query("UPDATE tickets SET done=1 WHERE id=?", (ticket_id,))
        self.send_response(303)
        self.send_header("Location", "/")
        self.send_header("Content-Length", "0")
        self.end_headers()


if __name__ == "__main__":
    DATA.mkdir(parents=True, exist_ok=True)
    query("CREATE TABLE IF NOT EXISTS tickets "
          "(id INTEGER PRIMARY KEY, title TEXT NOT NULL, done INTEGER NOT NULL)")
    print(f"Officina {VERSION} - database {DB}", flush=True)
    ThreadingHTTPServer(("0.0.0.0", int(os.environ.get("PORT", "8000"))), App).serve_forever()
```

Salvare con **Ctrl+O**, Invio e uscire con **Ctrl+X**. Il database contiene titolo e stato dei ticket; le query usano parametri e i titoli vengono resi come testo HTML.

### 2.2 Scrivere la ricetta dell'immagine

Aprire il Dockerfile:

**Linux — terminale EC2:**

```bash
nano Dockerfile
```

**Windows — PowerShell sul PC:**

```powershell
notepad.exe .\Dockerfile
```

**Contenuto del file:**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN mkdir /data && chown 10001:10001 /data
ENV PYTHONUNBUFFERED=1
USER 10001:10001
EXPOSE 8000
CMD ["python", "app.py"]
```

Salvare e uscire. L'app viene eseguita con UID 10001, non come root. Il comando RUN prepara il percorso dei dati; EXPOSE documenta la porta senza pubblicarla.

Aprire il file di esclusioni:

**Linux — terminale EC2:**

```bash
nano .dockerignore
```

**Windows — PowerShell sul PC:**

```powershell
notepad.exe .\.dockerignore
```

**Contenuto del file:**

```text
.git
.aws
.env
*.db
*.tar
*.log
k8s.yaml
install-k3s.sh
```

Salvare e uscire. La build copia soltanto `app.py`; database e credenziali non devono diventare parte dell'immagine.

Scaricare esplicitamente la base per renderne disponibile anche il digest al controllo successivo. Usare il comando del proprio ambiente:

**Linux — terminale EC2:**

```bash
sudo docker pull python:3.12-slim
```

**Windows — PowerShell sul PC:**

```powershell
docker pull --platform linux/amd64 python:3.12-slim
```

Non ripetere il pull tra le build 1.0 e 2.0.

Costruire:

**Linux — terminale EC2:**

```bash
sudo docker build -t officina07:1.0 .
```

**Windows — PowerShell sul PC:**

```powershell
docker build --platform linux/amd64 -t officina07:1.0 .
```

Controllare l'immagine:

**Linux — terminale EC2:**

```bash
sudo docker image inspect officina07:1.0 --format '{{.Id}} {{.Config.User}}'
```

**Windows — PowerShell sul PC:**

```powershell
docker image inspect officina07:1.0 --format '{{.Id}} {{.Config.User}}'
```

Annotare l'ID. Registrare anche il digest della base usata:

**Linux — terminale EC2:**

```bash
sudo docker image inspect python:3.12-slim --format '{{json .RepoDigests}}'
```

**Windows — PowerShell sul PC:**

```powershell
docker image inspect python:3.12-slim --format '{{json .RepoDigests}}'
```

### 2.3 Prima esecuzione e missione dello studente

Creare una rete dedicata:

**Linux — terminale EC2:**

```bash
sudo docker network create officina07-net
```

**Windows — PowerShell sul PC:**

```powershell
docker network create officina07-net
```

Avviare inizialmente **senza volume**, per osservare poi cosa si perde:

**Linux — terminale EC2:**

```bash
sudo docker run -d --name officina07 --network officina07-net -p 8080:8000 officina07:1.0
```

**Windows — PowerShell sul PC:**

```powershell
docker run -d --name officina07 --network officina07-net -p 127.0.0.1:8080:8000 officina07:1.0
```

Controllare la disponibilità dalla EC2:

**Linux — terminale EC2:**

```bash
curl --noproxy '*' --max-time 5 -i http://127.0.0.1:8080/healthz
```

**Windows — PowerShell sul PC:**

```powershell
curl.exe --noproxy "*" --max-time 5 -i http://127.0.0.1:8080/healthz
```

Atteso: 200 e `ok`. **Percorso Linux su EC2:** aprire `http://IP-PUBBLICO:8080`. **Percorso Windows con Docker Desktop:** aprire la prova locale dal PC:

```powershell
Start-Process "http://127.0.0.1:8080"
```

**Missione:** creare «Stampante laboratorio offline», «PC aula senza rete» e «Aggiornare browser di prova». Segnare il primo ticket come risolto. Aprire il collegamento **Dati JSON** e trovare lo stato corrispondente. Usare solo richieste inventate.

**Checkpoint pratico:** verificare che l'app consenta di creare e risolvere ticket e che l'API restituisca gli stati aggiornati.

## 3. Cambiare versione e verificare la rete - 20 minuti

### 3.1 Il file sorgente non aggiorna il container

Riaprire il codice:

**Linux — terminale EC2:**

```bash
nano app.py
```

**Windows — PowerShell sul PC:**

```powershell
notepad.exe .\app.py
```

Cambiare `VERSION = "1.0"` in `VERSION = "2.0"` e il colore del titolo da `#146c94` a `#7b2cbf`. Salvare e uscire.

Ricaricare il browser: mostra ancora V1. Costruire la nuova immagine:

**Linux — terminale EC2:**

```bash
sudo docker build -t officina07:2.0 .
```

**Windows — PowerShell sul PC:**

```powershell
docker build --platform linux/amd64 -t officina07:2.0 .
```

Ricaricare ancora: il container rimane V1. La build crea un artefatto, non sostituisce l'esecuzione.

Fermare il container:

**Linux — terminale EC2:**

```bash
sudo docker stop officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker stop officina07
```

Rimuoverlo:

**Linux — terminale EC2:**

```bash
sudo docker rm officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker rm officina07
```

Avviare V2:

**Linux — terminale EC2:**

```bash
sudo docker run -d --name officina07 --network officina07-net -p 8080:8000 officina07:2.0
```

**Windows — PowerShell sul PC:**

```powershell
docker run -d --name officina07 --network officina07-net -p 127.0.0.1:8080:8000 officina07:2.0
```

Verificare nel browser **V2 e lista vuota**. I ticket erano nel filesystem del vecchio container. Non erano nell'immagine e non sono stati trasferiti dalla build. Conservare screenshot 1 del nuovo rilascio.

### 3.2 Un client dentro la rete Docker

Da un container temporaneo interrogare l'API mediante nome e porta interna:

**Linux — terminale EC2:**

```bash
sudo docker run --rm --network officina07-net officina07:2.0 python -c "import urllib.request; print(urllib.request.urlopen('http://officina07:8000/api/tickets', timeout=3).read().decode())"
```

**Windows — PowerShell sul PC:**

```powershell
docker run --rm --network officina07-net officina07:2.0 python -c "import urllib.request; print(urllib.request.urlopen('http://officina07:8000/api/tickets', timeout=3).read().decode())"
```

Atteso: JSON, inizialmente `[]`. La porta è **8000**, non 8080.

Provare intenzionalmente la porta errata:

**Linux — terminale EC2:**

```bash
sudo docker run --rm --network officina07-net officina07:2.0 python -c "import urllib.request; print(urllib.request.urlopen('http://officina07:8080/api/tickets', timeout=3).read().decode())"
```

**Windows — PowerShell sul PC:**

```powershell
docker run --rm --network officina07-net officina07:2.0 python -c "import urllib.request; print(urllib.request.urlopen('http://officina07:8080/api/tickets', timeout=3).read().decode())"
```

È previsto un errore di connessione. Correggere **solo la porta**, ripetendo il primo comando.

Ora provare il loopback del client:

**Linux — terminale EC2:**

```bash
sudo docker run --rm --network officina07-net officina07:2.0 python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000/api/tickets', timeout=3).read().decode())"
```

**Windows — PowerShell sul PC:**

```powershell
docker run --rm --network officina07-net officina07:2.0 python -c "import urllib.request; print(urllib.request.urlopen('http://127.0.0.1:8000/api/tickets', timeout=3).read().decode())"
```

Il client temporaneo sta eseguendo Python come client, non il server. Il suo `127.0.0.1` non è il container Officina. Correggere **solo l'host**, ripetendo il primo comando.

**Checkpoint pratico:** lo studente deve far riuscire la richiesta interna e spiegare a voce la differenza tra `PC → EC2:8080 → app:8000` e `client Docker → officina07:8000`.

## 4. Rendere persistenti i ticket e provarne il recupero - 15 minuti

Rimuovere l'esecuzione vuota V2:

**Linux — terminale EC2:**

```bash
sudo docker rm -f officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker rm -f officina07
```

Creare lo storage separato:

**Linux — terminale EC2:**

```bash
sudo docker volume create officina07-data
```

**Windows — PowerShell sul PC:**

```powershell
docker volume create officina07-data
```

Avviare V2 montando lo storage su `/data` e impostando il titolo tramite configurazione:

**Linux — terminale EC2:**

```bash
sudo docker run -d --name officina07 --network officina07-net -p 8080:8000 --mount source=officina07-data,target=/data -e APP_TITLE="Officina persistente - SIGLA" officina07:2.0
```

**Windows — PowerShell sul PC:**

```powershell
docker run -d --name officina07 --network officina07-net -p 127.0.0.1:8080:8000 --mount source=officina07-data,target=/data -e APP_TITLE="Officina persistente - SIGLA" officina07:2.0
```

Sostituire SIGLA nel comando. Il nuovo volume eredita i permessi della directory preparata nell'immagine. Non usare `chmod 777` se la scrittura fallisce: verificare mount e UID.

Nel browser creare tre nuovi ticket sintetici; risolverne uno. Conservare screenshot 2 con titoli, ID e stati.

Controllare il mount effettivo:

**Linux — terminale EC2:**

```bash
sudo docker inspect officina07 --format '{{json .Mounts}}'
```

**Windows — PowerShell sul PC:**

```powershell
docker inspect officina07 --format '{{json .Mounts}}'
```

### 4.1 Sostituzione e rollback senza perdere i dati

Fermare V2:

**Linux — terminale EC2:**

```bash
sudo docker stop officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker stop officina07
```

Rimuovere il container:

**Linux — terminale EC2:**

```bash
sudo docker rm officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker rm officina07
```

Avviare V1 sullo **stesso volume**:

**Linux — terminale EC2:**

```bash
sudo docker run -d --name officina07 --network officina07-net -p 8080:8000 --mount source=officina07-data,target=/data -e APP_TITLE="Officina persistente - SIGLA" officina07:1.0
```

**Windows — PowerShell sul PC:**

```powershell
docker run -d --name officina07 --network officina07-net -p 127.0.0.1:8080:8000 --mount source=officina07-data,target=/data -e APP_TITLE="Officina persistente - SIGLA" officina07:1.0
```

Nel browser verificare V1, colore precedente e **gli stessi tre ticket con gli stessi stati**. I due rilasci hanno lo stesso schema dati: questo rollback non richiede migrazioni.

### 4.2 Un backup consistente, fuori dal volume

Creare una copia SQLite tramite la funzione di backup del database, evitando una copia grezza mentre potrebbe essere scritto:

**Linux — terminale EC2:**

```bash
sudo docker exec officina07 python -c "import sqlite3; src=sqlite3.connect('/data/tickets.db'); dst=sqlite3.connect('/tmp/tickets-backup.db'); src.backup(dst); dst.close(); src.close()"
```

**Windows — PowerShell sul PC:**

```powershell
docker exec officina07 python -c "import sqlite3; src=sqlite3.connect('/data/tickets.db'); dst=sqlite3.connect('/tmp/tickets-backup.db'); src.backup(dst); dst.close(); src.close()"
```

Copiare il backup sull'host EC2:

**Linux — terminale EC2:**

```bash
sudo docker cp officina07:/tmp/tickets-backup.db ./tickets-backup.db
```

**Windows — PowerShell sul PC:**

```powershell
docker cp officina07:/tmp/tickets-backup.db ./tickets-backup.db
```

Verificare il contenuto della copia con un container temporaneo e un mount in sola lettura:

**Linux — terminale EC2:**

```bash
sudo docker run --rm --mount type=bind,source="$PWD/tickets-backup.db",target=/backup.db,readonly officina07:1.0 python -c "import sqlite3; db=sqlite3.connect('file:/backup.db?mode=ro', uri=True); print(db.execute('SELECT id,title,done FROM tickets').fetchall()); db.close()"
```

**Windows — PowerShell sul PC:**

```powershell
docker run --rm --mount "type=bind,source=$((Get-Location).Path)\tickets-backup.db,target=/backup.db,readonly" officina07:1.0 python -c "import sqlite3; db=sqlite3.connect('file:/backup.db?mode=ro', uri=True); print(db.execute('SELECT id,title,done FROM tickets').fetchall()); db.close()"
```

I record devono corrispondere a quelli mostrati nel browser. Il backup è separato dal container e dal volume Docker, ma **è ancora sullo stesso disco EC2**: non protegge dalla perdita dell'istanza.

Non è richiesta la conservazione di questo database sintetico dopo la lezione. Per una strategia reale servirebbero anche una copia esterna e una prova di ripristino.

## Windows - passare il progetto alla EC2 prima di Kubernetes

Questo passaggio serve soltanto a chi ha svolto le sezioni 2–4 in PowerShell con Docker Desktop. Per il percorso Linux i file e l'immagine sono già sulla EC2: passare alla sezione 5.

Usare la EC2 preparata nella sezione 1. Trasferiamo i tre file di testo tramite l'editor della sessione remota e ricostruiamo l'immagine sul server. **Si tratta di una nuova build dagli stessi sorgenti**: il suo ID può differire dall'immagine locale. Il database Docker locale non viene trasferito; la prova Kubernetes parte con una nuova lista di ticket, come previsto dal lab.

**Windows — PowerShell**, dalla cartella lab07-windows: copiare il sorgente V2 negli appunti:

```powershell
Get-Content -Raw .\app.py | Set-Clipboard
```

**EC2 — Session Manager:** entrare nella cartella creata nella sezione 1:

```bash
cd "$HOME/lab07"
```

Aprire il file remoto, incollare con la funzione Incolla del terminale, poi salvare con Ctrl+O, Invio e uscire con Ctrl+X:

```bash
nano app.py
```

**Windows — PowerShell:** copiare il Dockerfile:

```powershell
Get-Content -Raw .\Dockerfile | Set-Clipboard
```

**EC2 — Session Manager:** aprire, incollare e salvare:

```bash
nano Dockerfile
```

**Windows — PowerShell:** copiare le esclusioni:

```powershell
Get-Content -Raw .\.dockerignore | Set-Clipboard
```

**EC2 — Session Manager:** aprire, incollare e salvare:

```bash
nano .dockerignore
```

Sulla EC2 costruire la versione 2:

```bash
sudo docker build -t officina07:2.0 .
```

Verificare che l'immagine remota esista:

```bash
sudo docker image inspect officina07:2.0 --format '{{.Id}}'
```

Continuare con la sezione 5 **nel terminale EC2**. Non eseguire l'installer K3s, `sudo k3s` o i comandi apt-get in PowerShell.

## 5. Distribuire la stessa app su Kubernetes - 30 minuti

### 5.1 Installare K3s sulla propria VM didattica

**Dove:** terminale **EC2 → Connect → Session Manager**, anche da Windows. Le impostazioni necessarie sono queste:

| Servizio / componente   | Impostazione del lab                                                                     |
| ----------------------- | ---------------------------------------------------------------------------------------- |
| Docker                  | Servizio `docker` attivo; immagini Linux                                                 |
| Kubernetes              | K3s `server`, canale `stable`, un nodo sulla EC2                                         |
| Ingress / load balancer | `traefik` e `servicelb` disabilitati nell'installer                                      |
| DNS Kubernetes          | CoreDNS, installato automaticamente da K3s                                               |
| Storage                 | Provisioner `local-path`, installato da K3s; PVC `officina-data`, `1Gi`, `ReadWriteOnce` |
| Metriche                | Metrics Server incluso in K3s; necessario solo per `kubectl top`                         |
| App                     | Namespace `officina07`; Deployment `officina`; una replica; strategia `Recreate`         |
| Accesso                 | Service `officina`, `NodePort`; `port: 80`, `targetPort: 8000`, `nodePort: 30080`        |
| Configurazione          | ConfigMap `officina-config`, variabile `APP_TITLE`                                       |
| Dati / readiness        | Mount `/data`; HTTP `/healthz` sulla porta `8000`                                        |

Il comando di installazione seguente crea il cluster e i componenti di base. Il file `k8s.yaml` della sezione 5.3 crea namespace, ConfigMap, PVC, Deployment e Service: non devono esistere prima. SQLite è incluso nell'immagine Python; non serve creare RDS o un database esterno.

Scaricare l'installer ufficiale in un file:

```bash
curl -fL https://get.k3s.io -o install-k3s.sh
```

Lo script modifica la **VM dedicata** e installa il servizio Kubernetes. Usare il canale stable; per ripetere esattamente una versione, specificare `INSTALL_K3S_VERSION` al posto del canale. Non eseguire su macchine personali o cluster condivisi.

Installare un singolo server, senza ingress Traefik o ServiceLB:

```bash
sudo env INSTALL_K3S_CHANNEL=stable sh install-k3s.sh server --disable traefik --disable servicelb
```

Verificare il servizio K3s:

```bash
sudo systemctl is-active k3s
```

Atteso `active`. Se è fermo, avviarlo:

```bash
sudo systemctl enable --now k3s
```

Se l'avvio fallisce, leggere il motivo prima di continuare:

```bash
sudo journalctl -u k3s -n 30 --no-pager
```

Attendere che il nodo sia pronto:

```bash
sudo k3s kubectl wait --for=condition=Ready node --all --timeout=180s
```

Controllare il nodo:

```bash
sudo k3s kubectl get nodes -o wide
```

Verificare la StorageClass `local-path`:

```bash
sudo k3s kubectl get storageclass
```

Attendere il provisioner dello storage:

```bash
sudo k3s kubectl -n kube-system rollout status deployment/local-path-provisioner --timeout=120s
```

Attendere il DNS del cluster:

```bash
sudo k3s kubectl -n kube-system rollout status deployment/coredns --timeout=120s
```

Controllare i componenti installati, incluso `metrics-server`:

```bash
sudo k3s kubectl -n kube-system get deployments
```

Se l'installazione o il nodo non sono pronti, diagnosticare il problema o usare una macchina di laboratorio funzionante. Non dichiarare eseguita la parte Kubernetes sulla base della sola lettura del YAML.

### 5.2 Portare l'immagine nel runtime del cluster

Docker e il containerd di K3s hanno archivi immagini distinti. Esportare la V2 già costruita:

```bash
sudo docker save -o officina07-2.0.tar officina07:2.0
```

Importare nell'archivio Kubernetes:

```bash
sudo k3s ctr -n k8s.io images import officina07-2.0.tar
```

Controllare che ci sia `docker.io/library/officina07:2.0`:

```bash
sudo k3s ctr -n k8s.io images list
```

Il manifesto userà `imagePullPolicy: Never` per usare proprio questa immagine. Non basta che `docker image ls` la mostri. Nel lab 06 si è distribuito via registry; qui si trasferisce lo stesso artefatto tramite archivio in un cluster a nodo singolo.

### 5.3 Creare il manifesto

Verificare che il namespace sia libero:

```bash
sudo k3s kubectl get namespace officina07
```

Atteso **NotFound**. Un errore di connessione non prova l'assenza del namespace; se esiste già, verificarne origine e contenuto prima di procedere.

Aprire il file:

```bash
nano k8s.yaml
```

**File k8s.yaml — incollare nell'editor e sostituire SIGLA nel titolo:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: officina07
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: officina-config
  namespace: officina07
data:
  APP_TITLE: "Officina Kubernetes - SIGLA"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: officina-data
  namespace: officina07
spec:
  storageClassName: local-path
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: officina
  namespace: officina07
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: officina07
  template:
    metadata:
      labels:
        app: officina07
    spec:
      initContainers:
        - name: prepara-dati
          image: docker.io/library/officina07:2.0
          imagePullPolicy: Never
          command: ["python", "-c", "import os; os.chown('/data', 10001, 10001)"]
          securityContext:
            runAsUser: 0
            runAsGroup: 0
          volumeMounts:
            - name: dati
              mountPath: /data
      containers:
        - name: app
          image: docker.io/library/officina07:2.0
          imagePullPolicy: Never
          envFrom:
            - configMapRef:
                name: officina-config
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 2
            periodSeconds: 3
            failureThreshold: 2
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 128Mi
          securityContext:
            runAsNonRoot: true
            runAsUser: 10001
            runAsGroup: 10001
            allowPrivilegeEscalation: false
          volumeMounts:
            - name: dati
              mountPath: /data
      volumes:
        - name: dati
          persistentVolumeClaim:
            claimName: officina-data
---
apiVersion: v1
kind: Service
metadata:
  name: officina
  namespace: officina07
spec:
  type: NodePort
  selector:
    app: officina07
  ports:
    - port: 80
      targetPort: 8000
      nodePort: 30080
```

Salvare e uscire. L'init container prepara i permessi del volume, poi termina; l'app gira come utente non privilegiato. Il volume Kubernetes è **nuovo**: non contiene automaticamente il database Docker.

Distribuire le risorse:

```bash
sudo k3s kubectl apply -f k8s.yaml
```

Attendere il deployment:

```bash
sudo k3s kubectl -n officina07 rollout status deployment/officina --timeout=180s
```

Controllare Pod, storage e servizio:

```bash
sudo k3s kubectl -n officina07 get pods,pvc,service
```

Attesi: Pod `1/1`, PVC `Bound`, NodePort `30080`. Il local-path provisioner alloca una directory sul nodo: non crea un nuovo volume EBS gestito da AWS.

### 5.4 Usare il servizio distribuito

Nel browser del PC aprire **http://IP-PUBBLICO:30080**. Deve apparire «Officina Kubernetes - SIGLA», versione 2, lista inizialmente vuota.

**Windows — verifica dell'app AWS da PowerShell:** inserire l'IPv4 pubblico attuale della EC2:

```powershell
$Ec2Ip = Read-Host "IPv4 pubblico della EC2"
```

Aprire il servizio Kubernetes remoto:

```powershell
Start-Process "http://${Ec2Ip}:30080"
```

Verificare la readiness HTTP attraverso il servizio:

```powershell
curl.exe --noproxy "*" --max-time 5 -i "http://${Ec2Ip}:30080/healthz"
```

Leggere i ticket del deployment remoto:

```powershell
curl.exe --noproxy "*" --max-time 5 -i "http://${Ec2Ip}:30080/api/tickets"
```

Ripetere queste verifiche dopo le prove della sezione 6. Il vecchio indirizzo `127.0.0.1:8080` mostra ancora l'app Docker locale e **non verifica** il deployment Kubernetes su AWS.

Creare tre ticket diversi da quelli della prova Docker e risolverne uno. Conservare screenshot 3. Il traffico segue `browser → EC2:30080 → Service → Pod:8000`; qui si prova il percorso NodePort, non un semplice port-forward.

Ricaricare una pagina per generare traffico e leggere i log:

```bash
sudo k3s kubectl -n officina07 logs deployment/officina -c app --tail=15
```

Cercare la propria richiesta. Leggere anche le misure, se il servizio metriche è già pronto:

```bash
sudo k3s kubectl -n officina07 top pod
```

Se compare Metrics API not available, attendere qualche minuto e riprovare una volta; il funzionamento dell'app non dipende da quel comando. Non inventare valori.

## 6. Ricreare, riconfigurare e riparare - 15 minuti

**Tutti i comandi di questa sezione si eseguono nella EC2 Linux**, da **AWS Console → EC2 → Instances → lab07-SIGLA → Connect → Session Manager → Connect**, anche se il PC è Windows. Le richieste `curl.exe` verso l’IPv4 pubblico si eseguono invece nel PowerShell del PC.

### 6.1 Il controller ricrea il Pod; il volume conserva i ticket

Annotare il nome attuale:

```bash
sudo k3s kubectl -n officina07 get pods
```

Eliminare **il solo Pod dell'app del lab**, selezionato tramite la sua etichetta:

```bash
sudo k3s kubectl -n officina07 delete pod -l app=officina07 --wait=true
```

Attendere che il Deployment torni disponibile:

```bash
sudo k3s kubectl -n officina07 rollout status deployment/officina --timeout=120s
```

Leggere il nuovo nome:

```bash
sudo k3s kubectl -n officina07 get pods
```

Nel browser devono restare gli stessi ticket e gli stessi stati. Il controller ha ricreato il Pod, non ricostruito il database. Se la pagina viene provata durante la sostituzione, è normale una breve interruzione.

### 6.2 Cambiare una configurazione senza ricostruire l'immagine

Aprire il manifesto:

```bash
nano k8s.yaml
```

Modificare **soltanto** `ConfigMap → data → APP_TITLE` in «Officina turno pomeriggio - SIGLA». Salvare e uscire.

Applicare:

```bash
sudo k3s kubectl apply -f k8s.yaml
```

Ricaricare il browser: il processo esistente conserva il vecchio titolo. Le variabili d'ambiente non si aggiornano in un processo già avviato.

Sostituire il Pod mediante rollout:

```bash
sudo k3s kubectl -n officina07 rollout restart deployment/officina
```

Attendere:

```bash
sudo k3s kubectl -n officina07 rollout status deployment/officina --timeout=120s
```

Verificare **nuovo titolo, stessa versione 2, stessi ticket**. Nessuna nuova build è stata necessaria.

### 6.3 Guasto di instradamento: selector errato

Cambiare soltanto il selector del Service:

```bash
sudo k3s kubectl -n officina07 set selector service/officina app=inesistente
```

Controllare gli endpoint dopo qualche secondo:

```bash
sudo k3s kubectl -n officina07 get endpointslices -l kubernetes.io/service-name=officina -o yaml
```

Non devono esserci indirizzi di Pod selezionati. Aprire una nuova richiesta dal browser: l'app non è raggiungibile tramite il Service. Il Pod può rimanere pronto.

Confrontare le sue etichette:

```bash
sudo k3s kubectl -n officina07 get pods --show-labels
```

Riparare il selector:

```bash
sudo k3s kubectl -n officina07 set selector service/officina app=officina07
```

Verificare che ricompaiano gli indirizzi negli EndpointSlice:

```bash
sudo k3s kubectl -n officina07 get endpointslices -l kubernetes.io/service-name=officina
```

Riprovare nel browser e aprire un nuovo ticket. **La riparazione deve restituire un'app utilizzabile**, non soltanto un comando senza errori.

### 6.4 Guasto di disponibilità: processo vivo, Pod non pronto

L'app include un interruttore didattico: quando esiste `/tmp/officina-not-ready`, `/healthz` risponde 503 mentre il processo rimane attivo.

Attivarlo nel solo container applicativo:

```bash
sudo k3s kubectl -n officina07 exec deployment/officina -c app -- python -c "from pathlib import Path; Path('/tmp/officina-not-ready').touch()"
```

Attendere l'esito della readiness:

```bash
sudo k3s kubectl -n officina07 wait --for=condition=Ready=false pod -l app=officina07 --timeout=60s
```

Osservare stato e conteggio riavvii:

```bash
sudo k3s kubectl -n officina07 get pods
```

Atteso: `0/1`, processo Running, nessun riavvio causato dalla sola readiness. Non è configurata una liveness probe.

Leggere la condizione degli endpoint:

```bash
sudo k3s kubectl -n officina07 get endpointslices -l kubernetes.io/service-name=officina -o yaml
```

L'indirizzo può ancora essere presente, ma con `ready: false`. Una nuova richiesta al servizio non deve essere instradata verso il Pod non pronto.

Ripristinare la disponibilità:

```bash
sudo k3s kubectl -n officina07 exec deployment/officina -c app -- python -c "from pathlib import Path; Path('/tmp/officina-not-ready').unlink(missing_ok=True)"
```

Attendere il ritorno a Ready:

```bash
sudo k3s kubectl -n officina07 wait --for=condition=Ready pod -l app=officina07 --timeout=60s
```

Riaprire l'app, segnare un ticket come risolto e conservare screenshot 4. Verificare che i dati siano rimasti e il servizio sia tornato utilizzabile.

## 7. Dimostrazione finale e cleanup - 25 minuti

### Verifiche pratiche finali

| Competenza       | Prova dal vivo                                |
| ---------------- | --------------------------------------------- |
| Build e rilascio | Pagina V2 ottenuta da un Dockerfile           |
| Rete             | Client Docker → app:8000; browser → EC2:30080 |
| Persistenza      | Ticket presenti dopo la sostituzione          |
| Configurazione   | Titolo cambiato tramite ConfigMap             |
| Kubernetes       | Nuovo Pod con gli stessi dati                 |
| Diagnosi         | Selector e readiness ripristinati             |
| Chiusura         | Namespace e risorse AWS rimossi               |

Conservare sul PC `app.py`, `Dockerfile`, `.dockerignore`, `k8s.yaml` e i quattro screenshot. I sorgenti possono essere quelli copiati e modificati nell'editor locale; se sono stati modificati soltanto su EC2, visualizzarli **uno alla volta** e copiarli in file di testo locali prima di terminare l'istanza.

Visualizzare il sorgente:

```bash
cat app.py
```

Visualizzare il Dockerfile:

```bash
cat Dockerfile
```

Visualizzare le esclusioni:

```bash
cat .dockerignore
```

Visualizzare il manifesto finale:

```bash
cat k8s.yaml
```

Non consegnare kubeconfig, credenziali, token SSM o chiavi. I ticket sono sintetici e possono essere eliminati.

### Cleanup Kubernetes

Annotare il PV associato alla PVC prima della rimozione:

```bash
sudo k3s kubectl -n officina07 get pvc officina-data -o wide
```

Controllarne la reclaim policy:

```bash
sudo k3s kubectl get pv
```

Eliminare il solo namespace del lab:

```bash
sudo k3s kubectl delete namespace officina07 --wait=true --timeout=120s
```

Verificarne l'assenza:

```bash
sudo k3s kubectl get namespace officina07
```

Atteso **NotFound**. Un AccessDenied o un errore di connessione non dimostra la cancellazione.

Controllare che il PV annotato sia stato rimosso secondo la policy Delete:

```bash
sudo k3s kubectl get pv
```

Se è ancora presente, controllare provisioning/eventi e annotare il residuo; non rimuovere finalizer né volumi estranei.

### Cleanup Docker

Eseguire i comandi **sul motore usato nelle sezioni 2–4**: terminale EC2 per il percorso Linux, PowerShell sul PC per il percorso Windows. Nel percorso Windows la EC2 contiene il cluster e l'immagine ricostruita, ma non il container, la rete e il volume Docker locali. Non tentare di rimuoverli sul server.

Rimuovere il proprio container:

**Linux — terminale EC2:**

```bash
sudo docker rm -f officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker rm -f officina07
```

Eliminare il volume sintetico:

**Linux — terminale EC2:**

```bash
sudo docker volume rm officina07-data
```

**Windows — PowerShell sul PC:**

```powershell
docker volume rm officina07-data
```

Eliminare la rete:

**Linux — terminale EC2:**

```bash
sudo docker network rm officina07-net
```

**Windows — PowerShell sul PC:**

```powershell
docker network rm officina07-net
```

Verificare i container:

**Linux — terminale EC2:**

```bash
sudo docker ps -a --filter name=officina07
```

**Windows — PowerShell sul PC:**

```powershell
docker ps -a --filter name=officina07
```

Verificare il volume:

**Linux — terminale EC2:**

```bash
sudo docker volume ls --filter name=officina07-data
```

**Windows — PowerShell sul PC:**

```powershell
docker volume ls --filter name=officina07-data
```

**Solo Windows — rimuovere le due immagini applicative locali dopo il container:**

```powershell
docker image rm officina07:1.0 officina07:2.0
```

Conservare i sorgenti e gli screenshot. Il backup sintetico tickets-backup.db può essere eliminato dal PC se non serve più:

```powershell
Remove-Item -LiteralPath .\tickets-backup.db
```

Completare anche il cleanup AWS: chiudere Docker Desktop non termina la EC2 e non elimina i suoi dischi.

### Cleanup AWS

Dopo aver salvato i sorgenti, seguire questo ordine:

1. In **EC2 → Instances** selezionare solo `lab07-SIGLA` → **Instance state → Terminate (delete) instance** → confermare. Attendere **Terminated**: Stop non elimina i dischi.
2. In **EC2 → Volumes** cercare l'ID del disco radice annotato all'avvio. Deve essere scomparso grazie a **Delete on termination**. Se è rimasto **Available**, verificarne la proprietà e cancellare solo quel disco con **Actions → Delete volume**.
3. In **EC2 → Network Interfaces** filtrare per l'ID della VPC del lab e attendere il rilascio dell'interfaccia della EC2. Non forzare la cancellazione di interfacce ancora in uso.
4. In **VPC → Security groups** selezionare `lab07-SIGLA-sg` → **Actions → Delete security groups** → confermare.
5. In **VPC → Subnets** selezionare `lab07-SIGLA-public` → **Actions → Delete subnet** → confermare. Controllare che non sia più elencata.
6. In **VPC → Route tables** selezionare `lab07-SIGLA-rt-public`, verificare **Main: No** e nessuna associazione residua → **Actions → Delete route table** → confermare.
7. In **VPC → Internet gateways** selezionare `lab07-SIGLA-igw` → **Actions → Detach from VPC** → confermare. Poi **Actions → Delete internet gateway** → confermare.
8. In **VPC → Your VPCs** selezionare `lab07-SIGLA-vpc` → **Actions → Delete VPC**. Controllare l'elenco delle risorse residue e confermare solo per la propria VPC. I suoi oggetti default vengono rimossi con essa; non cancellare il DHCP option set condiviso.
9. Aggiornare gli elenchi e verificare l'assenza dei propri ID. Se compare **DependencyViolation**, leggere la dipendenza indicata e completare la rimozione della propria risorsa prima di riprovare. Annotare eventuali residui; solo dopo chiudere Learner Lab.

La terminazione elimina anche K3s, gli archivi immagini, il backup locale e i dati rimasti sul disco radice. Non occorre disinstallare K3s prima di terminare questa VM dedicata. Non lasciare una t3.medium accesa dopo la lezione: EC2, EBS e IPv4 pubblico consumano budget.

## Estensioni pratiche per chi termina prima

Scegliere un'attività, eseguirla prima del cleanup e verificarne il risultato.

- **Immagine mancante:** modificare nel solo container app del Deployment il tag in `9.9`, applicare e osservare `ErrImageNeverPull`. Ripristinare `2.0` nel manifesto, riapplicare e verificare app e ticket.
- **Errore applicativo:** richiedere `/pagina-assente`, verificare HTTP 404 nei log mentre `/healthz` continua a dare 200. Tornare alla pagina principale senza riavviare il servizio.
- **Personalizzazione utile:** aggiungere nel rendering della pagina il numero dei ticket aperti, costruire `3.0`, importare l'immagine nel runtime e aggiornare entrambi i riferimenti immagine del manifesto. Verificare il contatore creando e risolvendo ticket, poi tornare a `2.0` conservando il database.
- **Esportazione del backup Docker:** avviare un container temporaneo con il file `tickets-backup.db` montato in sola lettura, esportarne i record in JSON e confrontarli con i ticket salvati prima del passaggio a Kubernetes. Questa è una verifica di lettura, non un ripristino completo dell'app. Non sovrascrivere il database del Pod.

## Diagnosi rapida

| Sintomo                                   | Controllo                                | Intervento mirato                                               |
| ----------------------------------------- | ---------------------------------------- | --------------------------------------------------------------- |
| App non parte                             | `sudo docker logs officina07`            | Correggere errori di copia/indentazione, ricostruire e ricreare |
| Browser bloccato, curl sulla EC2 funziona | IP pubblico e regola 8080/30080 da My IP | Correggere la sola regola del proprio security group            |
| SQLite permission denied                  | UID, mount e init container              | Correggere proprietà della directory dati; non chmod 777        |
| ErrImageNeverPull                         | Nome completo e archivio containerd      | Importare l'immagine corretta nel namespace k8s.io              |
| PVC Pending                               | StorageClass local-path ed eventi        | Riparare il provisioner; non inventare un volume disponibile    |
| Pod Init in errore                        | Log di prepara-dati                      | Verificare mount e permessi della directory                     |
| Pod Running ma 0/1                        | Readiness ed endpoint healthz            | Ripristinare il controllo applicativo                           |
| Titolo vecchio dopo ConfigMap             | Variabili del processo                   | Riavviare il Deployment e verificare                            |
| Namespace Terminating                     | Eventi e risorse residue                 | Segnalare il blocco; non forzare finalizer                      |

Per diagnosticare i log dell'app:

```bash
sudo k3s kubectl -n officina07 logs deployment/officina -c app --tail=30
```

Per diagnosticare l'init container:

```bash
sudo k3s kubectl -n officina07 logs deployment/officina -c prepara-dati --tail=30
```

Per leggere gli eventi del lab:

```bash
sudo k3s kubectl -n officina07 get events --sort-by=.metadata.creationTimestamp
```

Per esaminare la richiesta storage:

```bash
sudo k3s kubectl -n officina07 describe pvc officina-data
```

## Riferimenti operativi

- [Docker: Dockerfile](https://docs.docker.com/reference/dockerfile/)
- [Docker: volumi](https://docs.docker.com/engine/storage/volumes/)
- [K3s: installazione](https://docs.k3s.io/quick-start)
- [K3s: requisiti del nodo](https://docs.k3s.io/installation/requirements)
- [K3s: importare immagini](https://docs.k3s.io/add-ons/import-images)
- [K3s: storage locale](https://docs.k3s.io/add-ons/storage)
- [Kubernetes: Service e NodePort](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes: readiness e liveness](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [AWS: AMI con SSM Agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)


- [Docker Desktop su Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
- [Microsoft: passaggio di argomenti in PowerShell](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_parsing)

- [AWS: creare una VPC](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html)
- [AWS: creare una subnet](https://docs.aws.amazon.com/vpc/latest/userguide/create-subnets.html)
- [AWS: IPv4 pubblico della subnet](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-public-ip.html)
- [AWS: Internet Gateway e route](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-igw.html)
- [AWS: associazioni delle route table](https://docs.aws.amazon.com/vpc/latest/userguide/WorkWithRouteTables.html)
- [AWS: impostazioni DNS](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns-updating.html)
- [AWS: creazione del security group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-security-group.html)
- [AWS: eliminazione della VPC e dipendenze](https://docs.aws.amazon.com/vpc/latest/userguide/delete-vpc.html)

- [K3s: servizi di rete inclusi](https://docs.k3s.io/networking/networking-services)
