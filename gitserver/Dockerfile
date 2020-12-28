FROM alpine

RUN apk add --no-cache \
      git \
      openssh-server

RUN addgroup git \
      --system &&\
    adduser git \
      --disabled-password \
      --no-create-home \
      --home=/srv/git \
      --shell=/usr/bin/git-shell \
      --ingroup=git &&\
    passwd -u git

RUN rm -rf /etc/ssh/*
COPY ssh* /etc/ssh/

RUN mkdir /srv/git/.ssh --parent &&\
    git init --bare /srv/git/config

COPY authorized_keys /srv/git/.ssh/authorized_keys
RUN chown git:git /srv/git -R &&\
    chmod 700 /srv/git/.ssh &&\
    chmod 400 /srv/git/.ssh/authorized_keys

ENTRYPOINT ["/usr/sbin/sshd", "-D", "-e"]
