FROM ubuntu:24.04
ARG USUARIO
ARG CONTRASENA
ARG DEBIAN_FRONTEND=noninteractive

RUN test $USUARIO || { echo 'defina USUARIO'; exit 1; }

RUN apt-get update -y && \
    apt-get install -y openssh-server nano sudo && \
    useradd -ms /bin/bash "$USUARIO" && \
    echo "$USUARIO:${CONTRASENA:-pass}" | chpasswd && \
    usermod -aG sudo "$USUARIO" && \
    mkdir -p /home/$USUARIO/.ssh && \
    sed -i -e 's/^%sudo/#&/' -e "$ a %sudo ALL=(ALL) NOPASSWD: ALL" /etc/sudoers && \
    echo 'PermitRootLogin no' >> /etc/ssh/sshd_config && \
    echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config && \
    sed -i -e '$a sudo service ssh start >/dev/null 2>&1' /home/$USUARIO/.bashrc

USER $USUARIO
WORKDIR /home/$USUARIO
CMD ["bash"]
