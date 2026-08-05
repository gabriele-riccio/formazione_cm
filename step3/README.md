# Step 3 — Ruoli Ansible per la gestione dei container

Rifattorizzazione degli Step 1 e 2 in **ruoli Ansible** riutilizzabili,
parametrizzati e compatibili sia con **Docker** che con **Podman**.

## Obiettivo

Partendo dai playbook monolitici degli step precedenti, impacchettare
l'automazione in ruoli che:

1. creano e configurano un **registry** locale;
2. eseguono la **build** di almeno due container con OS differenti;
3. fanno il **push** delle immagini sul registry;
4. eseguono il **run** dei container su porte host distinte (nessun conflitto).

Il tutto parametrizzato e indipendente dal container engine.

## Struttura del progetto

```
step3/
├── ansible.cfg              # inventory, roles_path
├── inventory                # localhost + ansible_python_interpreter
├── site.yml                 # orchestrazione dei 5 ruoli (esegue i cinque ruoli in ordine sequenziale)
├── group_vars/
│   └── all.yml              # variabili condivise (liste immagini/container, registry_host)
└── roles/
    ├── detect_engine/       # rileva docker/podman, sceglie container_engine
    ├── registry/            # registry:2 sulla porta 5000
    ├── build_images/        # build da template Dockerfile.j2
    ├── push_images/         # tag + push verso il registry
    └── run_containers/      # run su porte host distinte
```

## I ruoli

### detect_engine
Verifica con `which docker` / `which podman`, imposta i fact
`docker_installed` / `podman_installed` e sceglie `container_engine`.
Un `assert` blocca l'esecuzione se non è presente nessun engine.
Viene eseguito per primo: i suoi fact sopravvivono per tutto il play e sono quindi visibili agli altri ruoli.

### registry
Pull di `registry:2` e run del container `registry` sulla porta **5000**,
con volume persistente `registry_data`.

### build_images
Genera un `Dockerfile` per ogni immagine da un template Jinja2 (`Dockerfile.j2`)
con ramo per famiglia OS (`debian` e `rhel`), infine esegue la build.
Immagini prodotte: `formazione-ssh-ubuntu` (ubuntu:22.04) e
`formazione-ssh-rocky` (rockylinux:9).

### push_images
Tag delle immagini verso `localhost:5000/v2/_catalog` e push sul registry.

### run_containers
Avvia i container assegnando a ciascuno una **porta host distinta** (8081, 8082),
evitando conflitti. Le porte sono parametriche.

## Strategia Docker / Podman

Il nome di un modulo non può essere una variabile, quindi ogni ruolo usa un
**dispatcher**:

```yaml
- include_tasks: "{{ container_engine }}.yml"
```

che contiene due file paralleli: `docker.yml` (moduli `community.docker.*`) e
`podman.yml` (moduli `containers.podman.*`). Il ruolo `detect_engine`,
eseguito per primo, imposta `container_engine`; i fact sopravvivono per tutto
il play, quindi tutti i ruoli successivi sanno quale ramo usare.

> **Nota:** su questa macchina è installato solo Docker. Il ramo Podman è
> implementato e corretto, ma testato solo staticamente (non eseguito).
> Su Podman il registry insecure `localhost:5000` richiederebbe una voce in
> `registries.conf`.

## Parametrizzazione

Le variabili condivise stanno in `group_vars/all.yml`:

- `build_images_list` — elenco immagini (nome, immagine base, famiglia);
- `run_containers_list` — elenco container da avviare (nome, immagine, porta host, porta container);
- `registry_host` — indirizzo del registry.

Aggiungere un container = aggiungere una voce alla lista, senza toccare i task.

## Esecuzione

```bash
cd step3
ansible-playbook site.yml
```
### Output

![sesta parte terminale](step3_track3/Screenshot%202026-08-05%20alle%2016.47.00.png)
![sesta parte terminale](step3_track3/Screenshot%202026-08-05%20alle%2016.48.22.png)

Per forzare Podman (su una macchina che lo abbia):

```bash
ansible-playbook site.yml -e container_engine=podman
```

## Verifica

```bash
docker ps --filter "name=registry" --filter "name=app-"
curl -s http://localhost:5000/v2/_catalog
```
Su internet `http://localhost:5000/v2/_catalog`

![sesta parte terminale](step3_track3/Screenshot%202026-08-05%20alle%2016.48.31.png)
![sesta parte terminale](step3_track3/Screenshot%202026-08-05%20alle%2016.48.39.png)
