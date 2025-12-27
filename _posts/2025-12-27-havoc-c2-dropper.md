---
title: "Dal C al C2: costruire un dropper per un laboratorio di Red Teaming"
date: 2025-12-27 00:00:00 +0800
categories: [Red Teamng, C2, hacking]
tags: [Red Teaming, C2, Hacking]
--- 

#### Intro 
Oggi andremo a vedere  qualcosa di diverso, ultimamente mi sono imbattuto su parecchi argomenti riguardanti il red teaming, in particolare i server C2 (Command and control)
In aggiunta, ho deciso di voler imparare bene C per poter fare del reverse engineering una del mie principali armi.

#### Progetto
Ho voluto approfittare del fatto che stessi imparando il linguaggio C per poter creare qualcosa di mio, un dropper, che evade sia AV che IDS/IPS che una volta nel sistema della vittima, scaricherà un demone generato da Havoc C2, per far si che il tutto fosse il più anonimo possibile e proteggero l'OPSec , ho inoltre offuscato il codice con un paio di tecniche particolari, risultando non vulnerabile a virustotal e difficile da leggere una volta compilato e non.


#### Strumenti
Per creare l'ambiente, ho usato:
1. Oracle Free tier VPS ( Macchina di command and control)
2. Havoc C2 Framework
3. C per la creazione del dropper

### Work flow
![[Pasted image 20251226111110.png]]
L'attacco non sfrutta una vulnerabilità nel sistema, ben si sfrutta un attacco a catena ( Phishing ad esempio).
1. La vittima viene ingannata a scaricare il file, ad esempio dicendo che è un gioco crackato (molto semplice)
2. Una volta nel sistema, l'AV non dovrebbe rilevare il file come Malware
3. il Portable Executable , una volta eseguito, installerà in una cartella scrivibile il demone generato da Havoc-C2
4. Il demone verrà eseguito e il client vittima farà parte del del nostro server C2

### Inizio
Ho iniziato creando una VPS su oracle
https://www.oracle.com/it/cloud/sign-in.html

Passi importanti da seguire sono:
1. Creazione di una VNIC  per poter raggiungere dall'esterno la macchina
2. Disabilitazione delle misure di sicurezza restrittive O aggiunta di porte esposte (Ovvero quella che useremo per connetterci al TeamServer)


Dopo aver ottenuto l'istanza, ho quindi creato e modificato la VNIC
![[Pasted image 20251227170703.png]]

Virtual cloud networks --> Subnets --> la tua VNIC  --> Security --> Security list --> security rules

![[Pasted image 20251227170740.png]]
Ho impostato la porta 4000 per le connessioni inbound e ICMP raggiungibile per una questione di debug.

>Un consiglio che do , non mantenere la porta ICMP con reply to all, bensì cambiarla con type code : 3, a quel punto quando la macchina verrà pingata, verrà contrassegnata come irraggiungibile, ottimo per non essere rilevato dalla maggior parte dei bot sulla rete

![[Pasted image 20251227170952.png]]
![[Pasted image 20251227171003.png]]
https://www.iana.org/assignments/icmp-parameters/icmp-parameters.xhtml



### Configurazione server C2

Una volta sul server, ho installato e configurato  Havoc team server

```bash
git clone https://github.com/HavocFramework/Havoc.git
cd Havoc

sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.10 python3.10-dev

apt-get install golang-go
cd teamserver
go mod download golang.org/x/sys
go mod download github.com/ugorji/go
cd ..

make ts-build


```

Prima di avviarlo dobbiamo sistemare il nostro config file 

`nano /Havoc/profiles/havoc.yaotl`

![[Pasted image 20251227171645.png]]
Qui possiamo creare user, configurare password e la porta che verrà esposta sulla rete.


Avviamo il team server con 

```bash
sudo ./havoc server --profile ./profiles/havoc.yaotl -v --debug
```


#### Configurazione client

Vi sono due modi, compilare tutto o installare da repository, la via più veloce:
`yay -S havoc-c2-git`

Adesso siamo pronti per avviare il nostro client e verificare che il tutto funzioni.

Qui ho però apportato una modifica, dato che il VPS è comunque molto restrittivo, per evitare falle di sicurezza, ho deciso di fare un forwarding della porta 4000 sulla mia macchina, così da poter raggiungere il team server dirittamente in localhost.

```
❯ ssh -i ssh.key ubuntu@server -L 4000:127.0.0.1:4000
```

Con un breve network scan  sul nostro sistema, vediamo che abbiamo la porta aperta. proviamo a raggiungerla con il client havoc
![[Pasted image 20251226110242.png]]

Creiamo un profilo e inseriamo tutti i dati
![[Pasted image 20251226110334.png]]

![[Pasted image 20251226110350.png]]

Abbiamo con successo ottenuto l'accesso al team server.

### Generazione del demone
Dobbiamo generare un demone, ovvero l'agente che verrà scaricato nel sistema vittima.

Impostiamo il jitter:

In questo tipo di post exploitation attack, rilevare molte connessioni può dare nell'occhio, settando un jitter di 15 secondi, 
faremo si che vi sia una variazione casuale aggiunta all’intervallo di tempo con cui l'agent (beacon) contatta il server C2.

Esempio:
- **Senza jitter** → beacon ogni **60 secondi** precisi
- **Con jitter 30%** → beacon tra **42 e 78 secondi**

Inoltre è ottimo per evasione da IDS / SIEM

Molti sistemi di detection cercano pattern come:
- traffico periodico 
- intervalli regolari
- stesso endpoint a tempo fisso

Il jitter **rompe la periodicità**, rendendo il beaconing simile a traffico legittimo.

Prima di tutto

## Creiamo un listener.
![[Pasted image 20251227172617.png]]

### Generiamo un payload:

In questo caso,potremmo generare un payload solo per windows, motivo per la quale il testing potrebbe avvenire solo su una macchina windows, ma il nostro obbiettivo non è questo, è iniettare l'agente , facendo si che venga eseguito tramite un dropper a provare di reverse engineering,
![[Pasted image 20251227172937.png]]

## Creazione del progamma 
Inizialmente ho scritto qualche riga di codice per vedere cosa si vedesse con hexdump o strings
![[Pasted image 20251227173200.png]]
Molto semplice:

Insieme a tanti pezzi di codice, che serviranno a rendere l'applicazione compilata ancora più difficile da leggere, grabberà il nostro agent.
Ma sorge un problema.
Se proviamo a fare un po' di analisi sul PE generato, notiamo delle cose CRITICHE.


![[Pasted image 20251227173235.png]]

IP esposto --> OpSec fallito

Quindi, dato che noi ci teniamo a mantenere un buon anonimato, scriviamo il codce in modo tale che sia il meno leggibile possibile.

### Creazione e offuscamento del codice: ELF
![[Pasted image 20251227123715.png]]

1. Verrà effettuato un piccolo health check del server e successivamente verrà scaricato il file.
![[Pasted image 20251227123907.png]]
Il tutto sembra andare

Ora dobbiamo trovare una cartella scrivibile ,
Su Linux:
`~/.local/bin/shared/clib/`
Su Windows
`C:\Users\<user>\AppData\Local\`

![[Pasted image 20251227130348.png]]

Con questa modifica:
- Il codice verrà inserito in una directory scrivibile
Adesso bisognerà eseguirlo e avere un PoC (Proof of Concept) del risultato

![[Pasted image 20251227174144.png]]
![[Pasted image 20251227174251.png]]

Come PoC abbiamo fatto si che ricevessimo una notifica una voolta che l'agente sarebbe stato eseguito.

Adesso: Purtroppo non possiamo testare l'agent su linux, in quanto ancora non è supportato al 100%, per tanto, scriveremo del codice per farlo esegure su windows:

Quello finale di Linux è il seguente:
![[Pasted image 20251227174531.png]]

### Scrittura codice Windows: PE 

- Rimpiazziamo il path linux con un path adatto a windows
- Al posto di wget useremo il nostro amato certutil
- Modifichiamo il ping per poter essere eseguito da wndows

![[Pasted image 20251227174519.png]]

Adesso sembrerebbe finito, ma manca il punto principale!
L'offuscamento.
![[Pasted image 20251227174810.png]]

## Offuscamento

Per offuscarlo, ho deciso di darmi un po' all'ingegno, il tutto è assolutamente reversibile, ma non come prima, l'ip non comparirà tra le stringhe e il tutto sarà più sicuro.

Per faro ho deciso di fare così:
- Ho scritto l'IP in hex, quando il progamma dovrà richiamare l'agent, manderà una richiesta alla stringa decodificata, ma questo non sarà visibile da string o hexdump

![[Pasted image 20251227155316.png]]
![[Pasted image 20251227175044.png]]

In fine , ho fatto si che ogni singola funzione, testo per debug e tutto ciò che si potesse offuscare al meglio, venisse reso completamente illegibile.


![[Pasted image 20251227175213.png]]

## Risultato:
1. Bypass di AV
2. Inserimento di macchina target nella botnet
3. Offuscamento del codice

![[Pasted image 20251227175837.png]]
