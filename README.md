# LEM (Linux Embedded Music)

Appunti sparsi di configurazione della scheda Rock Pi S per musicisti.  
La scheda [Rock Pi S](https://wiki.radxa.com/RockpiS), la più piccola della famiglia [Rock](https://wiki.radxa.com/Home), è provvista di un'uscita stereofonica e di 8 entrate microfoniche nella v1.1, 6 nella v1.2 e 4 nella v1.3.  
La ragione di quest'equipaggiamento risiede nel fatto che la scheda è stata concepita per applicazioni IoT con riconoscimento vocale.
A differenza delle altre schede *Raspberry-like*, che nella migliore delle ipotesi hanno un'uscita stereofonica integrata (a volte di pessima qualità) e un'entrata di linea (o microfonica), la *rockpis*, in virtù della sua dotazione hardware, si presta ad applicazioni musicali avanzate.  
La scheda si presenta in varie configurazioni hardware, ma per questo progetto è stata testata una scheda v1.1 con 512 MB di ram, WiFi e Bluetooth integrati. Il sistema operativo montato su una SD card da 8 giga è Debian Buster, [fornito dal produttore](https://wiki.radxa.com/RockpiS/downloads) e regolarmente aggiornato.

---

---

1. [Pinout audio](#pinout-audio)
2. [Collegarsi a `rockpis` tramite `ssh`](#collegarsi-a-rockpis-via-ssh-tramite-connessione-ethernet)

    - [Collegarsi a rockpis con Debian](#ssh-su-debian)
    - [Collegarsi a rockpis con Mac OS X](#ssh-su-mac-os-x)

3. [Aggiornare il sistema e installare il software per l'audio](#preparare-il-sistema)
4. [Configurazione del wifi](#configurazione-del-wifi)
5. [Installare e avviare `puredata`](#installare-e-avviare-puredata)
6. [Avviare `puredata` al boot di rockpis](#avviare-puredata-al-boot-di-rockpis)
7. [setup di `jack`](#setup-di-jack)
8. [`jack` in realtime priority](#jack-in-realtime-priority)
9. [Installazione di `supercollider`](#installazione-di-supercollider)
10. [Uso di un display virtuale con `xvfb`](#uso-di-un-display-virtuale-con-xvfb)

    - [Installazione di `xvfb`](#installazione-di-xvfb)
    - [avvio e test di sclang con `xvfb`](#avvio-e-test-di-sclang-con-xvfb)

11. [Esecuzione di uno script sc](#esecuzione-di-uno-script-sc)
12. [GPIO](#gpio)

    - [la libreria `mraa`](#la-libreria-mraa)
    - [programmazione del gpio](#programmazione-del-gpio)

13. [Opzione `python`](#opzione-python)
14. [Installare `nodejs`](#installare-nodejs)
15. [Configurare il `bluetooth`](#configurare-il-bluetooth)
16. [Access Point](#access-point)
17. [Led wifi](#led-wifi)

---

---

## Pinout audio

<p align="center">
  <img src="https://raw.githubusercontent.com/franeum/rockpis_for_audio/master/immagini/rockpis_audio_interface.png" alt="drawing" width="300"/>
</p>

_N.B. Nella versione 1.2 della scheda, i microfoni 3 e 4 sono stati eliminati, quindi restano attivi i microfoni 1,2,5,6,7,8_

---

## Collegarsi a rockpis via ssh (tramite connessione ethernet)

### ssh su Debian

1\. avviare una finestra di terminale e cercare il nome dell'interfaccia ethernet del computer:

```bash
ip a
```

l'interfaccia sarà **eth0** oppure qualcosa come **enxx**. Questo rappresenta il nostro _eth-device-name_

2\. creare una connessione con il seguente comando:

```bash
nmcli con add con-name my-eth-1 ifname <eth-device-name> \
type ethernet ipv4.method shared
```

3\. attivare la connessione:

```bash
nmcli con up my-eth-1
```

4\. verificare che la connessione sia attiva:

```bash
nmcli con show
```

5\. ottenere l'ip dell'interfaccia ethernet:

```bash
ip a
```

l'ip potrebbe essere qualcosa come 10.0.x.x oppure 192.168.x.x. Annotare i primi 3 valori (es: 10.0.1 oppure 192.168.1). questi rappresentano l'indirizzo di sottorete (_subnet_addr_) che ci interessa.

6\. collegare la _rockpis_ alla porta ethernet e pingare l'intera sottorete:

```bash
fping -r 1 -g subnet_addr.0/24
```

Nel mio caso l'ip della porta ethernet è 10.42.0.1, quindi per pingare la sottorete eseguirò il comando:

```bash
fping -r 1 -g 10.42.0.0/24
```

Se tutto ha funzionato il comando dovrebbe restituire almeno due ip, quello della scheda ethernet del computer e quello della rockpis, nel mio caso:

```bash
fping -r 1 -g 10.42.0.0/24 2> /dev/null | grep -v -i unreachable
10.42.0.1 is alive
10.42.0.250 is alive
```

7\. il primo ip è quello locale, il secondo è quello della rockpis, a questo punto possiamo connetterci (l'utente di default è _rock_) alla scheda con il protocollo ssh:

```bash
ssh rock@10.42.0.250
```

e inserire la password `rock`

---

## ssh su Mac OS X

1\. attivare la condivisione internet e il bridge
2\. da terminale cercare l'ip della scheda:

```bash
fping -g 192.168.x.0/24
```

3\. connettersi alla scheda via ssh:

```bash
ssh rock@192.168.x.x
```

oppure (più comodo):

```bash
ssh rock@rockpis
```

4\. inserire la password (`rock`)

---

## Preparare il sistema

upgrade:

```bash
sudo apt update
sudo apt dist-upgrade
```

verificare che lo spazio sulla SDcard sia giusto:

```bash
df -h
```

in caso di dimensioni errate eseguire le operazioni indicate in questo tutorial:
[link](https://www.youtube.com/watch?v=R4VovMDnsIE)

installare `alsa` e `jack`:

```bash
sudo apt install alsa-utils
sudo apt install jackd2
```

---

## Configurazione del wifi

attivare la connessione tramite `nmtui` sull'interfaccia wireless `wlan0`

```bash
sudo nmtui
```

Una volta che la connessione wi-fi del rockpis è attiva è possibile connettersi con lo stesso tramite il protocollo `ssh`:

```bash
ssh rock@192.168.1.11
```

e scollegare quindi il cavo ethernet. Il rockpis comincia a camminare con le sue gambe!

---

---

## installare e avviare puredata

```bash
sudo apt install puredata
```

con `alsamixer` possiamo configurare i dac e gli adc

starting puredata with output device 3:

```bash
puredata -nogui -alsa -audiodev 3,3 -inchannels 8 file.pd
```

Per avviare `puredata` con `jackd`:

```bash
puredata -nogui -jack file.pd
```

dopo aver [configurato opportunamente](#setup-di-jack) `jack`. Se `jack` non si auto-avvia all'avvio di `puredata`, eseguire da shell il coseguente comando:

```bash
bash ./.jackdrc
```

puredata riceve i messaggi tramite l'oggetto `netreceive`.
TODO: una _patch_ generica di ricezione.

---

## avviare puredata al boot di rockpis

1. installare `cron`:

```bash
sudo apt install cron
```

2. eseguire il seguente comando:

```bash
crontab -e
```

3. aggiungere questa voce al file che si apre:

```bash
@reboot puredata -nogui -alsa -audiodev 3,3 -inchannels 8 /path/to/file.pd
```

in questo modo all'avvio di rockpis viene eseguito il file file.pd con `pd`.

---

## Setup di jack

varificare il nome del dispositivo audio con il comando:

```bash
cat /proc/asound/cards
```

nel mio caso il dispositivo è `rockchiprk3308a`.
Per avviare `jackd` su richiesta copiare la seguente riga nel file `~/.jackdrc` (se il file non esiste, crearlo con `vim ~/.jackdrc`):

```bash
/usr/bin/jackd -R -P 95 -d alsa -d hw:rockchiprk3308a -r 44100 -p 256
```

l'opzione -p è 1024, verificare che il valore 256 non crei troppi dropouts. In quel caso incrementare il valore (che deve essere una potenza di 2).

---

## Jack in realtime priority

l'opzione -R del comando precedente tendenzialmente non funziona e restituisce il seguente errore:

```bash
Cannot use real-time scheduling (RR/95)(1: Operation not permitted)
```

Per abilitare la priorità realtime è necessario compiere i seguenti passi:

1. Nella directory `/etc/security/limits.d/` verificare la presenza del file `audio.conf.disabled`. Se presente copiare il file con il nome `audio.conf`, tramite il seguente comando:

```bash
sudo cp audio.conf.disabled audio.conf
```

2. Nel file `audio.conf` verificare la presenza delle seguenti righe

```bash
@audio - rtprio 95
@audio - memlock unlimited
```

3. Verificare l'esistenza del gruppo `audio` nel sistema con il comando `groups`. Se il grouppo non esiste, crearlo tramite il seguente comando:

```bash
sudo groupadd audio
```

4. Aggiungere l'utente al gruppo `audio`:

```bash
sudo usermod -a -G audio yourUserID
```

dove yourUserID sarà presumibilmente `rock`

5. Uscire dal sistema e riloggarsi con `ssh`

---

## Installazione di supercollider

```bash
sudo apt install supercollider
```

### uso di un display virtuale con xvfb

#### installazione di xvfb

```bash
sudo apt update
sudo apt install [-y] xvfb --fix-missing
```

#### avvio e test di sclang con xvfb

```bash
xvfb-run --auto-servernum /usr/bin/sclang ["$ @"]
```

A questo punto si apre un terminale interattivo `sclang`. Avviare il server:

```bash
s.boot
```

Create una funzione:

```bash
{SinOsc.ar([440,442],0,0.5)}.play
```

Spegnere il server e uscire dalla sessione interattiva:

```bash
s.quit
0.quit
```

### Esecuzione di uno script sc

Per eseguire uno script sc è succifiente eseguire il seguente comando:

```bash
xvfb-run --auto-servernum /usr/bin/sclang file.scd
```

All'interno del file è bene incapsulare come primo argomento del metodo `.waitForBoot()`.

Esempio:

1. con `vim` creare il file `test.scd` e scrivere al suo interno le seguenti righe:

```bash
Server.default.waitForBoot({
{SinOsc.ar([440,442],0,0.5)}.play
})
```

2. salvare e uscire con `:wq`
3. eseguire il seguente comando per avviare `sclang` con `scsynth` e `jackd`:

```bash
xvfb-run --auto-servernum sclang test.scd
```

---

## GPIO

La rockpis è dotata di due gruppi da 26 pin; il primo (`header 1`) è multipurpose, mentre il seconde (`header 2`) è dedicato soprattutto all'interfaccia audio. Fate riferimento a questo [schema](https://wiki.radxa.com/RockpiS/hardware/gpio) per verificare le diverse funzioni dei pin, che differiscono leggermente fra le vbersioni della scheda.

### la libreria mraa

A partire dal 20 febbraio 2020 il GPIO della rockpis può essere programmato tramite la libreria `mraa`. Per l'installazione del pacchetto `libmraa` fare riferimento a [questo documento](https://wiki.radxa.com/RockpiS/dev/libmraa).
L'installazione della libreria fornisce alcuni utili comandi da terminale, ad esempio il programma `mraa-gpio` permette di impostare lo stato dei pin o di ottenere informazione sullo stesso:

```bash
mraa-gpio list #resistuisce la lista dei pin e la loro funzione
mraa-gpio get <numero_pin> # fornisce lo stato di un pin
mraa-gpio set <numero_pin> <livello> # imposta il valore di un pin a 0 o 1
```

### programmazione del gpio

Purtroppo programmare l'interfaccia del rockpis non è semplicissimo. Non c'è un IDE comodo come arduino, quindi bisogna scrivere dei programmi in c, compilarli, ed eseguirli. Esiste un wrapper in python della libreria `mraa`, che, allo steto attuale, non funziona ancora sul rockpis. Sto scrivendo alcuni [external](https://github.com/franeum/rockpis_for_audio/tree/master/blinky) per `pd` per controllare i pin direttamente da puredata, ma il lavoro è appena iniziato.

### opzione python (non ancora funzionante)

Sclang ha bisogno di un display per funzionare, quindi data l'assenza dello stesso in rockpis, bisogna cercare un ambiente alternativo per guidare Scsynth. L'opzione è python e il modulo `supercollider`. Successivamente all'installazione del software si può inviare al rockpis il file con le `synthdefs`, avviare il motore audio (Scsynth) e iniziare a guidarlo da python.

#### installazione del software necessario

Eseguire questi comandi per installare il software:

```python
sudo apt install supercollider
sudo apt install liblo7 liblo-dev
pip3 install Cython
pip3 install pyliblo
pip3 install supercollider
```

_N.B. l'installazione di Cython potrebbe richiedere molto tempo_

# TODO:

1. preparare delle synthDefs e inviarle a `~/.local/share/SuperCollider/synthdefs`
2. installare in python il modulo `supercollider`
3. avviare il server(`scsynth`)
4. testare

---

## Installare nodejs

installare `curl` con il seguente comando:

```bash
sudo apt install curl
```

e poi:

```bash
curl -sL https://deb.nodesource.com/setup_12.x | sudo bash -
sudo apt install -y nodejs
```

---

## Configurare il `bluetooth`

### Rendere il rockpis _discoverable_

usare il programma `hciconfig` per configurare il bluetooth

Verificare la presenza del dispositivo:

```bash
hciconfig -a
```

La risposta sarà qualcosa del genere:

```
hci0:   Type: Primary  Bus: USB
        BD Address: D0:C6:37:8E:2E:01  ACL MTU: 1021:4  SCO MTU: 96:6
        DOWN
        RX bytes:34379 acl:69 sco:0 events:1286 errors:0
        TX bytes:13848 acl:73 sco:0 commands:724 errors:0
        Features: 0xff 0xfe 0x0f 0xfe 0xdb 0xff 0x7b 0x87
        Packet type: DM1 DM3 DM5 DH1 DH3 DH5 HV1 HV2 HV3
        Link policy: RSWITCH HOLD SNIFF
        Link mode: SLAVE ACCEPT
```

`hci0` è l'identificativo del dispositivo bluetooth.

attivare il dispositivo, attribuirgli un nome, attivare l'`ssp` (Secure Simple Pairing) e attivare l'inquiry scan e il page scan:

```bash
sudo hciconfig hci0 up
sudo hciconfig hci0 name rockpis-test
sudo hciconfig hci0 noscan
sudo hciconfig hci0 sspmode 1
sudo hciconfig hci0 piscan
```

verificare le opzioni:

```bash
sudo hciconfig hci0 -a
```

Ora il rockpis dovrebbe essere visibile da un altro dispositivo, con il nome di `rockpis-test`

Provare i seguenti comando dopo aver eseguito da terminale `sudo bluetoothctl`:

```
discoverable on
pairable on
agent NoInputNoOutput
default-agent (?)
```

---

## Access Point

Installazione dei pacchetti:

```bash
sudo apt install hostapd dnsmasq dhcpcd5
```

Interruzione dei servizi interessati:

```bash
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

Nel file `/etc/dhcpcd.conf` inserire queste righe:

```bash
nohook wpa_supplicant
interface wlan0
static ip_address=192.168.72.1/24
static routers=192.168.72.2
```

Riavviare il demone `dhcpdc`:

```bash
sudo service dhcpcd restart
```

Nel file `/etc/dnsmasq.conf` inserire queste righe:

```bash
interface=wlan0      # Use the require wireless interface - usually wlan0
  dhcp-range=192.168.72 .3,192.168.72.20,255.255.255.0,24h
```

Creare (se non esiste) il file `/etc/hostapd/hostapd.conf` con il seguente contenuto:

```bash
interface=wlan0
driver=nl80211
ssid=nome_di_prova
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
rsn_pairwise=CCMP
wpa=2
wpa_passphrase=passowrd_di_prova
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
```

**N.B. commentare le ultime 4 righe se si vuole una rete aperta (senza autenticazione)**

Inserire in `/etc/default/hostapd`:

```bash
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Avviare i servizi:

```bash
sudo service hostapd start
sudo service dnsmasq start
```

Se il servizio `hostapd` viene identificato come `masked`:

```bash
sudo systemctl unmask hostapd
sudo service hostapd start
```

Installare `iptables`:

```bash
sudo apt install iptables
```

In `/etc/sysctl.conf` decommentare la riga

```bash
net.ipv4.ip_forward=1
```

Eseguire:

```bash
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

In `/etc/rc.local` aggiungere la seguente linea prima di `exit 0`:

```
iptables-restore < /etc/iptables.ipv4.nat
```

Verificare che il file `/etc/NetworkManager/NetworkManager.conf` si presenti in questo modo:

```yaml
[main]
plugins=ifupdown,keyfile,ofono
dns=dnsmasq

[ifupdown]
managed=false
```

Riavviare

Ora si dovrebbe _vedere_ la rete creata nella rock. Se dovessero esserci problemi di disconnessione dopo qualche minuto si può provare a disabilitare un eventuale servizio `wpa_supplicant` attivo con:

```bash
ps aux | grep wpa_supplicant
kill -9 <pid>
```

Una volta configurato l'`access point`, si può attivare o disattivare con il comando:

```bash
sudo service hostapd {start|stop}
```

---

## Disabilitare permanentemente il servizio `hostapd` (access point)

semplicemente:

```bash
sudo systemctl disable hostapd
```

---

## Abilitare permanentemente il il servizio `hostapd` (access point)

Per abilitare il servizio all'avvio:

```bash
sudo systemctl enable hostapd
```

Per avviarlo:

```bash
sudo systemctl start hostapd
```


### Aggiornamento attivazione access point (2020-12-17)

Per (ri)attivare l'`access point` dopo una disconnessione dal wifi eseguire il comando:

comando intero:

```bash
sudo -i -- sh -c 'systemctl stop dnsmasq && \
systemctl stop hostapd && \
systemctl restart dhcpcd && \
systemctl start dnsmasq && \
systemctl start hostapd'
```

---

## Preparazione per la connessione automatica alla propria LAN

Accendere il dispositivo wifi del rockpis e connettersi alla rete:

```bash
nmcli r wifi on
nmcli d wifi list # opzionale? potrebbe servire per attivare lo scanning della rete wifi
sudo nmcli d wifi connect <my_net> password <password> ifname p2p0
```

Disconnettersi dalla rete:

```bash
sudo nmcli con down id UUID-RETE
```

---

## Disabilitare la connessione automatica alla rete wifi

```bash
sudo nmcli con modify "$PROFILE" autoconnect no
```

### Annotazione importante su `hostapd` e `dhcpcd`

Se i due servizi sono abilitati ad ogni avvio del `rockpis`, potrebbero non permettere la connessione alla rete wifi, quindi è una buona idea eliminare i file relativi dalla directory `/etc/rcX.d`, dove `X` corrisponde al valore indicato dal comando `sudo runlevel`.

I file relativi sono i seguenti:

```bash
K01dhcpcd -> ../init.d/dhcpcd
S01hostapd -> ../init.d/hostapd
S01dnsmasq -> ../init.d/dnsmasq
```

eliminarli semplicemente con il comando `rm`:

```bash
sudo rm K01dhcpcd && sudo rm S01hostapd && sudo rm S01dnsmasq
```

**Se il wififi non si connette automaticamente, eliminare il MAC address della scheda wifi tramite il comando `sudo nmtui`**

### Creazione di un dns server per il web server locale

Aggiungere le seguenti righe alla fine del file `/etc/dnsmasq.conf`:

```bash
server=8.8.8.8
no-resolv
no-poll
no-hosts
addn-hosts=/etc/hosts-dns
expand-hosts
domain=rockpis.net
log-queries
clear-on-reload
```

Di fatto, eliminando tutte le righe commentate, il file `dnsmasq.conf` dopo la configurazione per `dhcpcd` e per il `dns` avrà le seguenti direttive:

```bash
server=8.8.8.8
no-resolv
no-poll
no-hosts
addn-hosts=/etc/hosts-dns
expand-hosts
domain=rockpis.net
log-queries
clear-on-reload

interface=wlan0
domain-needed
bogus-priv
dhcp-range=192.168.72.3,192.168.72.20,255.255.255.0,24h
```

Creare il file `/etc/hosts-dns` e al suo interno inserire le seguenti righe:

```bash
192.168.72.1 myrock.rockpis.net
```

Quet'ultimo file contiene la risoluzione dei nomi, cioè la tabella di conversione nome -> ip. In questo caso un unico record.

---

## Led wifi

Per configurare un led è necessario creare un servizio apposito nella folder `/usr/lib/systemd/system`. Il servizio sarà definito in un file con estensione `.service`. In questo caso creiamo il file `ledwifi.service`. Il contenuto del file è il seguente:\

```bash
TODO
```
