# Infraestrutura — matheussoares.dev.br

Documentação de tudo que foi configurado na VPS, no DNS e no CI/CD do portfólio.
Última atualização: 11/09/2026

---

## 1. Visão geral

| Item | Valor |
|---|---|
| Provedor VPS | Hostinger |
| Plano | KVM 1 — 4GB RAM, 1 vCPU, 50GB disco |
| SO | Ubuntu 24.04 LTS |
| Hostname | `srv1972172.hstgr.cloud` |
| IP público | `2.25.177.108` |
| Domínio | `matheussoares.dev.br` (Registro.br) |
| Backup | Semanal (padrão Hostinger) |

**Stack instalada:** Nginx · MySQL 8 · PHP 8.3 (FPM)

---

## 2. Acesso SSH

Acesso como root, autenticado por **chave SSH** (a Hostinger já provisiona a chave automaticamente — não pede senha):

```bash
ssh root@2.25.177.108
```

A senha do root foi trocada via `passwd` logo após o primeiro acesso (backup para emergências via Web Console do painel Hostinger).

Para conferir quais chaves estão autorizadas:

```bash
cat /root/.ssh/authorized_keys
```

---

## 3. DNS — configuração no Registro.br

Feito no painel do domínio → **Configurar endereçamento** (modo básico) e depois **Configurar zona DNS** (modo avançado):

| Tipo | Nome | Dados |
|---|---|---|
| A | `matheussoares.dev.br` | `2.25.177.108` |
| CNAME | `www.matheussoares.dev.br` | `matheussoares.dev.br` |

**Limitação encontrada:** o Registro.br **não aceita** o caractere `*` (wildcard) nem `@` no modo avançado. Foi o motivo da migração para o Cloudflare (seção 4).

**Trava de transição:** depois de alterar a zona DNS, o Registro.br bloqueia temporariamente (~1h) a opção "Alterar servidores DNS", impedindo a delegação para nameservers externos. É uma proteção contra sequestro de domínio recém-alterado.

---

## 4. DNS — migração para Cloudflare (wildcard)

**Objetivo:** poder usar `*.matheussoares.dev.br` e não precisar criar um registro DNS manual a cada projeto novo.

Conta criada no Cloudflare, domínio adicionado no plano **Free**.

Registros configurados lá (todos **Proxied** — nuvem laranja, o que já cuida do SSL automaticamente):

| Tipo | Nome | Conteúdo | Proxy |
|---|---|---|---|
| A | `matheussoares.dev.br` | `2.25.177.108` | Proxied |
| CNAME | `www` | `matheussoares.dev.br` | Proxied |
| A | `*` | `2.25.177.108` | Proxied |

Nameservers fornecidos pelo Cloudflare:

```
eugene.ns.cloudflare.com
leah.ns.cloudflare.com
```

> **PENDENTE:** trocar os nameservers no Registro.br (painel do domínio → "Alterar servidores DNS"), removendo `a.auto.dns.br` e `b.auto.dns.br`. Ficou bloqueado pela trava de transição descrita acima. Verificar também se DNSSEC está desativado antes de trocar.

**Como o wildcard funciona:** o Cloudflare passa a resolver qualquer subdomínio (`projeto1.`, `api.`, etc.) para o IP da VPS automaticamente. Quem decide o que responder em cada subdomínio é o **Nginx** dentro do servidor, via `server_name` de cada virtual host.

---

## 5. MySQL

### Instalação e hardening

```bash
apt install mysql-server -y
systemctl status mysql
mysql_secure_installation
```

Respostas usadas no `mysql_secure_installation`:

- VALIDATE PASSWORD COMPONENT → **Y**
- Política de senha → **2 (STRONG)**
- Remove anonymous users → **Y**
- Disallow root login remotely → **Y**
- Remove test database → **Y**
- Reload privilege tables → **Y**

> O root do MySQL no Ubuntu usa `auth_socket`: só quem estiver logado como `root` no Linux acessa o MySQL como root, e **sem senha**. Por isso o script pula a definição de senha para o root.

### Banco e usuário do portfólio

Padrão adotado: **1 banco + 1 usuário dedicado por projeto** (princípio do menor privilégio — se um projeto for comprometido, o dano fica contido naquele banco).

```sql
CREATE DATABASE portfolio;
CREATE USER 'portfolio_user'@'localhost' IDENTIFIED BY 'SENHA_FORTE';
GRANT ALL PRIVILEGES ON portfolio.* TO 'portfolio_user'@'localhost';
FLUSH PRIVILEGES;
```

Acessar o MySQL como root (dentro da VPS, logado como root):

```bash
mysql
```

---

## 6. PHP

```bash
apt install php-fpm php-mysql php-cli php-curl php-mbstring php-xml php-zip php-gd php-intl -y
```

| Item | Valor |
|---|---|
| Versão | PHP 8.3.6 |
| Serviço | `php8.3-fpm` |
| Socket | `/run/php/php8.3-fpm.sock` |
| Config do pool | `/etc/php/8.3/fpm/pool.d/www.conf` |

Comandos úteis:

```bash
php -v
systemctl status php8.3-fpm
systemctl restart php8.3-fpm
```

> O portfólio atual é HTML/CSS estático e **não usa** o PHP-FPM. Ele já está instalado para os projetos Laravel/Symfony que virão.

---

## 7. Nginx — virtual host do portfólio

### Estrutura de pastas

```bash
mkdir -p /var/www/matheussoares.dev.br/html
```

### Arquivo de configuração

`/etc/nginx/sites-available/matheussoares.dev.br`

```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/matheussoares.dev.br/html;
    index index.html index.htm;

    server_name matheussoares.dev.br www.matheussoares.dev.br;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Ativação

```bash
ln -s /etc/nginx/sites-available/matheussoares.dev.br /etc/nginx/sites-enabled/
nginx -t                  # valida a sintaxe antes de aplicar
systemctl reload nginx
```

### Comandos úteis

```bash
ls -la /etc/nginx/sites-enabled/          # sites ativos
nginx -t                                   # testar config
systemctl reload nginx                     # aplicar sem derrubar conexões
systemctl restart nginx                    # reiniciar de fato
tail -20 /var/log/nginx/error.log          # logs de erro (precisa ser root)
```

### Testar sem depender do navegador

```bash
# Testa o virtual host específico, sem cache de navegador no meio:
curl -I -H "Host: matheussoares.dev.br" http://localhost
```

> Atenção: `curl -I http://localhost` sem o header `Host` cai no site `default` do Nginx, não no seu virtual host.

---

## 8. Usuário `deploy` (automação sem root)

Criado para que o CI/CD não use `root`. Se a chave do GitHub Actions vazar, o atacante só alcança as pastas dos sites — não o servidor inteiro.

```bash
adduser deploy
chown -R deploy:deploy /var/www/matheussoares.dev.br
usermod -aG deploy www-data
chmod -R 750 /var/www/matheussoares.dev.br
chmod o+x /var/www/matheussoares.dev.br
```

Características:

- **Sem sudo** — qualquer comando administrativo precisa ser rodado como root (`exit` da sessão do deploy).
- Dono das pastas em `/var/www/`.
- `www-data` (usuário do Nginx) foi adicionado ao grupo `deploy` para conseguir ler os arquivos publicados.

---

## 9. Chave SSH do GitHub Actions

Gerada **dentro da VPS**, como o usuário `deploy`:

```bash
su - deploy
ssh-keygen -t ed25519 -C "github-actions-deploy"   # SEM passphrase (automação)
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

A chave **privada** (`~/.ssh/id_ed25519`) foi copiada inteira — incluindo as linhas `-----BEGIN OPENSSH PRIVATE KEY-----` e `-----END OPENSSH PRIVATE KEY-----` — e colada no GitHub como Secret.

```bash
cat ~/.ssh/id_ed25519    # exibe a chave privada para copiar
```

---

## 10. GitHub — Secrets do repositório

Repositório: `github.com/matheusdeveloperphp/portifolio`
Caminho: **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Valor |
|---|---|
| `SSH_PRIVATE_KEY` | conteúdo completo da chave privada (seção 9) |
| `SSH_HOST` | `2.25.177.108` |
| `SSH_USER` | `deploy` |
| `DEPLOY_PATH` | `/var/www/matheussoares.dev.br/html` |

---

## 11. GitHub Actions — workflow de deploy

Arquivo: `.github/workflows/deploy.yml`

```yaml
name: Deploy Portfolio

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "."
          target: ${{ secrets.DEPLOY_PATH }}
          rm: false
```

**O que cada parte faz:**

- `on.push.branches: [main]` — dispara a cada push/merge na `main`.
- `actions/checkout@v4` — baixa o código do repositório dentro do runner.
- `appleboy/scp-action` — conecta na VPS via SSH e copia os arquivos.
- `source: "."` — copia tudo a partir da raiz do repositório.
- `rm: false` — **não apaga** o que já está no destino. Sobrescreve e adiciona, mas arquivos deletados do repositório continuam existindo na VPS (ver pendências).

---

## 12. Git local — duas contas GitHub na mesma máquina

**Situação:** a chave SSH padrão da máquina pertence à conta de trabalho (`msoareslima`), mas o portfólio está na conta `matheusdeveloperphp`. Tentar usar a chave errada resulta em `Permission denied (publickey)`.

**Solução:** chave separada + alias de host, sem tocar na configuração da conta de trabalho.

### 1. Gerar a chave (no Git Bash, máquina local)

```bash
ssh-keygen -t ed25519 -C "matheusdeveloperphp" -f ~/.ssh/id_ed25519_portfolio
cat ~/.ssh/id_ed25519_portfolio.pub
```

### 2. Cadastrar a pública no GitHub

Logado como `matheusdeveloperphp`: **Settings → SSH and GPG keys → New SSH key**.

### 3. Criar o alias em `~/.ssh/config`

```bash
cat >> ~/.ssh/config << 'EOF'

Host github.com-portfolio
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_portfolio
EOF
```

### 4. Apontar o remote do repositório para o alias

```bash
git remote set-url origin git@github.com-portfolio:matheusdeveloperphp/portifolio.git
```

### 5. Testar

```bash
ssh -T git@github.com-portfolio
# Esperado: Hi matheusdeveloperphp! You've successfully authenticated...
```

> O alias `github.com-portfolio` não é um domínio real — é só um apelido que o SSH resolve para `github.com` usando a chave certa. Repositórios da empresa continuam usando `github.com` normal e a chave antiga.

---

## 13. Fluxo de trabalho diário

```bash
# 1. editar o código localmente
git add .
git commit -m "mensagem"
git push origin main
```

O GitHub Actions dispara sozinho, copia os arquivos para a VPS e o site reflete a mudança. Nenhum acesso manual ao servidor é necessário.

Acompanhar a execução: aba **Actions** do repositório no GitHub. Se falhar, clicar na execução → clicar no step com ❌ para ver o log.

---

## 14. Problemas encontrados e como foram resolvidos

### 404 Not Found, mesmo com os arquivos presentes na VPS

**Sintoma:** `curl -I -H "Host: matheussoares.dev.br" http://localhost` retornava 404, mas `ls` mostrava o `index.html` no lugar certo.

**Causa:** `chmod -R 750` na pasta pai tirou a permissão de execução ("passagem") para "outros". Sem o bit `x` no diretório, o usuário do Nginx (`www-data`) não consegue nem atravessar a pasta para chegar nos arquivos.

**Solução:**

```bash
chmod o+x /var/www/matheussoares.dev.br
```

### Workflow falhando com `ssh.ParsePrivateKey: ssh: no key found`

**Causa:** o secret `SSH_PRIVATE_KEY` foi colado incompleto (faltavam as linhas `BEGIN`/`END` ou as quebras de linha se perderam).

**Solução:** recopiar o `cat ~/.ssh/id_ed25519` inteiro, do `-----BEGIN-----` ao `-----END-----`, e regravar o secret. Depois, **Re-run all jobs** na aba Actions.

### Aba "Settings" não aparecia no repositório do GitHub

**Causa:** estava logado na conta `msoareslima`, que não é dona do repositório `matheusdeveloperphp/portifolio`. Sem permissão de admin, o GitHub esconde a aba e retorna 404 na URL direta.

**Solução:** logar com a conta dona do repositório.

### `sudo` negado para o usuário `deploy`

Esperado — o `deploy` foi criado de propósito sem privilégios administrativos. Para comandos de root, sair da sessão dele:

```bash
exit    # volta para root@srv1972172
```

---

## 15. Pendências

- [ ] **Trocar os nameservers no Registro.br** para `eugene.ns.cloudflare.com` e `leah.ns.cloudflare.com` (remover `a.auto.dns.br` e `b.auto.dns.br`); conferir se DNSSEC está desativado.
- [ ] **Configurar HTTPS** com Let's Encrypt/Certbot — o site hoje responde só em HTTP e o navegador marca como "Não seguro".
- [ ] **Adaptar o workflow para Angular/React** — adicionar steps de `npm ci` + `npm run build` e publicar apenas o conteúdo de `dist/`.
- [ ] **Adaptar o workflow para projetos PHP** (Laravel/Symfony) — `composer install --no-dev`, migrations, permissões de `storage/` e `var/`.
- [ ] **Revisar `rm: false`** — hoje arquivos deletados do repositório continuam órfãos na VPS. Avaliar `rm: true` ou trocar a estratégia para `git pull` no servidor.
- [ ] **Excluir `.git`, `.github` e `README.md` do deploy** — hoje são copiados junto para a pasta pública sem necessidade.
- [ ] **Criar o primeiro subdomínio de projeto**, replicando o padrão do virtual host (seção 7) e, para Java/Spring Boot, adicionando `proxy_pass` para a porta interna da aplicação.

---

## 16. Padrão para adicionar um projeto novo

Referência rápida para quando for subir o primeiro projeto no subdomínio.

### Projeto estático (Angular/React buildado)

```nginx
server {
    listen 80;
    server_name projeto1.matheussoares.dev.br;

    root /var/www/projeto1/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;   # fallback para SPA
    }
}
```

### Projeto PHP (Laravel/Symfony)

```nginx
server {
    listen 80;
    server_name projeto2.matheussoares.dev.br;

    root /var/www/projeto2/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

### Projeto Java (Spring Boot) — proxy reverso

A aplicação roda numa porta interna (ex.: 8081) e o Nginx repassa:

```nginx
server {
    listen 80;
    server_name projeto3.matheussoares.dev.br;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Em todos os casos, depois de criar o arquivo em `sites-available`:

```bash
ln -s /etc/nginx/sites-available/NOME /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```
