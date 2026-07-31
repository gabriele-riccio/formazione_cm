# Step 2 — Build di container SSH con Ansible

Playbook Ansible che effettua la build di due container con **OS differenti**
(Ubuntu e Rocky Linux), ciascuno configurato con un servizio SSH pronto
all'uso e un utente amministratore che accede tramite chiave.

## Requisiti soddisfatti

Ogni container prodotto:

- è in ascolto sulla **porta 22** (SSH esposta e mappata sull'host);
- ha il **servizio sshd attivo** come processo principale (`sshd -D`);
- ha l'utente **`gabriele`**, abilitato al login via **chiave SSH** e con
  permessi **sudo** (senza password).

## Struttura
