# Step 5 — Jenkins & Ansible

## Traccia
Automazione end-to-end di **build → tag progressivo → push su registry → deploy con Ansible**,
orchestrata da una pipeline Jenkins.
- Creare un container che oltre ai requisiti dello Step 2 abbia anche le seguenti caratteristiche:
  - Avere attivo il servizio Docker/Podman;
- Configurare una pipeline Jenkins che:
  - Esegua una build di un'immagine e la tagghi in modo progressivo
  - Faccia il push dell'immagine sul registry
  - Utilizzi Ansibile per eseguire il deploy sul container precedentemente creato 

---

## Architettura

Due container distinti collaborano, più il registry e Jenkins:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Mac (Docker Desktop)                                               │
│                                                                     │
│   ┌──────────┐   docker.sock   ┌────────────────┐                   │
│   │ registry │◄────────────────│    Jenkins     │ (jenkins-ansible) │
│   │  :5000   │                 │  build/push    │                   │
│   └────┬─────┘                 │  + Ansible     │                   │
│        │                       └───────┬────────┘                   │
│        │ pull                          │ ansible-playbook(SSH :2222)│
│        ▼                               ▼                            │
│   ┌───────────────────────────────────────────────┐                 │
│   │  step5-target  (formazione-ssh-dind)          │                 │
│   │  - sshd attivo (requisito Step 2)             │                 │
│   │  - dockerd interno attivo (requisito Step 5)  │                 │
│   │                                               │                 │
│   │      ┌───────────────────────────┐            │                 │
│   │      │ formazione-app-running    │  ← deploy  │                 │
│   │      │ (ubuntu:22.04 + sshd)     │            │                 │
│   │      └───────────────────────────┘            │                 │
│   └───────────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

- **target** (`formazione-ssh-dind`): l'*ambiente* dove l'app gira. È raggiungibile via SSH
  (così Ansible può entrarci) e ha un demone Docker interno (così può eseguire container).
- **app** (`formazione-app`): il *carico* che la pipeline builda, tagga, pusha e che Ansible
  deploya dentro il target.

---

## Struttura dei file

```
step5/
├── Jenkinsfile              # pipeline dichiarativa (4 stage)
├── deploy.yml               # playbook Ansible di deploy
├── inventory.ini            # inventory per esecuzione manuale (da Mac → localhost:2222)
├── inventory-jenkins.ini    # inventory per la pipeline (da Jenkins → host.docker.internal:2222)
├── app/
│   └── Dockerfile           # immagine applicativa (ubuntu:22.04 + sshd)
├── target/
│   ├── Dockerfile           # immagine target (docker:dind + openssh + python/SDK)
│   └── docker-entrypoint.sh # avvia sshd in background, poi dockerd
├── jenkins/
│   └── Dockerfile           # immagine Jenkins custom (Ansible + docker CLI)
└── keys/
    └── id_rsa_step5.pub     # chiave pubblica (la privata è in .gitignore)
```

---

## Svolgimento

## 1. Analisi: il modello a due container

Ci sono da distinguere due container che collaborano:

- **target** — l'*ambiente* dove l'applicazione girerà. Deve essere raggiungibile via SSH
  (così Ansible può entrarci per fare il deploy) e deve avere un **demone Docker attivo** al
  suo interno (Docker-in-Docker), così da poter eseguire container.
- **app** — il *carico* che la pipeline builda, tagga, pusha sul registry e che poi Ansible
  fa girare *dentro* il target.

La pipeline automatizza la catena:

```
Checkout → Build immagine → Tag progressivo → Push su registry → Deploy con Ansible
```

L'intera infrastruttura gira in Docker sul Mac: registry, target, app e lo stesso Jenkins
(come container). Questo semplifica la rete: tutti condividono lo stesso host.

---

## 2. Il container target (Docker-in-Docker + SSH)

### Dockerfile (`target/Dockerfile`)

Base `docker:dind` (che porta già il demone Docker), con l'aggiunta di `openssh` (accesso SSH,
requisito Step 2) e `python3` + SDK Docker (necessari ai moduli `community.docker` che Ansible
esegue *dentro* il target).

```dockerfile
FROM docker:dind

RUN apk add --no-cache openssh bash
RUN apk add --no-cache python3 py3-pip \
 && pip3 install --no-cache-dir --break-system-packages docker

RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh

RUN sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config \
 && sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/'     /etc/ssh/sshd_config \
 && sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/'  /etc/ssh/sshd_config

COPY docker-entrypoint.sh /usr/local/bin/custom-entrypoint.sh
RUN chmod +x /usr/local/bin/custom-entrypoint.sh

EXPOSE 22 2375 2376
ENTRYPOINT ["/usr/local/bin/custom-entrypoint.sh"]
```

### Entrypoint (`target/docker-entrypoint.sh`)

Fa da "direttore d'orchestra" tra i due servizi: rigenera le host key SSH, inietta la chiave
pubblica autorizzata (da variabile d'ambiente), avvia `sshd -D` in **background** e infine
`exec dockerd-entrypoint.sh` — così il demone Docker resta il processo principale (PID 1) e
tiene vivo il container.

```sh
#!/bin/sh
set -e

ssh-keygen -A
mkdir -p /var/run/sshd

mkdir -p /root/.ssh
echo "${AUTHORIZED_KEYS:-}" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

mkdir -p /root/.docker
echo "${DOCKER_CONFIG_JSON:-}" > /root/.docker/config.json

/usr/sbin/sshd -D &

exec dockerd-entrypoint.sh "$@"
```

### Chiave SSH, build e avvio

```bash
# coppia di chiavi (client = Ansible; la pubblica va nel target)
ssh-keygen -t rsa -b 4096 -f keys/id_rsa_step5 -N "" -C "ansible-step5"

# build
docker build -t formazione-ssh-dind:latest ./target

# avvio: --privileged serve al Docker interno; la pubblica va in AUTHORIZED_KEYS
docker run -d --privileged --name step5-target \
  -p 2222:22 \
  -e AUTHORIZED_KEYS="$(cat keys/id_rsa_step5.pub)" \
  formazione-ssh-dind:latest
```

### Verifica

```bash
docker exec step5-target docker info | grep "Server Version"   # dockerd interno attivo
ssh -i keys/id_rsa_step5 -p 2222 root@localhost "docker --version"   # SSH + Docker OK
```

---

## 3. L'immagine applicativa (`app/Dockerfile`)

Immagine "app" da deployare: un Ubuntu 22.04 con SSH, accesso root a chiave (la stessa
`id_rsa_step5`). La build usa come contesto la cartella `step5` (per vedere `keys/`), col
Dockerfile in `app/`.

```dockerfile
FROM ubuntu:22.04

RUN apt-get update \
 && apt-get install -y --no-install-recommends openssh-server \
 && rm -rf /var/lib/apt/lists/* \
 && mkdir -p /run/sshd /root/.ssh

COPY keys/id_rsa_step5.pub /root/.ssh/authorized_keys
RUN chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys

RUN sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config \
 && sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/'  /etc/ssh/sshd_config

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
```

Build e push manuali di prova (poi automatizzati da Jenkins):

```bash
docker build -f app/Dockerfile -t localhost:5000/formazione-app:1 .
docker login localhost:5000        # utente: admin
docker push localhost:5000/formazione-app:1
```

Il registry (già attivo dagli step precedenti) è protetto con `htpasswd`: utente `admin`,
password recuperata dal Vault dello Step 4.

---

## 4. Il deploy con Ansible

### Inventory

Due inventory, che differiscono solo per l'host con cui si raggiunge il target:

- `inventory.ini` — esecuzione manuale dal Mac → `ansible_host=localhost`.
- `inventory-jenkins.ini` — esecuzione dalla pipeline → `ansible_host=host.docker.internal`
  (dal container Jenkins, `localhost` sarebbe sé stesso). La chiave non è indicata qui: la
  fornisce il plugin Ansible.

```ini
[target]
step5-target

[target:vars]
ansible_host=host.docker.internal
ansible_port=2222
ansible_user=root
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args=-o StrictHostKeyChecking=no
```

### Playbook (`deploy.yml`)

Ansible entra via SSH nel target e comanda il suo Docker interno. Due problemi di rete da
risolvere: il registry visto dal target è `host.docker.internal:5000`, ed è in HTTP (va
dichiarato *insecure*).

```yaml
---
- name: Deploy dell'immagine sul container target via SSH
  hosts: target
  gather_facts: false

  vars:
    registry_host: "host.docker.internal:5000"
    registry_user: "admin"
    registry_password: "Admin123"
    app_image: "formazione-app"
    app_tag: "1"
    app_container: "formazione-app-running"
    app_port: 2200

  tasks:
    - name: Assicuro la cartella di configurazione di Docker
      ansible.builtin.file:
        path: /etc/docker/
        state: directory

    - name: Dichiaro il registry come insecure (HTTP) nel daemon.json
      ansible.builtin.copy:
        dest: /etc/docker/daemon.json
        content: |
          { "insecure-registries": ["{{ registry_host }}"] }
      register: daemon_json

    - name: Ricarico dockerd per applicare la configurazione (SIGHUP)
      ansible.builtin.command: kill -HUP 1
      when: daemon_json.changed

    - name: Attendo che dockerd sia di nuovo pronto
      ansible.builtin.command: docker info
      register: docker_ready
      retries: 10
      delay: 2
      until: docker_ready.rc == 0
      changed_when: false

    - name: Login al registry
      community.docker.docker_login:
        registry_url: "{{ registry_host }}"
        username: "{{ registry_user }}"
        password: "{{ registry_password }}"

    - name: Pull dell'immagine app dal registry
      community.docker.docker_image:
        name: "{{ registry_host }}/{{ app_image }}:{{ app_tag }}"
        source: pull

    - name: Avvia il container dell'app
      community.docker.docker_container:
        name: "{{ app_container }}"
        image: "{{ registry_host }}/{{ app_image }}:{{ app_tag }}"
        state: started
        restart_policy: unless-stopped
        ports:
          - "{{ app_port }}:22"
```

Eseguito a mano ha dato `ok=6`, e al secondo run `changed=0` (idempotenza confermata).

---

## 5. Jenkins come container (immagine custom)

L'immagine ufficiale di Jenkins non ha Ansible né il client Docker. Ne creiamo una custom.

### `jenkins/Dockerfile`

```dockerfile
FROM jenkins/jenkins:lts

USER root

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg ansible openssh-client sshpass && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

RUN ansible-galaxy collection install community.docker

USER jenkins
```

> Nota: su Debian *trixie* (base di `jenkins:lts`) il pacchetto `docker.io` non fornisce più
> il binario `docker`; serve `docker-ce-cli` dal repo ufficiale.

### Build e avvio

```bash
docker build -f jenkins/Dockerfile -t jenkins-ansible:lts jenkins/

docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --group-add 0 \
  jenkins-ansible:lts
```

Montando il socket, i comandi `docker` di Jenkins girano sul Docker del Mac (dove vivono
registry e immagini).

### Plugin e credenziali

- Plugin installati: **Ansible plugin**, **Docker Pipeline**, **Docker plugin**.
- Credenziali (referenziate solo per ID, mai in chiaro nel Jenkinsfile):
  - `registry-credentials` — Username with password (`admin` / password del registry);
  - `target-ssh-key` — SSH Username with private key (username `root`, chiave `id_rsa_step5`).

---

## 6. Il Jenkinsfile

Pipeline dichiarativa a quattro stage. Le stage operative sono avvolte in `dir('step5')`
perché `checkout scm` si posiziona nella radice del repo, mentre i path (`app/Dockerfile`,
`deploy.yml`) sono relativi a `step5/`. Il tag progressivo è dato da `${BUILD_NUMBER}`.

```groovy
pipeline {
    agent any

    environment {
        REGISTRY   = "host.docker.internal:5000"
        IMAGE_NAME = "formazione-app"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        FULL_IMAGE = "${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build Immagine') {
            steps {
                dir('step5') {
                    script { docker.build("${FULL_IMAGE}", "-f app/Dockerfile .") }
                }
            }
        }

        stage('Push Immagine su Registry') {
            steps {
                dir('step5') {
                    script {
                        docker.withRegistry("http://${REGISTRY}", 'registry-credentials') {
                            docker.image("${FULL_IMAGE}").push()
                        }
                    }
                }
            }
        }

        stage('Deploy con Ansible') {
            steps {
                dir('step5') {
                    ansiblePlaybook(
                        playbook: 'deploy.yml',
                        inventory: 'inventory-jenkins.ini',
                        credentialsId: 'target-ssh-key',
                        extraVars: [ app_tag: "${IMAGE_TAG}" ]
                    )
                }
            }
        }
    }

    post {
        success { echo "Deploy completato con successo: ${FULL_IMAGE}" }
    }
}
```

Il job è di tipo **Pipeline script from SCM** → Git → repo pubblico → Script Path
`step5/Jenkinsfile`.

---

## 8. Esito

Pipeline verde end-to-end (`Finished: SUCCESS`). Ogni run builda un tag incrementale
(`formazione-app:1`, `:2`, `:3`, ...), lo pusha e Ansible deploya quel tag nel target.

```bash
$ docker exec step5-target docker ps
host.docker.internal:5000/formazione-app:3   Up   formazione-app-running
```
