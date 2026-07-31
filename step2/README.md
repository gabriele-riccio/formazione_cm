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


![prima parte terminale](step2_track3/Screenshot%202026-07-31%20alle%2015.10.56.png)
![prima parte terminale](step2_track3/Screenshot%202026-07-31%20alle%2015.11.39.png)


---

# Spiegazione del codice

## Dockerfile.j2 e scelte

Jinja2 è il motore di templating di Ansible e un file `.j2` non è un Dockerfile "vero e proprio": è un **modello** con dei buchi da riempire. 
Contiene testo normale più due tipi di marcatori:

- `{{ ... }}` → **sostituzione di variabile**: viene rimpiazzato con un valore.
  `{{ item.base_image }}` diventa `ubuntu:22.04`.
- `{% ... %}` → **logica**: `if`/`elif`/`for`, che decidono *quali pezzi* di
  testo produrre.

Quando il playbook esegue il modulo `template`, Jinja2 **renderizza** il `.j2` cioè risolve variabili e logica e scrive il risultato finale come Dockerfile pulito.


L'ho scritto in `.j2` invece che come Dockerfile fisso perché in questo modo **un solo modello
genera due Dockerfile diversi** e mi servono due Dockerfile diversi per i due container con OS differenti.
Ad esempio il ramo `{% if item.family == 'debian' %}` produce le istruzioni `apt` per Ubuntu ed il ramo `{% elif item.family == 'rhel' %}`
produce quelle `dnf` per Rocky.

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

Le variabili del template (`item.base_image`, `item.family`, `item.sudo_group`,
`ssh_user`, `pubkey_name`) sono riempite dal playbook per generare il Dockerfile per l'immagine Ubuntu e quella Rocky. 

## Perché un file drop-in invece dei `sed`

Nella traccia c'era un consiglio per modificare `/etc/ssh/sshd_config` con dei `sed`, tipo
`sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config`.
Funziona, ma ha **tre debolezze** che diventano evidenti proprio quando si
lavora su due OS diversi:

1. **Il `sed` cerca una stringa esatta che potrebbe non esistere.** Esso sostituisce
   *solo* se nel file c'è letteralmente quella riga. Ma il `sshd_config` di
   default **non è identico** tra Ubuntu e Rocky (cambiano commenti, valori,
   ordine). Se la riga è scritta diversamente o assente, il `sed` non trova
   nulla, **non dà errore**, e la direttiva non viene applicata.
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
più in basso. Quindi invece di *modificare* il file esistente,  ne **aggiungo
uno proprio** in `/etc/ssh/sshd_config.d/00-formazione.conf` in modo che esso possa funzionare per entrambe le distro e di non toccare nessun file di sistema.

## Il playbook `build-containers.yml`
Con questo playbook ho buildato le due immagini per i due tipi di container con OS differenti:

### Intestazione del play

```yaml
- name: Build dei container SSH (Ubuntu e Rocky)
  hosts: localhost
  connection: local
  gather_facts: false
```

Il playbook non gestisce macchine remote, sto costruendo immagini **in locale**
parlando con il demone Docker della macchina. Per questo `hosts: localhost` e
`connection: local` (Ansible esegue in locale, senza aprire una connessione SSH
verso se stesso). `gather_facts: false` salta la raccolta di informazioni
sull'host: non servono in questo caso e velocizzo l'esecuzione.

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
Ho descritto ogni elemento (container) con i suoi attributi (nome, immagine base, famiglia di distro, gruppo sudo), i quali valori il `Dockerfile.j2` legge come `item.*`. 

### I quattro task e il meccanismo del `loop`

Tutti e quattro i task condividono lo stesso schema: `loop: "{{ containers }}"`
li fa girare **una volta per container**, e a ogni giro la variabile `item`
diventa l'elemento corrente della lista. Il `loop_control: label: "{{ item.name }}"`
serve solo a rendere leggibile l'output (stampa `formazione-ssh-ubuntu` invece
dell'intero dizionario).

**Task 1 — creo la cartella di contesto** (`ansible.builtin.file`). 

**Task 2 — genero il Dockerfile dal template** (`ansible.builtin.template`). 
**Task 3 — copio la chiave pubblica nel contesto** (`ansible.builtin.copy`). 
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

È il task con cui costruisco davvero l'immagine:
- con `source: build` dico al modulo di *costruire* l'immagine (non di scaricarla da un registry);
- `build.path` è il contesto creato ai task 1–3;
- con `build.pull: true` scarico l'ultima versione dell'immagine base;
- con `force_source: true` forzo il rebuild anche se un'immagine con quel nome esiste già;
- `state: present` è il risultato desiderato.

### Il flusso completo

Per ciascun container il playbook: crea la cartella di contesto, renderizza il
Dockerfile dal template, copia la chiave pubblica accanto e lancia la build.


