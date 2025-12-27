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

### Workflow
<img width="514" height="232" alt="image" src="https://github.com/user-attachments/assets/2279c663-80c6-4af5-ab38-4be1ca6c2172" />

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
<img width="1502" height="421" alt="image" src="https://github.com/user-attachments/assets/d2c1ab50-22f3-416c-a2da-13039c632a68" />


Virtual cloud networks --> Subnets --> la tua VNIC  --> Security --> Security list --> security rules

<img width="1566" height="409" alt="image" src="https://github.com/user-attachments/assets/80c5c0c1-5003-4464-8ae2-b31e69cdd06a" />

Ho impostato la porta 4000 per le connessioni inbound e ICMP raggiungibile per una questione di debug.

>Un consiglio che do , non mantenere la porta ICMP con reply to all, bensì cambiarla con type code : 3, a quel punto quando la macchina verrà pingata, verrà contrassegnata come irraggiungibile, ottimo per non essere rilevato dalla maggior parte dei bot sulla rete

<img width="1261" height="410" alt="image" src="https://github.com/user-attachments/assets/b1496df8-a6d7-482d-92e8-f80278c41f5a" />
<img width="363" height="272" alt="image" src="https://github.com/user-attachments/assets/13ad6090-4f90-470b-bcf7-2ef9f95b23dc" />

[Info!](https://www.iana.org/assignments/icmp-parameters/icmp-parameters.xhtml)



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

<img width="704" height="316" alt="image" src="https://github.com/user-attachments/assets/2a9ee96b-1757-45c9-bb8e-9bc4a9e1fc84" />
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
<img width="239" height="89" alt="image" src="https://github.com/user-attachments/assets/73b2aadc-8bca-4e3e-bbb5-55b08d3c30e7" />

Creiamo un profilo e inseriamo tutti i dati
<img width="500" height="277" alt="image" src="https://github.com/user-attachments/assets/b19ddaaa-1dcc-4169-a9ea-b052c23f1070" />

<img width="1643" height="826" alt="image" src="https://github.com/user-attachments/assets/a01df206-3b34-45ad-9a2e-475fd55aa8de" />

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
<img width="532" height="786" alt="image" src="https://github.com/user-attachments/assets/69400a72-4157-4fa9-ae61-0f6031000ed2" />

### Generiamo un payload:

In questo caso,potremmo generare un payload solo per windows, motivo per la quale il testing potrebbe avvenire solo su una macchina windows, ma il nostro obbiettivo non è questo, è iniettare l'agente , facendo si che venga eseguito tramite un dropper a provare di reverse engineering,
<img width="549" height="678" alt="image" src="https://github.com/user-attachments/assets/d52a29bb-1a67-40d5-b742-ed5a5ea4edd2" />

## Creazione del progamma 
Inizialmente ho scritto qualche riga di codice per vedere cosa si vedesse con hexdump o strings<br>

<img width="606" height="332" alt="image" src="https://github.com/user-attachments/assets/8d8fcafe-e848-4a9b-8f65-e8d9c2f12c14" />
Molto semplice:

Insieme a tanti pezzi di codice, che serviranno a rendere l'applicazione compilata ancora più difficile da leggere, grabberà il nostro agent.
Ma sorge un problema.
Se proviamo a fare un po' di analisi sul PE generato, notiamo delle cose CRITICHE.


<img width="653" height="316" alt="image" src="https://github.com/user-attachments/assets/2c23f85b-1140-493e-9178-11702202aa2f" />

IP esposto --> OpSec fallito

Quindi, dato che noi ci teniamo a mantenere un buon anonimato, scriviamo il codce in modo tale che sia il meno leggibile possibile.

### Creazione e offuscamento del codice: ELF
<img width="753" height="611" alt="image" src="https://github.com/user-attachments/assets/6e604b19-789d-4417-8f94-f75f73bd7bcb" />

1. Verrà effettuato un piccolo health check del server e successivamente verrà scaricato il file.
<img width="1643" height="158" alt="image" src="https://github.com/user-attachments/assets/84e328ef-9b36-45d8-928a-6bb2fe8664a2" />
Il tutto sembra andare

Ora dobbiamo trovare una cartella scrivibile ,
Su Linux:
`~/.local/bin/shared/clib/`
Su Windows
`C:\Users\<user>\AppData\Local\`

<img width="808" height="362" alt="image" src="https://github.com/user-attachments/assets/0c6f9484-6323-4af7-a325-ecbea06a3be6" />

Con questa modifica:
- Il codice verrà inserito in una directory scrivibile
Adesso bisognerà eseguirlo e avere un PoC (Proof of Concept) del risultato

<img width="641" height="331" alt="image" src="https://github.com/user-attachments/assets/9dbb7d11-b9bd-4f7f-bd53-ce8e26b8fcde" />
<img width="804" height="387" alt="image" src="https://github.com/user-attachments/assets/d8f38807-23c6-4da8-b81d-e4948530b8e1" />

Come PoC abbiamo fatto si che ricevessimo una notifica una voolta che l'agente sarebbe stato eseguito.

Adesso: Purtroppo non possiamo testare l'agent su linux, in quanto ancora non è supportato al 100%, per tanto, scriveremo del codice per farlo esegure su windows:

Quello finale di Linux è il seguente:
<img width="1098" height="846" alt="image" src="https://github.com/user-attachments/assets/3ffb4796-14bf-48ed-a1fb-48dc1ec20ff9" />

### Scrittura codice Windows: PE 

- Rimpiazziamo il path linux con un path adatto a windows
- Al posto di wget useremo il certutil
- Modifichiamo il ping per poter essere eseguito da windows

<img width="994" height="874" alt="image" src="https://github.com/user-attachments/assets/3791ad3d-f58b-46ad-9bba-0aae27e57637" />

Adesso sembrerebbe finito, ma manca il punto principale!
L'offuscamento.
<img width="648" height="348" alt="image" src="https://github.com/user-attachments/assets/695901f9-5123-42ff-a713-bae1beadfeaf" />

## Offuscamento

Per offuscarlo, ho deciso di darmi un po' all'ingegno, il tutto è assolutamente reversibile, ma non come prima, l'ip non comparirà tra le stringhe e il tutto sarà più sicuro.

Per faro ho deciso di fare così:
- Ho scritto l'IP in hex, quando il progamma dovrà richiamare l'agent, manderà una richiesta alla stringa decodificata, ma questo non sarà visibile da string o hexdump

<img width="512" height="491" alt="image" src="https://github.com/user-attachments/assets/a30c8adb-28ab-40a2-938b-6c1427cab60a" />
<img width="315" height="110" alt="image" src="https://github.com/user-attachments/assets/282bba25-8bb1-470d-b256-e632f796ca47" />

In fine , ho fatto si che ogni singola funzione, testo per debug e tutto ciò che si potesse offuscare al meglio, venisse reso completamente illegibile.


<img width="829" height="886" alt="image" src="https://github.com/user-attachments/assets/3cd8781b-4c75-4da1-ad89-861c9e95072a" />

## Risultato:
1. Bypass di AV
2. Inserimento di macchina target nella botnet
3. Offuscamento del codice

<img width="969" height="466" alt="image" src="https://github.com/user-attachments/assets/aac06714-92b5-44b9-aa73-7b432574d19a" />
