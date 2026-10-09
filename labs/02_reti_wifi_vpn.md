# Lab 02 - Reti Wi-Fi e VPN

## Obiettivo

Raccogliere parametri Wi-Fi, progettare la separazione tra rete interna e ospiti e verificare il percorso prima e dopo una VPN senza modificare l’infrastruttura.

## Durata (timebox)

90 minuti: 20 raccolta Wi-Fi, 25 schema, 30 confronto VPN, 15 consegna e cleanup.

## Prerequisiti

Computer con Wi-Fi. La VPN è opzionale e deve essere già autorizzata e configurata dall’organizzazione: non creare account, certificati o tunnel. Cartella di lavoro `Documenti/lab02-wifi-vpn`.

## Scenario

Un’organizzazione usa SSID separati per dipendenti e ospiti. Un dipendente remoto accede a una risorsa privata mediante VPN. Occorre documentare copertura, autenticazione, segmentazione ed estremi del tunnel.

## Step (numerati)

1. Creare `Documenti/lab02-wifi-vpn/consegna02.md`.
2. Ubuntu: aprire `Impostazioni -> Wi-Fi -> Ingranaggio della rete connessa`; annotare SSID, intensità segnale, frequenza, velocità collegamento, IPv4, gateway e DNS disponibili.
3. macOS: tenere premuto `Option` e selezionare l’icona Wi-Fi; annotare SSID, BSSID, canale, RSSI, rumore, Tx Rate e standard PHY. Non riportare il BSSID nella consegna pubblica.
4. Windows: aprire `Impostazioni -> Rete e Internet -> Wi-Fi -> Proprietà hardware`; poi PowerShell e usare `netsh wlan show interfaces > wifi.txt`.
5. Ubuntu, se NetworkManager è disponibile: `nmcli -f ACTIVE,SSID,SIGNAL,CHAN,RATE,SECURITY dev wifi > wifi.txt`.
6. macOS: salvare manualmente in `wifi.txt` i valori mostrati dal menu diagnostico; non eseguire scansioni aggressive.
7. In `consegna02.md`, creare la tabella `Parametro | Valore`: SSID anonimizzato, banda `2.4/5/6 GHz`, canale, intensità, metodo di sicurezza, IPv4, gateway.
8. Disegnare in `schema-wifi.md` questo percorso: `Client interno -> AP -> VLAN interna -> risorse interne/Internet` e `Client ospite -> AP -> VLAN ospiti -> solo Internet`.
9. Usare i parametri logici: VLAN interna `10`, rete esempio `10.10.10.0/24`; VLAN ospiti `20`, rete esempio `10.10.20.0/24`; regola firewall inter-VLAN che nega dalla VLAN 20 alla VLAN 10; DNS approvato e accesso Internet previsti per entrambe. La sola presenza di SSID diversi non dimostra isolamento.
10. Specificare che gli indirizzi sono sintetici e non devono essere applicati alla rete reale.
11. Senza VPN, salvare il percorso a `example.com` in `percorso-senza-vpn.txt`: Ubuntu/macOS `traceroute example.com`; Windows `tracert example.com`.
12. Se è disponibile una VPN autorizzata, aprire il client già installato, selezionare esclusivamente il profilo fornito dall’organizzazione e connettersi. Non cambiare server, protocollo, DNS o certificati.
13. Salvare la nuova configurazione in `configurazione-con-vpn.txt`: Ubuntu `ip address && ip route`; macOS `ifconfig && netstat -rn`; Windows PowerShell `Get-NetIPConfiguration` e `Get-NetRoute`.
14. Salvare il percorso con VPN in `percorso-con-vpn.txt` usando lo stesso comando del punto 11.
15. Identificare gli estremi: `client VPN` e `gateway VPN`; indicare se il traffico Internet usa full tunnel o split tunnel solo quando è deducibile dalle rotte.
16. Se non è disponibile una VPN autorizzata, creare `caso-vpn-sintetico.md` con client `192.0.2.10`, gateway pubblico `198.51.100.20`, rete privata `10.20.0.0/16` e rotta tunnel `10.20.0.0/16 -> interfaccia VPN`.
17. Compilare due checklist: “SSID visibile ma autenticazione fallita” e “VPN connessa ma risorsa privata non raggiungibile”.

## Svolgimento guidato

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

`consegna02.md`, `wifi.txt`, `schema-wifi.md`, `percorso-senza-vpn.txt` e, in alternativa, `percorso-con-vpn.txt` oppure `caso-vpn-sintetico.md`.

## Checkpoint

La rete ospiti è separata dalla rete interna; il tunnel ha due estremi identificati; la checklist distingue segnale, autenticazione, indirizzamento, rotta e autorizzazione.

## Troubleshooting rapido

- SSID non visibile: verificare radio attiva, copertura e banda supportata.
- Autenticazione fallita: non ripetere molte password; verificare profilo e credenziali con il docente.
- VPN connessa ma destinazione irraggiungibile: controllare rotta verso `10.20.0.0/16`, DNS e autorizzazione.
- Nessuna VPN disponibile: usare obbligatoriamente il caso sintetico, senza installare software.

## Cleanup obbligatorio

Disconnettere soltanto la VPN usata per il test, senza eliminare il profilo istituzionale. Dimenticare eventuali SSID di prova soltanto se creati per il lab. Eliminare screenshot contenenti BSSID, IP pubblici, utenti o certificati; conservare la consegna anonimizzata.
