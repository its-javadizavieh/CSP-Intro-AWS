# Lab 02 - Reti Wi-Fi e VPN

## Obiettivo

Raccogliere parametri Wi-Fi, progettare la separazione tra rete interna e ospiti e verificare il percorso prima e dopo una VPN senza modificare l’infrastruttura.

## Durata (timebox)

90 minuti: 20 raccolta Wi-Fi, 25 schema, 30 confronto VPN, 15 consegna e cleanup.

## Prerequisiti

Computer con Wi-Fi oppure collegato via Ethernet (in questo caso analizzare i parametri radio sintetici); su Windows usare PowerShell normale (non è richiesta l’esecuzione come amministratore). Se i dettagli Wi-Fi sono bloccati, usare il percorso alternativo descritto sotto. La VPN è opzionale e deve essere già autorizzata e configurata dall’organizzazione: non creare account, certificati o tunnel. Cartella di lavoro `Documenti/lab02-wifi-vpn`.

## Scenario

Un’organizzazione usa SSID separati per dipendenti e ospiti. Un dipendente remoto accede a una risorsa privata mediante VPN. Occorre documentare copertura, autenticazione, segmentazione ed estremi del tunnel.

## Step (numerati)

1. Creare `Documenti/lab02-wifi-vpn/consegna02.md` e aprire il terminale nella cartella di lavoro; su Windows seguire i comandi dello svolgimento guidato. Eseguire solo i passi relativi al proprio sistema operativo.
2. Ubuntu: aprire `Impostazioni -> Wi-Fi -> Ingranaggio della rete connessa`; annotare SSID, intensità segnale, frequenza, velocità collegamento, IPv4, gateway e DNS disponibili.
3. macOS: tenere premuto `Option` e selezionare l’icona Wi-Fi; annotare SSID, BSSID, canale, RSSI, rumore, Tx Rate e standard PHY. Non riportare il BSSID nella consegna pubblica.
4. Windows: seguire il percorso guidato sotto: raccogliere configurazione IP e dati visibili nelle Impostazioni, poi provare `netsh wlan show interfaces`. Se compare la richiesta di posizione, l’errore 5 o il messaggio che `wlansvc` non è in esecuzione, documentare il limite e proseguire con il percorso alternativo; non è necessario sbloccare il comando per completare il lab.
5. Ubuntu, se NetworkManager è disponibile: `nmcli -f ACTIVE,SSID,SIGNAL,CHAN,RATE,SECURITY dev wifi > wifi.txt`.
6. macOS: salvare manualmente in `wifi.txt` i valori mostrati dal menu diagnostico; non eseguire scansioni aggressive.
7. In `consegna02.md`, creare la tabella `Parametro | Valore | Fonte`: SSID anonimizzato, banda `2.4/5/6 GHz`, canale, intensità, metodo di sicurezza, IPv4, gateway e DNS. Indicare la fonte (comando, Impostazioni o caso sintetico); per i dati reali non leggibili scrivere `non disponibile`, senza inventare misure.
8. Disegnare in `schema-wifi.md` questo percorso: `Client interno -> AP -> VLAN interna -> risorse interne/Internet` e `Client ospite -> AP -> VLAN ospiti -> solo Internet`.
9. Usare i parametri logici: VLAN interna `10`, rete esempio `10.10.10.0/24`; VLAN ospiti `20`, rete esempio `10.10.20.0/24`; regola firewall inter-VLAN che nega dalla VLAN 20 alla VLAN 10; DNS approvato e accesso Internet previsti per entrambe. La sola presenza di SSID diversi non dimostra isolamento.
10. Specificare che gli indirizzi sono sintetici e non devono essere applicati alla rete reale.
11. Senza VPN, salvare anche la configurazione IP e le rotte in `configurazione-senza-vpn.txt`, usando i comandi dello svolgimento guidato con questo nome file. Se una VPN istituzionale deve rimanere attiva, non disconnetterla: usare il caso sintetico per il confronto. Salvare il percorso a `example.com` in `percorso-senza-vpn.txt`: Ubuntu/macOS `traceroute example.com`; Windows `tracert -d example.com`.
12. Se è disponibile una VPN autorizzata, aprire il client già installato, selezionare esclusivamente il profilo fornito dall’organizzazione e connettersi. Non cambiare server, protocollo, DNS o certificati.
13. Salvare la nuova configurazione in `configurazione-con-vpn.txt`: Ubuntu `ip address && ip route`; macOS `ifconfig && netstat -rn`; Windows PowerShell `Get-NetIPConfiguration` e `Get-NetRoute`.
14. Salvare il percorso con VPN in `percorso-con-vpn.txt` usando lo stesso comando del punto 11.
15. Identificare gli estremi: `client VPN` e `gateway VPN`; indicare se il traffico Internet usa full tunnel o split tunnel solo quando è deducibile dalle rotte.
16. Se non è disponibile una VPN autorizzata, creare `caso-vpn-sintetico.md` con client `192.0.2.10`, gateway pubblico `198.51.100.20`, rete privata `10.20.0.0/16` e rotta tunnel `10.20.0.0/16 -> interfaccia VPN`.
17. Compilare due checklist: “SSID visibile ma autenticazione fallita” e “VPN connessa ma risorsa privata non raggiungibile”.

## Svolgimento guidato

### Windows: raccogliere dati e interpretare l’errore WLAN

**Eseguire un blocco alla volta, nell’ordine indicato, nella stessa finestra PowerShell.** Copiare tutta la riga e premere Invio. Anche se una riga lunga va a capo sullo schermo, è un unico comando: includere tutte le parti separate da `|`. Alcuni comandi salvano dati senza mostrare un risultato a video; se ricompare il prompt senza errori, passare al successivo.

Aprire **PowerShell** normale. Preparare la cartella Documenti, anche se reindirizzata su OneDrive. I comandi seguenti presumono che la VPN sia disconnessa; se deve restare attiva, sostituire `configurazione-senza-vpn.txt` con `configurazione-attuale.txt` e usare il caso sintetico per il confronto:

Impostare il percorso della cartella di lavoro:

```powershell
$lab = Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'lab02-wifi-vpn'
```

Creare la cartella, se non esiste già:

```powershell
New-Item -ItemType Directory -Path $lab -Force | Out-Null
```

Entrare nella cartella di lavoro:

```powershell
Set-Location $lab
```

Creare il file della consegna solo se non esiste già:

```powershell
if (-not (Test-Path 'consegna02.md')) { New-Item -ItemType File -Path 'consegna02.md' | Out-Null }
```

Visualizzare le schede di rete e il loro stato:

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, LinkSpeed
```

Salvare la configurazione IP nel file indicato (sostituisce il contenuto precedente):

```powershell
Get-NetIPConfiguration | Format-List * | Out-File configurazione-senza-vpn.txt -Encoding utf8
```

Aggiungere le rotte allo stesso file, conservando la configurazione IP già salvata:

```powershell
Get-NetRoute | Format-Table -AutoSize | Out-String -Width 240 | Out-File configurazione-senza-vpn.txt -Append -Encoding utf8
```

Salvare il risultato dell’interrogazione Wi-Fi, compresi eventuali messaggi di errore:

```powershell
netsh wlan show interfaces 2>&1 | Out-File wifi.txt -Encoding utf8
```

Leggere il file appena salvato:

```powershell
Get-Content wifi.txt
```

`Get-NetAdapter` identifica le schede e il loro stato; `Get-NetIPConfiguration` mostra indirizzi, gateway e DNS; `Get-NetRoute` mostra le rotte. Collegare i dati alla scheda Wi-Fi tramite nome/indice: una scheda Ethernet o VPN può avere indirizzi diversi. `LinkSpeed` è la velocità di collegamento dichiarata, non una misura della velocità Internet. Questi comandi non sostituiscono le misure radio di `netsh`.

**Se `netsh` funziona:** annotare i valori disponibili nella tabella della consegna.

**Se compare “Il servizio Configurazione automatica wireless (wlansvc) non è in esecuzione”:** Windows segnala che il servizio che gestisce le connessioni WLAN non è avviato. Questo caso è distinto dal consenso alla posizione: aprire la pagina Posizione non avvia il servizio. Verificare senza modificare il sistema:

Controllare lo stato del servizio WLAN:

```powershell
Get-Service -Name WlanSvc | Select-Object Name, Status, StartType
```

Visualizzare le schede fisiche riconosciute da Windows:

```powershell
Get-NetAdapter -Physical | Select-Object Name, InterfaceDescription, Status, LinkSpeed
```

Visualizzare la configurazione IP delle connessioni disponibili:

```powershell
Get-NetIPConfiguration
```

- `Status = Stopped` significa servizio fermo; `Running` significa in esecuzione. `StartType` descrive il tipo di avvio, non lo stato attuale. Se il servizio non viene trovato o la lettura è negata, annotare il messaggio e proseguire con il caso sintetico.
- Osservare quali schede fisiche Windows riconosce e quale ha configurazione IP e gateway. Il servizio fermo **non dimostra l’assenza della scheda Wi-Fi**; una scheda non elencata può anche avere un problema di driver. Non dedurre la presenza dell’hardware dal solo errore di `netsh`.
- Se si sta usando Ethernet, raccogliere IPv4, gateway e DNS di quella scheda indicando `Fonte: Ethernet reale`. Una connessione cablata può funzionare anche con `WlanSvc` fermo. Proseguire con schema e confronto VPN sulla connessione disponibile.
- Conservare il messaggio in `wifi.txt`, aggiungere manualmente lo stato del servizio e annotare `Parametri radio reali non disponibili: servizio WLAN fermo`. Usare per la parte Wi-Fi il caso sintetico riportato sotto, separato dai dati Ethernet. Non è necessario avviare il servizio per completare il lab; eventuali interventi sul PC dell’aula spettano al docente o al supporto tecnico.

**Se `netsh` segnala che non sono presenti interfacce wireless:** annotarlo e usare lo stesso percorso Ethernet + caso sintetico. È possibile, per esempio, che si stia lavorando su un PC senza Wi-Fi o in una macchina virtuale che espone solo Ethernet.

**Se nel file compare `non ├¿` ma a video compare `non è`:** è un problema di codifica del testo salvato, distinto dallo stato del servizio. In PowerShell `cat` è un alias di `Get-Content`: legge il file già creato e non ripete la diagnosi. Eseguire `netsh wlan show interfaces` a video per leggere il messaggio corrente e, se necessario, trascriverlo correttamente in `wifi.txt`. Salvare in UTF-8 non ripara caratteri già decodificati male.

**Se compare “autorizzazione di posizione” / `WlanQueryInterface` errore 5:** il sistema sta negando l’accesso ad alcune informazioni WLAN. I dati sulle reti vicine possono essere usati per ricavare la posizione. Il messaggio non dimostra che la scheda sia guasta, che la password Wi-Fi sia sbagliata o che Internet non funzioni. L’esecuzione come amministratore non sostituisce il consenso alla posizione.

1. Conservare il messaggio in `wifi.txt` e annotare nella consegna: `Interrogazione WLAN bloccata dai permessi; connettività da verificare separatamente`.
2. Aprire `Impostazioni -> Rete e Internet -> Wi-Fi`, quindi le proprietà della rete connessa o `Proprietà hardware` (le voci variano con la versione). Trascrivere in `wifi.txt` i valori disponibili, senza BSSID/MAC; mantenere il messaggio già salvato. Per i campi mancanti scrivere `non disponibile`.
3. Completare comunque indirizzo IP, gateway e DNS usando la configurazione salvata. Non dedurre banda, canale o sicurezza dal solo indirizzo IP.

**Caso Wi-Fi sintetico comune (permessi bloccati, servizio fermo o Wi-Fi assente):** per esercitarsi sui parametri radio mancanti, analizzare **separatamente** questo caso sintetico: SSID `LAB-DIPENDENTI`, banda `5 GHz`, canale `36`, segnale `78%`, autenticazione `WPA2-Enterprise`, velocità collegamento `433 Mbit/s`. Non presentarlo come misurazione del proprio PC. La percentuale del segnale non è un valore RSSI in dBm.

**Facoltativo, solo per il blocco di posizione e se consentito sul dispositivo:** aprire la pagina Posizione con:

```powershell
Start-Process 'ms-settings:privacy-location'
```

È la versione PowerShell del comando `start ms-settings:privacy-location` indicato da Windows. Il comando apre solo le Impostazioni. Se consentito dal docente e dalle regole del dispositivo, annotare le impostazioni iniziali, abilitare i servizi di posizione e l’accesso richiesto dalle opzioni disponibili, quindi riprovare `netsh wlan show interfaces`. Se le opzioni sono gestite dall’organizzazione o l’errore persiste, usare il percorso alternativo; non cambiare criteri, registro o privilegi. Il lab è completo anche senza questa operazione.

Rispondere in `consegna02.md`:

- Quali dati descrivono il collegamento radio e quali la configurazione IP?
- Perché il blocco di un’interrogazione WLAN non dimostra l’assenza di connettività?
- Qual è la differenza tra servizio WLAN fermo, permesso di posizione negato e scheda Wi-Fi non rilevata? Quali verifiche aiutano a distinguerli?
- Un segnale forte garantisce autenticazione riuscita e accesso a Internet? Motivare.
- La velocità di collegamento equivale alla velocità di un download? Motivare.

Riferimenti: [Microsoft — Modifiche all’accesso Wi-Fi e alla posizione](https://learn.microsoft.com/it-it/windows/win32/nativewifi/wi-fi-access-location-changes) e [Microsoft — Diagnosi delle connessioni wireless e ruolo di WlanSvc](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/wireless-network-connectivity-issues-troubleshooting).

### Confronto prima e dopo la VPN

Su Windows, dalla stessa cartella, prima della VPN:

```powershell
tracert -d example.com 2>&1 | Out-File percorso-senza-vpn.txt -Encoding utf8
```

Dopo aver connesso esclusivamente la VPN già autorizzata:

Salvare la configurazione IP nel file indicato (sostituisce il contenuto precedente):

```powershell
Get-NetIPConfiguration | Format-List * | Out-File configurazione-con-vpn.txt -Encoding utf8
```

Aggiungere le rotte allo stesso file, conservando la configurazione IP già salvata:

```powershell
Get-NetRoute | Format-Table -AutoSize | Out-String -Width 240 | Out-File configurazione-con-vpn.txt -Append -Encoding utf8
```

Salvare il percorso verso example.com con la VPN connessa:

```powershell
tracert -d example.com 2>&1 | Out-File percorso-con-vpn.txt -Encoding utf8
```

Confrontare interfacce, indirizzi e rotte nei due file di configurazione. `-d` evita la risoluzione dei nomi dei router intermedi; `example.com` richiede comunque DNS. Gli asterischi indicano risposte mancanti entro il timeout, non dimostrano da soli un guasto. Un percorso Internet invariato è compatibile con uno split tunnel: il solo `tracert` non prova se il traffico verso la rete privata passa nella VPN. Valutare IPv4 e IPv6 separatamente quando presenti e scrivere `non determinabile` se le rotte non bastano.

Percorso senza VPN: aggiungere `> percorso-senza-vpn.txt 2>&1` al comando del punto 11; dopo la connessione usare `> percorso-con-vpn.txt 2>&1`. Configurazione Linux: `ip address > configurazione-con-vpn.txt` e `ip route >> configurazione-con-vpn.txt`; macOS: `ifconfig > configurazione-con-vpn.txt` e `netstat -rn >> configurazione-con-vpn.txt`; PowerShell: `Get-NetIPConfiguration > configurazione-con-vpn.txt` e `Get-NetRoute >> configurazione-con-vpn.txt`.

Caso sintetico completo quando manca la VPN (dati da analizzare, non da applicare):

| Momento | Destinazione | Uscita                    |
| ------- | ------------ | ------------------------- |
| Prima   | 0.0.0.0/0    | Gateway della rete locale |
| Prima   | 10.20.0.0/16 | Nessuna rotta specifica   |
| Dopo    | 0.0.0.0/0    | Gateway della rete locale |
| Dopo    | 10.20.0.0/16 | Interfaccia VPN           |

È uno split tunnel IPv4 del caso. `10.20.0.10` segue la rotta /16; `example.com` segue la default se non esistono rotte più specifiche. Il gateway VPN pubblico è `198.51.100.20`; non eseguire test verso questi indirizzi sintetici. Registrare separatamente che DNS e autorizzazione applicativa non sono dimostrati da questa tabella.

Configurazione sintetica di riferimento:

- SSID dipendenti: `AZIENDA-INTERNA`, WPA2/WPA3-Enterprise;
- SSID ospiti: `AZIENDA-OSPITI`, isolamento client;
- VLAN interna: `10`, rete `10.10.10.0/24`;
- VLAN ospiti: `20`, rete `10.10.20.0/24`;
- rete privata remota: `10.20.0.0/16`;
- accesso dalla rete ospiti alla rete interna: negato.

Le prove sono di sola lettura. Non mostrare password, chiavi, certificati, BSSID o indirizzi pubblici personali.

## Output atteso

`consegna02.md`, `wifi.txt`, `schema-wifi.md`, `configurazione-senza-vpn.txt`, `percorso-senza-vpn.txt` e, in alternativa, la coppia `configurazione-con-vpn.txt` / `percorso-con-vpn.txt` oppure `caso-vpn-sintetico.md`. Se la VPN istituzionale non può essere disconnessa, indicare che le misure senza VPN non sono disponibili e consegnare il confronto sintetico.

`wifi.txt` può contenere il messaggio di errore e i dati trascritti dalle Impostazioni: non è richiesto un output `netsh` riuscito. Prima della consegna anonimizzare anche i file di testo (SSID reali, BSSID/MAC, nomi macchina, IP pubblici e dettagli istituzionali).

## Checkpoint

Lo schema prevede l’isolamento della rete ospiti (non verificato sulla rete reale); i dati misurati sono distinti da quelli sintetici e dai dati non disponibili; il tunnel ha due estremi identificati; la checklist distingue segnale, autenticazione, indirizzamento, rotta e autorizzazione.

## Troubleshooting rapido

- Windows, `wlansvc` non in esecuzione: leggere lo stato con `Get-Service WlanSvc`; raccogliere i dati della connessione disponibile e usare il caso Wi-Fi sintetico.
- Windows, caratteri come `├¿` nel file: problema di codifica; leggere il messaggio corrente a video e trascriverlo se necessario.
- Windows, richiesta di posizione / errore 5: seguire il percorso alternativo; distinguere permessi del comando e stato della connessione.
- SSID non visibile: verificare radio attiva, copertura e banda supportata.
- Autenticazione fallita: non ripetere molte password; verificare profilo e credenziali con il docente.
- VPN connessa ma destinazione irraggiungibile: controllare rotta verso `10.20.0.0/16`, DNS e autorizzazione.
- Nessuna VPN disponibile: usare obbligatoriamente il caso sintetico, senza installare software.

## Cleanup obbligatorio

Se sono state modificate per la prova, ripristinare le impostazioni di posizione annotate all’inizio. Disconnettere soltanto la VPN usata per il test, senza eliminare il profilo istituzionale. Dimenticare eventuali SSID di prova soltanto se creati per il lab. Eliminare screenshot contenenti BSSID, IP pubblici, utenti o certificati; conservare la consegna anonimizzata.
