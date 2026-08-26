#Dockerfile immagine che la pipeline builda e deploya
FROM ubuntu:22.04

# SSH server: l'immagine è Ubuntu raggiungibile via SSH (come nello Step 2)
RUN apt-get update \
 && apt-get install -y --no-install-recommends openssh-server \
 && rm -rf /var/lib/apt/lists/* \
 && mkdir -p /run/sshd /root/.ssh

 # Copy della chiave pubblica per l'accesso SSH (come nello Step 2)
COPY keys/id_rsa_step5.pub /root/.ssh/authorized_keys
RUN chmod 700 /root/.ssh && chmod 600 /root/.ssh/authorized_keys

#Hardening con solo chiave e niente password
RUN sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config \
 && sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D", "-e"]