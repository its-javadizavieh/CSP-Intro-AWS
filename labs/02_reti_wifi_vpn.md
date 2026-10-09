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

1. **Preparare il lavoro e scegliere il proprio percorso.**

   In assenza di dati radio usare il caso sintetico del passo 6.

   **Eseguire solo i comandi del proprio sistema operativo, un blocco alla volta, nella stessa finestra del terminale.** Copiare tutta la riga e premere Invio: anche quando va a capo sullo schermo, la riga è un unico comando, comprese le parti separate da `|`. Alcuni comandi salvano dati senza mostrare risultati a video.

   Su Windows aprire PowerShell normale, senza privilegi amministrativi. Le prove di rete sono di sola lettura; non mostrare password, chiavi o certificati.

2. **Creare la cartella di lavoro e il file della consegna.**

   **Windows — PowerShell:** impostare il percorso di Documenti, anche se reindirizzato su OneDrive:

   ```powershell
   $lab = Join-Path ([Environment]::GetFolderPath('MyDocuments')) 'lab02-wifi-vpn'
   ```

   Creare la cartella:

   ```powershell
   New-Item -ItemType Directory -Path $lab -Force | Out-Null
   ```

   Entrare nella cartella:

   ```powershell
   Set-Location $lab
   ```

   Creare la consegna senza sovrascrivere un file esistente:

   ```powershell
   if (-not (Test-Path 'consegna02.md')) { New-Item -ItemType File -Path 'consegna02.md' | Out-Null }
   ```

   **Ubuntu/macOS:** nel gestore file creare `lab02-wifi-vpn` dentro Documenti e aprire il terminale in quella cartella. Creare il file:

   ```bash
   touch consegna02.md
   ```

   Usare un editor di testo per compilare i file Markdown richiesti nei passi successivi.

3. **Raccogliere i dati Wi-Fi del proprio sistema operativo.**

   **Windows — PowerShell:** visualizzare le schede e il loro stato:

   ```powershell
   Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, LinkSpeed
   ```

   Salvare l’interrogazione WLAN, compresi eventuali errori:

   ```powershell
   netsh wlan show interfaces 2>&1 | Out-File wifi.txt -Encoding utf8
   ```

   Leggere il risultato:

   ```powershell
   Get-Content wifi.txt
   ```

   Aprire anche `Impostazioni -> Rete e Internet -> Wi-Fi`, quindi le proprietà della rete connessa o `Proprietà hardware`. Trascrivere in `wifi.txt` i valori disponibili, senza BSSID/MAC e conservando eventuali messaggi di errore. Se il comando è bloccato, seguire il passo 4 o 5 pertinente.

   **Ubuntu:** aprire `Impostazioni -> Wi-Fi -> Ingranaggio della rete connessa`. Annotare SSID, segnale, frequenza, velocità, IPv4, gateway e DNS disponibili. Se NetworkManager è disponibile, salvare:

   ```bash
   nmcli -f ACTIVE,SSID,SIGNAL,CHAN,RATE,SECURITY dev wifi > wifi.txt
   ```

   Usare la riga della rete attiva. Se il comando non è disponibile, trascrivere i dati delle Impostazioni in `wifi.txt`.

   **macOS:** tenere premuto `Option` e selezionare l’icona Wi-Fi. Trascrivere in `wifi.txt` SSID, canale, RSSI, rumore, Tx Rate e standard PHY disponibili; non riportare il BSSID. Consultare le impostazioni della rete per gli altri dati disponibili.

   `LinkSpeed` e Tx Rate descrivono il collegamento, non misurano la velocità di un download. Non eseguire scansioni aggressive.

4. **Solo Windows: gestire il servizio WLAN fermo, le interfacce assenti o il testo illeggibile.**

   Se compare “Il servizio Configurazione automatica wireless (wlansvc) non è in esecuzione”, controllare il servizio:

   ```powershell
   Get-Service -Name WlanSvc | Select-Object Name, Status, StartType
   ```

   Visualizzare le schede fisiche riconosciute:

   ```powershell
   Get-NetAdapter -Physical | Select-Object Name, InterfaceDescription, Status, LinkSpeed
   ```

   `Stopped` significa servizio fermo; `Running` significa in esecuzione. `StartType` indica il tipo di avvio, non lo stato attuale. Se la lettura è negata o il servizio non viene trovato, annotare il messaggio.

   Il servizio fermo non dimostra l’assenza della scheda Wi-Fi. Se non sono rilevate interfacce wireless, il PC potrebbe non avere Wi-Fi, avere un problema di driver o essere una macchina virtuale con sola Ethernet. Annotare quanto osservato, senza dedurre la causa dal solo errore.

   Conservare in `wifi.txt` il messaggio e lo stato del servizio. Proseguire con Ethernet, se disponibile, e con i dati radio sintetici del passo 6. Non è necessario avviare il servizio; eventuali interventi sul PC dell’aula spettano al docente o al supporto tecnico. Aprire la pagina Posizione non avvia `WlanSvc`.

   Se nel file compare `non ├¿` anziché `non è`, leggere il messaggio corrente a video:

   ```powershell
   netsh wlan show interfaces
   ```

   Trascriverlo correttamente se necessario. È un problema di codifica distinto dal guasto segnalato; salvare in UTF-8 non ripara caratteri già decodificati male. In PowerShell `cat` è un alias di `Get-Content`: legge un file già salvato, senza ripetere la diagnosi.

5. **Solo Windows: gestire la richiesta di posizione o l’errore 5 WLAN.**

   Se il messaggio richiede l’autorizzazione di posizione, conservarlo in `wifi.txt` e annotare: `Interrogazione WLAN bloccata dai permessi; connettività da verificare separatamente`.

   Alcune informazioni WLAN possono essere usate per ricavare la posizione. Il blocco dell’interrogazione non dimostra una password errata, una scheda guasta o l’assenza di Internet. L’esecuzione come amministratore non sostituisce il consenso alla posizione.

   Usare i dati delle Impostazioni già raccolti al passo 3 e il caso sintetico del passo 6 per quelli mancanti.

   **Facoltativo, solo se consentito sul dispositivo:** aprire la pagina Posizione:

   ```powershell
   Start-Process 'ms-settings:privacy-location'
   ```

   Il comando apre solo le Impostazioni. Se consentito dal docente e dalle regole del dispositivo, annotare le impostazioni iniziali e abilitare i servizi di posizione e l’accesso richiesto dalle opzioni disponibili. Riprovare l’interrogazione del passo 3 solo dopo questa modifica. Se le opzioni sono gestite dall’organizzazione o l’errore persiste, proseguire con il caso sintetico senza cambiare criteri, registro o privilegi.

   Riferimenti: [Microsoft — Accesso Wi-Fi e posizione](https://learn.microsoft.com/it-it/windows/win32/nativewifi/wi-fi-access-location-changes) e [Microsoft — Diagnosi wireless e ruolo di WlanSvc](https://learn.microsoft.com/en-us/troubleshoot/windows-client/networking/wireless-network-connectivity-issues-troubleshooting).

6. **Registrare i parametri radio e distinguere le fonti.**

   In `consegna02.md` creare la tabella `Parametro | Valore | Fonte` con SSID anonimizzato, banda (2.4/5/6 GHz), canale, intensità e metodo di sicurezza. Per i dati reali non leggibili scrivere `non disponibile`.

   Se i dati radio mancano, analizzare separatamente questo **caso sintetico**, senza presentarlo come misurazione del PC:

   | Parametro                | Valore sintetico |
   | ------------------------ | ---------------- |
   | SSID                     | LAB-DIPENDENTI   |
   | Banda                    | 5 GHz            |
   | Canale                   | 36               |
   | Segnale                  | 78%              |
   | Autenticazione           | WPA2-Enterprise  |
   | Velocità di collegamento | 433 Mbit/s       |

   La percentuale del segnale non è un valore RSSI in dBm. Non dedurre banda, canale o sicurezza dal solo indirizzo IP.

7. **Disegnare la separazione tra rete interna e ospiti.**

   Creare `schema-wifi.md` con questi due percorsi:

   - `Client interno -> AP -> VLAN interna -> risorse interne/Internet`
   - `Client ospite -> AP -> VLAN ospiti -> solo Internet`

   **AP** significa Access Point: collega i dispositivi Wi-Fi alla rete locale.

   Usare SSID `AZIENDA-INTERNA` con WPA2/WPA3-Enterprise e `AZIENDA-OSPITI` con isolamento client. Assegnare nello schema VLAN interna `10`, rete `10.10.10.0/24`; VLAN ospiti `20`, rete `10.10.20.0/24`. Prevedere una regola firewall che neghi il traffico dalla VLAN 20 alla VLAN 10, DNS approvato e accesso Internet per entrambe.

   Indicare che si tratta di un progetto con dati sintetici, da non applicare alla rete reale. SSID diversi non dimostrano isolamento; lo schema descrive l’isolamento previsto, non una verifica sull’infrastruttura dell’aula.

8. **Salvare la configurazione IP e le rotte prima della VPN.**

   Se una VPN istituzionale deve rimanere attiva, non disconnetterla: in questo passo sostituire `configurazione-senza-vpn.txt` con `configurazione-attuale.txt`, poi saltare i passi 9–13 e usare il caso sintetico del passo 14.

   **Windows — PowerShell:** salvare indirizzi, gateway e DNS:

   ```powershell
   Get-NetIPConfiguration | Format-List * | Out-File configurazione-senza-vpn.txt -Encoding utf8
   ```

   Aggiungere le rotte allo stesso file:

   ```powershell
   Get-NetRoute | Format-Table -AutoSize | Out-String -Width 240 | Out-File configurazione-senza-vpn.txt -Append -Encoding utf8
   ```

   **Ubuntu:** salvare gli indirizzi:

   ```bash
   ip address > configurazione-senza-vpn.txt
   ```

   Aggiungere le rotte:

   ```bash
   ip route >> configurazione-senza-vpn.txt
   ```

   **macOS:** salvare le interfacce:

   ```bash
   ifconfig > configurazione-senza-vpn.txt
   ```

   Aggiungere le rotte:

   ```bash
   netstat -rn >> configurazione-senza-vpn.txt
   ```

   Aprire il file con l’editor e aggiungere alla tabella della consegna IPv4, gateway e DNS disponibili; su Ubuntu/macOS leggere i DNS nelle impostazioni di rete. Associare i dati alla scheda corretta tramite nome/indice. Se si usa Ethernet, indicare `Fonte: Ethernet reale`, separata dai dati Wi-Fi sintetici.

   `>` e `Out-File` sostituiscono il contenuto del file; `>>` e `-Append` aggiungono dati.

9. **Salvare il percorso prima della VPN.**

   **Windows — PowerShell:**

   ```powershell
   tracert -d example.com 2>&1 | Out-File percorso-senza-vpn.txt -Encoding utf8
   ```

   **Ubuntu/macOS:**

   ```bash
   traceroute -n example.com > percorso-senza-vpn.txt 2>&1
   ```

   Aprire il file con l’editor. Un **hop** è un salto lungo il percorso, generalmente attraverso un router. Le righe numerate mostrano gli hop e i tempi di risposta in millisecondi. Gli asterischi indicano risposte mancanti entro il timeout, non dimostrano da soli un guasto.

   `-d` su Windows e `-n` su Ubuntu/macOS evitano la risoluzione dei nomi dei router intermedi; `example.com` richiede comunque DNS. Se `traceroute` non è installato, annotare il limite nel file e continuare con l’analisi delle rotte, senza installare software.

10. **Scegliere il percorso VPN.**

    Se non è disponibile una VPN autorizzata, saltare i passi 11–13 e svolgere il passo 14.

    Se è disponibile, aprire il client già installato e connettersi esclusivamente con il profilo fornito dall’organizzazione. Non cambiare server, protocollo, DNS o certificati. Proseguire con il passo 11.

11. **Salvare la configurazione dopo la connessione VPN.**

    **Windows — PowerShell:** salvare la configurazione IP:

    ```powershell
    Get-NetIPConfiguration | Format-List * | Out-File configurazione-con-vpn.txt -Encoding utf8
    ```

    Aggiungere le rotte:

    ```powershell
    Get-NetRoute | Format-Table -AutoSize | Out-String -Width 240 | Out-File configurazione-con-vpn.txt -Append -Encoding utf8
    ```

    **Ubuntu:** salvare gli indirizzi:

    ```bash
    ip address > configurazione-con-vpn.txt
    ```

    Aggiungere le rotte:

    ```bash
    ip route >> configurazione-con-vpn.txt
    ```

    **macOS:** salvare le interfacce:

    ```bash
    ifconfig > configurazione-con-vpn.txt
    ```

    Aggiungere le rotte:

    ```bash
    netstat -rn >> configurazione-con-vpn.txt
    ```

12. **Salvare il percorso con la VPN connessa.**

    **Windows — PowerShell:**

    ```powershell
    tracert -d example.com 2>&1 | Out-File percorso-con-vpn.txt -Encoding utf8
    ```

    **Ubuntu/macOS:**

    ```bash
    traceroute -n example.com > percorso-con-vpn.txt 2>&1
    ```

    Aprire il file con l’editor; se il comando non è disponibile, annotare il limite come al passo 9.

13. **Confrontare le misure reali prima e dopo la VPN.**

    Confrontare interfacce, indirizzi, rotte e percorsi nei file salvati. Identificare i due estremi del tunnel: client VPN e gateway VPN, senza pubblicare dettagli istituzionali.

    Indicare full tunnel o split tunnel solo se deducibile dalle rotte; altrimenti scrivere `non determinabile`. Un percorso Internet invariato è compatibile con uno split tunnel: il solo traceroute non dimostra se il traffico verso la rete privata passa nella VPN. Considerare IPv4 e IPv6 separatamente; i comandi Ubuntu di questo lab raccolgono le rotte IPv4, quindi non consentono conclusioni sulle rotte IPv6.

    Dopo questo confronto passare al passo 15.

14. **In alternativa alle misure VPN, analizzare il caso sintetico.**

    Creare `caso-vpn-sintetico.md` con client `192.0.2.10`, gateway VPN pubblico `198.51.100.20` e rete privata remota `10.20.0.0/16`.

    | Momento | Destinazione | Uscita                    |
    | ------- | ------------ | ------------------------- |
    | Prima   | 0.0.0.0/0    | Gateway della rete locale |
    | Prima   | 10.20.0.0/16 | Nessuna rotta specifica   |
    | Dopo    | 0.0.0.0/0    | Gateway della rete locale |
    | Dopo    | 10.20.0.0/16 | Interfaccia VPN           |

    Spiegare perché questo è uno split tunnel IPv4: `10.20.0.10` segue la rotta /16; il traffico IPv4 verso `example.com` segue la default se non esistono rotte più specifiche. Identificare gli estremi del tunnel e precisare che DNS e autorizzazione applicativa non sono dimostrati dalla tabella.

    Questi dati sono solo da analizzare: non applicare rotte e non eseguire test verso gli indirizzi sintetici.

15. **Completare le domande e le checklist nella consegna.**

    Rispondere in `consegna02.md`:

    - Quali dati descrivono il collegamento radio e quali la configurazione IP?
    - Perché un’interrogazione WLAN bloccata non dimostra l’assenza di connettività?
    - Come distinguere servizio WLAN fermo, permesso di posizione negato e scheda Wi-Fi non rilevata?
    - Un segnale forte garantisce autenticazione riuscita e accesso a Internet? Motivare.
    - La velocità di collegamento equivale alla velocità di un download? Motivare.

    Compilare due checklist: “SSID visibile ma autenticazione fallita” e “VPN connessa ma risorsa privata non raggiungibile”. Distinguere segnale, autenticazione, indirizzamento, rotta, DNS e autorizzazione.

    Usare la sezione “Troubleshooting rapido” come riferimento.

16. **Verificare e consegnare i file.**

    Preparare e anonimizzare i file elencati in “Output atteso”, quindi verificare i criteri della sezione “Checkpoint”.

17. **Completare il cleanup.**

    Seguire la sezione “Cleanup obbligatorio” prima di terminare il laboratorio.

## Output atteso

`consegna02.md`, `wifi.txt`, `schema-wifi.md`, `configurazione-senza-vpn.txt`, `percorso-senza-vpn.txt` e, in alternativa, la coppia `configurazione-con-vpn.txt` / `percorso-con-vpn.txt` oppure `caso-vpn-sintetico.md`. Se la VPN istituzionale non può essere disconnessa, indicare che le misure senza VPN non sono disponibili e consegnare il confronto sintetico.

`wifi.txt` può contenere il messaggio di errore e i dati trascritti dalle Impostazioni: non è richiesto un output `netsh` riuscito. Prima della consegna anonimizzare anche i file di testo (SSID reali, BSSID/MAC, nomi macchina, IP pubblici e dettagli istituzionali).

Nel caso della VPN istituzionale sempre attiva, allegare `configurazione-attuale.txt` al posto di `configurazione-senza-vpn.txt`. I file dei percorsi possono documentare l’indisponibilità del comando.

## Checkpoint

Lo schema prevede l’isolamento della rete ospiti (non verificato sulla rete reale); i dati misurati sono distinti da quelli sintetici e dai dati non disponibili; il tunnel ha due estremi identificati; la checklist distingue segnale, autenticazione, indirizzamento, rotta e autorizzazione.

## Troubleshooting rapido

- Windows, `wlansvc` non in esecuzione: leggere lo stato con `Get-Service WlanSvc` (passo 4); raccogliere i dati della connessione disponibile e usare il caso Wi-Fi sintetico.
- Windows, caratteri come `├¿` nel file: problema di codifica; leggere il messaggio corrente a video e trascriverlo se necessario.
- Windows, richiesta di posizione / errore 5: seguire il percorso alternativo; distinguere permessi del comando e stato della connessione.
- SSID non visibile: verificare radio attiva, copertura e banda supportata.
- Autenticazione fallita: non ripetere molte password; verificare profilo e credenziali con il docente.
- VPN connessa ma destinazione irraggiungibile: controllare rotta verso `10.20.0.0/16` nel caso sintetico (verso la rete privata prevista nel caso reale), DNS e autorizzazione.
- Nessuna VPN disponibile: usare obbligatoriamente il caso sintetico, senza installare software.

## Cleanup obbligatorio

Se sono state modificate per la prova, ripristinare le impostazioni di posizione annotate all’inizio. Disconnettere soltanto la VPN usata per il test, senza eliminare il profilo istituzionale. Dimenticare eventuali SSID di prova soltanto se creati per il lab. Eliminare screenshot contenenti BSSID, IP pubblici, utenti o certificati; conservare la consegna anonimizzata.
