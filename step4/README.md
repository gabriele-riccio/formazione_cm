# Step 4 — Ansible Vault

Uso di **Ansible Vault** per oscurare tutte le password del modulo, in particolare le
credenziali di accesso al Docker registry (`user` / `password`), così che non compaiano mai
in chiaro nel repository versionato su Git.

Sono partito  dalla base dello **Step 3** (5 ruoli orchestrati da `site.yml`) ed ho aggiunto
l'autenticazione basic sul registry con le credenziali protette da Vault.

---

## Prerequisiti

- Docker attivo
- `ansible-core` (2.15.x), collection `community.docker`
- Un **vault password file** in `~/.vault` 

## Creazione VAULT PASSWORD FILE 

Per prima cosa ho creato `il vault password file` con la sua sola password e gli ho fornito i permessi di sola lettura e scrittura.
```bash
echo 'Admin123' > ~/.vault && chmod 600 ~/.vault
```

> Il file `~/.vault` vive **fuori dal repo** e non viene mai committato.

## Cifrare le password del registry (encrypt_string)

```bash
ansible-vault encrypt_string 'Admin123' --name 'registry_password' \
  --vault-password-file ~/.vault
```
Ho prodotto un blocco `!vault` da incollare in `group_vars/all.yml`.
Il file resta leggibile, con in chiaro il nome della variabile e i valori non segreti, con solo la password cifrata.

```bash
# group_vars/all.yml
registry_host: "localhost:5000"
registry_user: "admin"
registry_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256 63656133363839626266663234363033...
```
## Htpasswd per il registry

Poi ho generato l'`htpasswd` per il registry dato che il registry basic-auth richiede un file htpasswd in formato bcrypt.
In roles/registry/tasks/htpasswd.yml :

```bash
---
- name: Creo la cartella auth del registry
  ansible.builtin.file:
    path: "{{ registry_auth_dir }}"
    state: directory
    mode: "0755"
- name: Genero il file htpasswd (bcrypt) con le credenziali del registry
  ansible.builtin.shell: >
    docker run --rm --entrypoint htpasswd httpd:2
    -Bbn {{ registry_user }} {{ registry_password }} register: htpasswd_out
  changed_when: true
  no_log: true
- name: Scrivo il file htpasswd su disco
ansible.builtin.copy:
  content: "{{ htpasswd_out.stdout }}\n"
  dest: "{{ registry_auth_dir }}/htpasswd"
  mode: "0644"
```
> htpasswd è assente in registry:2 perch ho generato l'hash con l'immagine httpd:2 (Apache), che lo contiene.
> Flag usati:
> -B bcrypt,
> -b password da riga di comando,
> -n stampa su stdout

Il file `htpasswd` è **rigenerato a ogni run** dal playbook a partire dalla password cifrata,
quindi non è versionato (nel repo sta la fonte cifrata, non l'artefatto derivato).

## Registry con auth basic
Ho modificato il file `docker roles/registry/tasks/docker.yml` includendo l'htpasswd e avviando il container attraverso le env di autenticazione e il mount della cartella `auth`:

```bash
- name: Genero il file htpasswd per il registry (Docker)
  ansible.builtin.include_tasks: htpasswd.yml

- name: Avvio del container registry con auth basic (Docker)
  community.docker.docker_container:
    name: "{{ registry_name }}"
    image: "{{ registry_image }}"
    state: started
    restart_policy: "{{ registry_restart_policy }}"
    ports:
      - "{{ registry_port }}:5000"
    env:
      REGISTRY_AUTH: "htpasswd"
      REGISTRY_AUTH_HTPASSWD_REALM: "{{ registry_auth_realm }}"
      REGISTRY_AUTH_HTPASSWD_PATH: "/auth/htpasswd"
    volumes:
      - "{{ registry_data_volume }}:/var/lib/registry"
      - "{{ registry_auth_dir }}:/auth:ro"
```
Poi ho aggiunto le variabili in `roles/registry/defaults/main.yml`:

``` bash
registry_auth_dir: "{{ playbook_dir }}/auth"
registry_auth_realm: "Registry Realm"
```
Ed ho aggiunto il login cifrato prima del push in `roles/push_images/tasks/docker.yml`
Il docker_login usa la password decifrata da Vault a runtime, senza login il registry risponderebbe 401 Unauthorized:

- name: Login al registry con le credenziali cifrate (Docker)
  community.docker.docker_login:
    registry_url: "{{ registry_host }}"
    username: "{{ registry_user }}"
    password: "{{ registry_password }}"
  no_log: true
  
- name: Tag delle immagini verso il registry (Docker)
  community.docker.docker_image:
    name: "{{ item.name }}"
    repository: "{{ registry_host }}/{{ item.name }}"
    tag: latest
    source: local
    push: true
  loop: "{{ build_images_list }}"

--

## Come si esegue

Ho copiato i file già utilizzati nello step 3 (per questo il Readme all'inizio era quello di quello step) ed ho modificato alcuni file per poi riapplicare il playbook con tutte le password del modulo, in particolare le
credenziali di accesso al Docker registry, oscurate tramite l'ansible vault.

Ho eliminato il container docker che usavo nello step 3, lasciando vivo il volume, in modo da farlo ripartire con l'auth che ho aggiunto nello step 4. Lo faccio una tantum per liberare la porta 5000 se occupata.

```bash
# (una tantum) rimuovi un eventuale registry senza auth dello Step 3
docker rm -f registry        # il volume registry_data resta intatto

# esecuzione completa
ansible-playbook site.yml --vault-password-file ~/.vault
```

![sesta parte terminale](step4_track3/Screenshot%202026-08-10%20alle%2010.30.04.png)
![sesta parte terminale](step4_track3/Screenshot%202026-08-10%20alle%2010.30.17.png)

### Verifica dell'autenticazione

```bash
# Senza credenziali => 401
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:5000/v2/_catalog

# Con credenziali => 200 + lista repository
curl -s -u admin:Admin123 http://localhost:5000/v2/_catalog

```
![sesta parte terminale](step4_track3/Screenshot%202026-08-10%20alle%2010.29.49.png)
![sesta parte terminale](step4_track3/Screenshot%202026-08-10%20alle%2011.12.59.png)
![sesta parte terminale](step4_track3/Screenshot%202026-08-10%20alle%2011.13.17.png)


---

## Cosa fa Vault qui

Vault cifra i **dati** (le variabili), non la logica: playbook e task restano leggibili,
mentre i segreti vengono decifrati in memoria a runtime dietro richiesta della vault password.

Due granularità usate nel modulo:

| Approccio | Comando | Risultato | Uso nel modulo |
|---|---|---|---|
| **Stringa inline** | `ansible-vault encrypt_string` | solo il valore è cifrato, il file YAML resta leggibile e diffabile | password del registry in `group_vars/all.yml` |
| **File intero** | `ansible-vault encrypt` | tutto il file diventa un blob illeggibile | demo in `vars/secrets.yml` |
