# Step 5 — Jenkins & Ansible

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

## Il container target (Docker-in-Docker + SSH)

Base `docker:dind` (demone Docker già pronto) con l'aggiunta di `openssh` (accesso SSH) e
`python3` + SDK Docker (necessari ai moduli `community.docker` che Ansible esegue *dentro* il
target). L'entrypoint fa da direttore d'orchestra: rigenera le host key SSH, inietta la chiave
pubblica autorizzata, avvia `sshd -D` in background e infine `exec dockerd-entrypoint.sh`
(così il demone Docker resta il processo principale e tiene vivo il container).

Avvio:

```bash
docker run -d --privileged --name step5-target \
  -p 2222:22 \
  -e AUTHORIZED_KEYS="$(cat keys/id_rsa_step5.pub)" \
  formazione-ssh-dind:latest
```

`--privileged` è necessario perché il demone Docker interno deve gestire network, cgroups e
filesystem.

---

## La pipeline Jenkins

`agent any`, quattro stage:

1. **Checkout** — clona il repo (Pipeline script from SCM, Script Path `step5/Jenkinsfile`).
2. **Build Immagine** — `docker.build` dell'app con tag `${BUILD_NUMBER}` → **tag progressivo**.
3. **Push su Registry** — `docker.withRegistry(..., 'registry-credentials')` + push.
4. **Deploy con Ansible** — `ansiblePlaybook` lancia `deploy.yml` sul target, iniettando la
   chiave SSH dalla credenziale `target-ssh-key` e passando `app_tag=${BUILD_NUMBER}`.

Poiché `checkout scm` si posiziona nella radice del repo, le stage operative sono avvolte in
`dir('step5')` per risolvere i path relativi (`app/Dockerfile`, `deploy.yml`, ...).

### Credenziali Jenkins

- `registry-credentials` — *Username with password* (utente/password del registry).
- `target-ssh-key` — *SSH Username with private key* (username `root`, chiave `id_rsa_step5`).

I segreti non compaiono mai nel Jenkinsfile: si referenziano solo per ID.

---

## Il deploy con Ansible

`deploy.yml` entra via SSH nel target e comanda il suo Docker interno. I task, in ordine:

1. configura il registry come **insecure** (`/etc/docker/daemon.json`) e ricarica dockerd (SIGHUP);
2. attende che dockerd sia di nuovo pronto;
3. `docker_login` al registry;
4. `docker_image` — pull dell'immagine app;
5. `docker_container` — avvio del container app.

Punto chiave di rete: dal *dentro* del target il registry sul Mac si raggiunge come
`host.docker.internal:5000`, non `localhost`.

---

## Plugin Jenkins utilizzati

- **Ansible plugin** — fornisce lo step `ansiblePlaybook`.
- **Docker Pipeline** — fornisce il DSL `docker.build` / `docker.withRegistry` / `.push()`.
- **Docker plugin** — installato per completezza rispetto alla traccia.

---

## Esito

Pipeline verde end-to-end (`Finished: SUCCESS`). La build progressiva produce
`formazione-app:1`, `:2`, `:3`, ... e ad ogni run Ansible deploya il tag corrispondente
dentro il target. Verifica:

```bash
docker exec step5-target docker ps
# host.docker.internal:5000/formazione-app:<N>   Up   formazione-app-running
```
