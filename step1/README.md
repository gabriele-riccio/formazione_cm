## Step 1 — Configurare un Docker Registry locale

### Obiettivo
Scrivere un playbook Ansible (`container-playbook.yml`) che configuri un **Docker Registry**
privato e locale (anche senza autenticazione), avviato come container a partire
dall'immagine ufficiale `registry:2` ed esposto su `localhost:5000`.

### File
- `container-playbook.yml` — il playbook.

### Prerequisiti
- **Docker Desktop** attivo (il demone deve rispondere: `docker ps`).
- **Collection** `community.docker` installata.
- **SDK Python** `docker` disponibile sull'interprete usato da Ansible.

### Il playbook

```yaml
---
- name: Configura un Docker Registry locale
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    ansible_python_interpreter: /usr/local/bin/python3

  tasks:
    - name: Assicura la presenza dell'immagine del registry
      community.docker.docker_image:
        name: registry:2
        source: pull
        state: present

    - name: Avvia il container del registry
      community.docker.docker_container:
        name: registry
        image: registry:2
        state: started
        restart_policy: always
        ports:
          - "5000:5000"
        volumes:
          - registry_data:/var/lib/registry
```

### Cosa fa, task per task
- **docker_image** → garantisce che l'immagine `registry:2` sia presente in locale
  (`source: pull` la scarica se manca; `state: present` è lo stato desiderato).
- **docker_container** → crea e avvia il container `registry`:
  - `ports "5000:5000"` mappa la porta del container sull'host → raggiungibile su `localhost:5000`;
  - `volumes registry_data:/var/lib/registry` rende **persistenti** su un volume le immagini pushate;
  - `restart_policy: always` fa ripartire il container dopo un riavvio del demone/host.

### Esecuzione

```bash
ansible-playbook container-playbook.yml
```

### Verifica

```bash
# Il container è attivo con la porta esposta
docker ps --filter "name=registry"

# L'API del registry risponde (vuoto se non è stato pushato nulla)
curl http://localhost:5000/v2/_catalog     # -> {"repositories":[]}
```
![prima parte terminale](step1_track3/Screenshot%202026-07-30%20alle%2010.46.39.png)
![prima parte terminale](step1_track3/Screenshot%202026-07-30%20alle%2010.46.19.png)

```bash
# Prova di push reale
docker pull hello-world
docker tag hello-world localhost:5000/hello-world
docker push localhost:5000/hello-world
curl http://localhost:5000/v2/_catalog     # -> {"repositories":["hello-world"]}
```
![prima parte terminale](step1_track3/Screenshot%202026-07-30%20alle%2010.48.34.png)
![prima parte terminale](step1_track3/Screenshot%202026-07-30%20alle%2010.48.27.png)

### Idempotenza
Rieseguendo il playbook senza modifiche, il `PLAY RECAP` riporta `changed=0`:
il registry è già nello stato desiderato e Ansible non modifica nulla.

---

## Note tecniche: Come ho risolto un problema

Durante lo Step 1 sono emersi due problemi tipici dell'ambiente locale:

1. **Mismatch versione collection ↔ ansible-core.**
   Con `ansible-core 2.15` la `community.docker` recente (3.4.x) crea il container ma
   **non lo avvia** (resta in stato `Created`), emettendo il warning
   *"does not support Ansible version 2.15.13"*.
   **Soluzione:** installare una versione compatibile della collection:
   ```bash
   ansible-galaxy collection install community.docker:2.7.0 --force
   ```

2. **Interprete Python sbagliato.**
   Ansible eseguiva i moduli con un `python3` privo dell'SDK `docker`
   (`ModuleNotFoundError: No module named 'docker'`).
   **Soluzione:** dichiarare nel playbook l'interprete che ha l'SDK installato:
   ```yaml
   vars:
     ansible_python_interpreter: /usr/local/bin/python3
   ```
Il playbook dopo l'esecuzione creava il container `registry` ma non lo faceva partire per le due motivazioni sopracitate.
Installando la versione compatibile della collection e dichiarando nel playbook l'interprete python che ha l'SDK installato ho risolto i problemi.
