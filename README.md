# python-local-server

## Hospedar localmente e publicar via túnel reverso

Este repositório mostra como hospedar um projeto estático/leve na sua máquina local usando o servidor HTTP embutido do Python e, ao mesmo tempo, torná-lo acessível pela internet por meio de um túnel SSH reverso para um servidor em nuvem (`<IP-DO-SERVIDOR-EM-NUVEM>`). Fornecemos dois serviços systemd (no escopo do usuário): um para iniciar o servidor local e outro para manter o túnel aberto e resiliente.

## Services systemd (user) para servidor local e túnel SSH

Este workspace inclui dois arquivos de serviço systemd (user) para:
- Subir um servidor HTTP Python na porta 5001.
- Abrir um túnel reverso SSH para a instância `<IP-DO-SERVIDOR-EM-NUVEM>`, expondo a porta 5001.

Arquivos:
- `systemd/python-local-server.service`
- `systemd/ssh-reverse-tunnel.service`

## Pré-requisitos
- Chaves SSH funcionando para o host `<IP-DO-SERVIDOR-EM-NUVEM>` (ex.: `~/.ssh/id_rsa` + `~/.ssh/config`).
- O hostname/IP `<IP-DO-SERVIDOR-EM-NUVEM>` resolvendo via `~/.ssh/config` ou DNS.
- `python3` e `ssh` instalados em `/usr/bin`.

## Instalação (user services)
1) Copiar os serviços para a pasta do usuário:

```
mkdir -p ~/.config/systemd/user
cp systemd/*.service ~/.config/systemd/user/
```

2) Recarregar o systemd do usuário:

```
systemctl --user daemon-reload
```

3) Habilitar e iniciar os serviços:

```
systemctl --user enable --now python-local-server.service
systemctl --user enable --now ssh-reverse-tunnel.service
```

4) (Opcional) Manter após logout (linger):

```
loginctl enable-linger "$USER"
```

## Verificação e logs
- Status:

```
systemctl --user status python-local-server.service
systemctl --user status ssh-reverse-tunnel.service
```

- Logs ao vivo:

```
journalctl --user -u python-local-server.service -f
journalctl --user -u ssh-reverse-tunnel.service -f
```

## Parar/Desabilitar
```
systemctl --user stop ssh-reverse-tunnel.service
systemctl --user stop python-local-server.service

systemctl --user disable ssh-reverse-tunnel.service
systemctl --user disable python-local-server.service
```

## Observações
- O serviço `python-local-server.service` usa `WorkingDirectory=%h/projetos/python-server`. Ajuste se o caminho do projeto for diferente.
- O túnel não usa `-f` porque o systemd espera o processo em foreground.
- O `ssh-reverse-tunnel.service` falha de forma explícita se a ligação do túnel não for possível (`ExitOnForwardFailure=yes`) e faz keepalive.
- Para melhor robustez, você pode usar `autossh` no lugar de `ssh` (requer instalação do pacote):

```
# Substitua o ExecStart do túnel por:
/usr/bin/autossh -M 0 -N -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -R 5001:localhost:5001 <IP-DO-SERVIDOR-EM-NUVEM>
```

- Se quiser que a porta 5001 na instância AWS fique acessível publicamente:
  - Use `-R 0.0.0.0:5001:localhost:5001` no comando.
  - No servidor AWS, em `/etc/ssh/sshd_config`, habilite `GatewayPorts clientspecified` e recarregue o sshd: `sudo systemctl reload sshd`.
  - Libere a porta 5001 no Security Group da instância.

## Teste rápido
- A partir da máquina local: `curl http://localhost:5001` deve retornar o conteúdo de `index.html`.
- A partir da instância `<IP-DO-SERVIDOR-EM-NUVEM>`: `curl http://localhost:5001` deve retornar o mesmo conteúdo (via túnel reverso).

## Autor
- Nome: Fábio Berbert de Paula
- Email: fberbert@gmail.com
