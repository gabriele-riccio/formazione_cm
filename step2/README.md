# Step 2 — Creare Build di container

Playbook Ansible che effettua la build di due container con **OS differenti**
(Ubuntu e Rocky Linux), ciascuno configurato in modo che abbiano queste caratteristiche:
- Essere sempre in ascolto sulla porta 22 del container
- Avere attivo il servizio ssh
- Avere un utente abilitato a collegarsi tramite ssh key e poter fare sudo


## Requisiti soddisfatti

Ogni container prodotto:

- è in ascolto sulla **porta 22** (SSH esposta e mappata sull'host);
- ha il **servizio sshd attivo** come processo principale (`sshd -D`);
- ha l'utente **`gabriele`**, abilitato al login via **chiave SSH** e con
  permessi **sudo** (root senza password).

## Struttura

```
step2/
├── build-containers.yml        # playbook principale
├── templates/
│   └── Dockerfile.j2           # Dockerfile parametrico per OS
├── keys/                       # coppia di chiavi SSH 
│   └── id_rsa_gabriele.pub     # `id_rsa_gabriele` è la privata, ed è nel git-ignored
└── build/                      # contesti di build generati (git-ignored)
```

## Container prodotti

| Immagine                | Base image     | Package manager | Gruppo sudo |
|-------------------------|----------------|-----------------|-------------|
| formazione-ssh-ubuntu   | ubuntu:22.04   | apt             | sudo        |
| formazione-ssh-rocky    | rockylinux:9   | dnf             | wheel       |

Le differenze tra le distro sono gestite con Jinja2 nel template
(`{% if item.family == 'debian' %}` vs `rhel`), così un unico Dockerfile
genera entrambe le immagini.

## Prerequisiti

- Docker Desktop avviato
- Ansible con la collection `community.docker` e l'SDK Docker per Python

## Svolgimento:

1. Genero la coppia di chiavi (una sola volta):

   ```bash
   ssh-keygen -t rsa -b 4096 -C "gabriele@formazione-cm" \
     -f keys/id_rsa_gabriele -N ""
   ```

2. Scrivo il file `Dockerfile.j2` nella cartella `templates` e il playbook `build-containers.yml`  per buildare le immagini (spiegazione sotto):

3. Lancio il playbook per buildare le immagini:

   ```bash
   ansible-playbook build-containers.yml
   ```

4. Avvio i container (con porte host diverse (`2222` e `2223`) per farli coesistere):

   ```bash
   docker run -d --name ssh-ubuntu -p 2222:22 formazione-ssh-ubuntu
   docker run -d --name ssh-rocky  -p 2223:22 formazione-ssh-rocky
   ```

5. Verifica login via chiave ssh e sudo:

   ```bash
   ssh -i keys/id_rsa_gabriele -p 2222 gabriele@localhost 'whoami && sudo whoami'
   ssh -i keys/id_rsa_gabriele -p 2223 gabriele@localhost 'whoami && sudo whoami'
   ```

## Output:

---

# Spiegazione del codice

Questa sezione spiega i due file del progetto (`Dockerfile.j2` e
`build-containers.yml`) e le scelte tecniche dietro di essi.

## Cos'è un template Jinja2 (`.j2`)

**Jinja2** è il motore di *templating* che Ansible usa sotto il cofano. Un file
`.j2` non è un Dockerfile "vero e proprio": è un **modello** con dei buchi da
riempire. Contiene testo normale più due tipi di marcatori:

- `{{ ... }}` → **sostituzione di variabile**: viene rimpiazzato con un valore.
  `{{ item.base_image }}` diventa `ubuntu:22.04`.
- `{% ... %}` → **logica**: `if`/`elif`/`for`, che decidono *quali pezzi* di
  testo produrre.

Quando il playbook esegue il modulo `template`, Jinja2 **renderizza** il `.j2`
— cioè risolve variabili e logica — e scrive il risultato finale come Dockerfile
normale in `build/<nome>/Dockerfile`. Docker riceve un Dockerfile pulito, senza
più graffe: la parte Jinja2 è già stata "cotta" da Ansible prima.

L'ho scritto in `.j2` invece che come Dockerfile fisso perché **un solo modello
genera due Dockerfile diversi**. Il ramo `{% if item.family == 'debian' %}`
produce le istruzioni `apt` per Ubuntu, il ramo `{% elif item.family == 'rhel' %}`
produce quelle `dnf` per Rocky. Senza template avrei dovuto scrivere e mantenere
due Dockerfile quasi identici, con il rischio che divergano nel tempo; con Jinja2
la logica "cosa cambia tra le distro" sta in un posto solo. La convenzione `.j2`
è semplicemente il modo standard di segnalare "questo file è un template Ansible".

## Il file `templates/Dockerfile.j2`

```dockerfile
FROM {{ item.base_image }}

# 1. openssh-server + sudo (comando specifico per famiglia di distro)
{% if item.family == 'debian' %}
RUN apt-get update && \
    apt-get install -y --no-install-recommends openssh-server sudo && \
    rm -rf /var/lib/apt/lists/*
{% elif item.family == 'rhel' %}
RUN dnf install -y openssh-server sudo && \
    dnf clean all
{% endif %}

# 2. Utente + sudo SENZA password (la password è disabilitata, quindi serve NOPASSWD)
RUN useradd -m -s /bin/bash {{ ssh_user }} && \
    usermod -aG {{ item.sudo_group }} {{ ssh_user }} && \
    echo '{{ ssh_user }} ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/{{ ssh_user }} && \
    chmod 440 /etc/sudoers.d/{{ ssh_user }}

# 3. Chiave pubblica -> authorized_keys (permessi rigorosi o sshd rifiuta)
RUN mkdir -p /home/{{ ssh_user }}/.ssh && chmod 700 /home/{{ ssh_user }}/.ssh
COPY {{ pubkey_name }} /home/{{ ssh_user }}/.ssh/authorized_keys
RUN chmod 600 /home/{{ ssh_user }}/.ssh/authorized_keys && \
    chown -R {{ ssh_user }}:{{ ssh_user }} /home/{{ ssh_user }}/.ssh

# 4. Hardening SSH via drop-in (Include già attivo su Ubuntu e Rocky)
RUN mkdir -p /etc/ssh/sshd_config.d && \
    printf '%s\n' \
      'PermitRootLogin no' \
      'PasswordAuthentication no' \
      'PubkeyAuthentication yes' \
      'AllowUsers {{ ssh_user }}' \
      > /etc/ssh/sshd_config.d/00-formazione.conf

# 5. sshd: privilege separation dir + host keys
RUN mkdir -p /run/sshd && ssh-keygen -A

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
```

**`FROM {{ item.base_image }}`** — l'immagine di partenza non è fissa:
`item.base_image` viene sostituito con `ubuntu:22.04` o `rockylinux:9` a seconda
del container che si sta buildando.

**Blocco 1 — installazione di `openssh-server` e `sudo`.** La logica cambia in
base alla famiglia della distro, gestita con un `if/elif` Jinja2: se
`item.family == 'debian'` si usa `apt-get` (con `rm -rf /var/lib/apt/lists/*`
finale per cancellare la cache dei pacchetti e alleggerire l'immagine); se
`item.family == 'rhel'` si usa `dnf` (con `dnf clean all` per lo stesso motivo).
Solo uno dei due rami finisce nel Dockerfile generato.

**Blocco 2 — utente e sudo senza password.** `useradd -m -s /bin/bash` crea
l'utente con home e shell bash; `usermod -aG {{ item.sudo_group }}` lo aggiunge
al gruppo di amministrazione (`sudo` su Ubuntu, `wheel` su Rocky — parametrico);
infine si scrive un file in `/etc/sudoers.d/` con la regola `NOPASSWD:ALL`. Il
`chmod 440` è obbligatorio: sudo **rifiuta** i file di configurazione con
permessi troppo aperti.

**Blocco 3 — chiave pubblica in `authorized_keys`.** Si crea la cartella `.ssh`
(permessi `700`), si copia dentro la chiave pubblica dal contesto di build
rinominandola in `authorized_keys`, e le si assegnano permessi `600` più il
proprietario corretto. Questi permessi rigorosi non sono un vezzo: se `.ssh` o
`authorized_keys` sono troppo aperti, sshd ignora la chiave e il login fallisce
silenziosamente.

**Blocco 4 — hardening di SSH via file drop-in.** Vedi la sezione dedicata qui
sotto sul perché di questa scelta. Le direttive impostate: `PermitRootLogin no`,
`PasswordAuthentication no`, `PubkeyAuthentication yes`, `AllowUsers {{ ssh_user }}`.

**Blocco 5 — preparazione di sshd.** `mkdir -p /run/sshd` crea la directory di
*privilege separation* richiesta da sshd, e `ssh-keygen -A` genera le **host
key** del server (una per algoritmo). Senza queste due cose sshd non parte.

**`EXPOSE 22`** dichiara che il servizio ascolta sulla porta 22, e
**`CMD ["/usr/sbin/sshd", "-D", "-e"]`** avvia sshd come processo principale del
container: `-D` lo tiene in foreground (così il container resta vivo e "sempre
in ascolto"), `-e` manda i log su stderr, dove `docker logs` li può leggere.

Le variabili del template (`item.base_image`, `item.family`, `item.sudo_group`,
`ssh_user`, `pubkey_name`) sono riempite dal playbook. La `COPY` prende la chiave
pubblica dal contesto di build, quindi il playbook la copia in `build/<nome>/`
accanto al Dockerfile prima di lanciare la build.

## Perché un file drop-in invece dei `sed`

Il *tip* della traccia modifica `/etc/ssh/sshd_config` con dei `sed`, tipo
`sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config`.
Funziona, ma ha **tre debolezze** che diventano evidenti proprio quando si
lavora su due OS diversi:

1. **Il `sed` cerca una stringa esatta che potrebbe non esistere.** Sostituisce
   *solo* se nel file c'è letteralmente quella riga. Ma il `sshd_config` di
   default **non è identico** tra Ubuntu e Rocky (cambiano commenti, valori,
   ordine). Se la riga è scritta diversamente o assente, il `sed` non trova
   nulla, **non dà errore**, e la direttiva non viene applicata: un fallimento
   silenzioso, il peggior tipo di bug.
2. **In `sshd_config` vince la *prima* occorrenza, non l'ultima.** È una regola
   di come sshd legge il config: per la maggior parte delle direttive, se un
   parametro compare più volte, conta il **primo** valore. Quindi accodare le
   proprie righe in fondo al file (`echo ... >>`) spesso **non ha effetto**.
3. **Si modifica un file che appartiene alla distro.** È invasivo: un
   aggiornamento del pacchetto openssh potrebbe sovrascriverlo.

La soluzione **drop-in** risolve tutti e tre i problemi. Sia Ubuntu 22.04 sia
Rocky 9 hanno, in cima al loro `sshd_config`, la riga:

```
Include /etc/ssh/sshd_config.d/*.conf
```

Significa "prima di leggere il resto, includi tutti i file `.conf` in questa
cartella". Poiché l'`Include` è **in cima** e vince la prima occorrenza (punto
2), le direttive messe lì dentro **prevalgono** su qualsiasi valore di default
più in basso. Quindi invece di *modificare* il file esistente, se ne **aggiunge
uno proprio** in `/etc/ssh/sshd_config.d/00-formazione.conf`. Vantaggi:

- **funziona identico su entrambe le distro**, perché non dipende dal testo del
  file di default ma scrive un file nuovo da zero;
- **le direttive vincono di sicuro**, grazie alla posizione dell'`Include` e
  alla regola del primo valore;
- il prefisso **`00-`** garantisce che, in ordine alfabetico, sia letto prima di
  eventuali `.conf` della distro;
- **non tocca file di sistema**: la configurazione è isolata e facile da rimuovere.

## Il playbook `build-containers.yml`

Se il `Dockerfile.j2` descrive *com'è fatta* un'immagine, il playbook descrive
*come costruirla*: è l'orchestratore che prende il template, lo riempie per ogni
OS e lancia la build.

### Intestazione del play

```yaml
- name: Build dei container SSH (Ubuntu e Rocky)
  hosts: localhost
  connection: local
  gather_facts: false
```

Il playbook non gestisce macchine remote: costruisce immagini **in locale**
parlando con il demone Docker della macchina. Per questo `hosts: localhost` e
`connection: local` (Ansible esegue in locale, senza aprire una connessione SSH
verso se stesso). `gather_facts: false` salta la raccolta di informazioni
sull'host: non servono e velocizza l'esecuzione.

### Le variabili

```yaml
  vars:
    ansible_python_interpreter: /usr/local/bin/python3
    ssh_user: gabriele
    pubkey_name: id_rsa_gabriele.pub
    containers:
      - { name: formazione-ssh-ubuntu, base_image: "ubuntu:22.04", family: debian, sudo_group: sudo }
      - { name: formazione-ssh-rocky,  base_image: "rockylinux:9", family: rhel,   sudo_group: wheel }
```

Il cuore del design è la lista **`containers`**: ogni elemento è un container con
i suoi attributi (nome, immagine base, famiglia di distro, gruppo sudo) — gli
stessi valori che il `Dockerfile.j2` legge come `item.*`. Aggiungere un terzo OS
significa aggiungere una riga qui, senza toccare né il template né i task.

`ssh_user` e `pubkey_name` sono variabili globali (valgono per tutti i
container). `ansible_python_interpreter: /usr/local/bin/python3` forza Ansible a
usare l'interprete Python che ha installato l'SDK Docker: senza questa riga
userebbe un Python diverso e il modulo `docker_image` fallirebbe perché non
troverebbe la libreria.

### I quattro task e il meccanismo del `loop`

Tutti e quattro i task condividono lo stesso schema: `loop: "{{ containers }}"`
li fa girare **una volta per container**, e a ogni giro la variabile `item`
diventa l'elemento corrente della lista. Il `loop_control: label: "{{ item.name }}"`
serve solo a rendere leggibile l'output (stampa `formazione-ssh-ubuntu` invece
dell'intero dizionario).

**Task 1 — creo la cartella di contesto** (`ansible.builtin.file`). Crea
`build/formazione-ssh-ubuntu/` e `build/formazione-ssh-rocky/`: il *contesto di
build*, la cartella che Docker userà come radice, e dentro cui devono trovarsi
il Dockerfile e la chiave pubblica.

**Task 2 — genero il Dockerfile dal template** (`ansible.builtin.template`). È
il passo in cui Jinja2 entra in azione: prende `templates/Dockerfile.j2`, lo
renderizza con i valori di `item` (e delle variabili globali) e scrive il
Dockerfile finale — già "cotto", senza più graffe — in `build/<nome>/Dockerfile`.
Per Ubuntu produce il ramo `apt`, per Rocky il ramo `dnf`.

**Task 3 — copio la chiave pubblica nel contesto** (`ansible.builtin.copy`). Il
Dockerfile contiene `COPY {{ pubkey_name }} ...`, e Docker può copiare **solo
file che stanno dentro il contesto di build**. Quindi prima della build si porta
`keys/id_rsa_gabriele.pub` dentro `build/<nome>/`, accanto al Dockerfile. Senza
questo task la build fallirebbe con un "file not found" sulla `COPY`.

**Task 4 — build dell'immagine** (`community.docker.docker_image`).

```yaml
    - name: Build dell'immagine Docker
      community.docker.docker_image:
        name: "{{ item.name }}"
        tag: latest
        source: build
        force_source: true
        build:
          path: "build/{{ item.name }}"
          dockerfile: Dockerfile
          pull: true
        state: present
      loop: "{{ containers }}"
      loop_control:
        label: "{{ item.name }}"
```

È il task che costruisce davvero l'immagine. I parametri chiave: `source: build`
dice al modulo di *costruire* l'immagine (non di scaricarla da un registry);
`build.path` è il contesto creato ai task 1–3; `build.pull: true` scarica sempre
l'ultima versione dell'immagine base; `force_source: true` forza il rebuild
anche se un'immagine con quel nome esiste già; `state: present` è il risultato
desiderato — l'immagine deve esistere al termine.

> **Nota di sintassi YAML.** `loop` e `loop_control` sono attributi **del task**,
> non del modulo: vanno allineati alla chiave `community.docker.docker_image:`
> (stessa colonna), non ai suoi parametri. Un errore di indentazione qui è la
> causa d'errore più comune del playbook.

### Il flusso completo

Per ciascun container il playbook: crea la cartella di contesto → renderizza il
Dockerfile dal template → copia la chiave pubblica accanto → lancia la build. Un
`ansible-playbook build-containers.yml` produce così entrambe le immagini in un
colpo solo (`ok=4 changed=4`), pronte per essere avviate con `docker run`.

---

## Scelte tecniche (riepilogo)

- **`sshd -D`**: sshd gira in foreground come PID principale, così il container
  resta attivo e "sempre in ascolto".
- **Host key** generate a build time con `ssh-keygen -A` e directory di
  privilege separation `/run/sshd`.
- **Hardening** via drop-in in `/etc/ssh/sshd_config.d/00-formazione.conf`,
  robusto e valido su entrambe le distro grazie all'`Include` già presente nei
  rispettivi `sshd_config`.
- **sudo NOPASSWD**: la password è disabilitata, quindi il sudo dell'utente è
  concesso senza password tramite `/etc/sudoers.d/gabriele`.

## Sicurezza

La **chiave privata non è versionata** (`.gitignore`: `keys/id_*` esclusa,
`*.pub` tracciabile con `!keys/id_*.pub`). Nel repo finisce solo la chiave
pubblica.
