# 🔒 Hardening de Acesso ao Zabbix com Nginx + HTTPS + Cloudflare WAF

Este guia prático documenta como proteger seu Zabbix exposto na internet usando Nginx com HTTPS, cabeçalhos seguros, Cloudflare como WAF, controle de acesso por IP e DNS, além de técnicas para disfarçar a presença do serviço.

---

## 📌 Requisitos

* Ubuntu/Debian com Nginx
* Zabbix rodando localmente na porta `8080`
* Domínio válido (ex: `zabbix.seudominio.com`)
* Certbot para HTTPS (Let's Encrypt)
* Cloudflare com proxy ativado (nuvem laranja ☁️)
* Firewall no host só permitindo conexões vindas da Cloudflare

---

## 🧱 Arquitetura Recomendada

```
Usuário Externo
      ↓
[ ☁️ Cloudflare WAF com ACLs + Obscuridade ]
      ↓ (somente IPs da Cloudflare liberados)
[ 🔥 Firewall local ]
      ↓ (porta 443 liberada apenas para Cloudflare)
[ 🌐 Nginx Proxy com TLS e Hardening ]
      ↓ (Apenas localhost)
[ 📊 Zabbix na porta 8080 ]
```

---

## ✅ 1. Gerar Certificado HTTPS

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d zabbix.seudominio.com
```

---

## ⚙️ 2. Configuração Segura do Proxy Reverso no Nginx

**Arquivo:** `/etc/nginx/sites-available/default`

```nginx
server {
    listen 80;
    server_name zabbix.seudominio.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name zabbix.seudominio.com;

    ssl_certificate /etc/letsencrypt/live/zabbix.seudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/zabbix.seudominio.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Cabeçalhos de segurança recomendados
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Server "";
    add_header X-Powered-By "";

    # Proteção contra Slowloris
    client_body_timeout 10s;
    client_header_timeout 10s;
    keepalive_timeout 15s;
    send_timeout 10s;

    location / {
        satisfy any;
        allow <IP-CLOUDFLARE-1>;
        allow <IP-CLOUDFLARE-2>;
        include /etc/nginx/whitelist.dns-allow.conf;
        deny all;

        error_page 403 = @log_denied;

        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_request_buffering off;
        chunked_transfer_encoding off;
    }

    location @log_denied {
        access_log /var/log/nginx/denied.log blocked;
        return 444;
    }

    location ~* /(wp-admin|admin|phpmyadmin|upload|\.git) {
        return 200 "OK";
        access_log /var/log/nginx/denied.log blocked;
    }
}
```

---

## 🔗 3. Isolar backend Zabbix

**Arquivo:** `/etc/nginx/conf.d/zabbix.conf`

```nginx
server {
    listen 127.0.0.1:8080;
    server_name localhost;

    root /usr/share/zabbix;
    index index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_pass unix:/var/run/php/zabbix.sock;
        fastcgi_param SCRIPT_FILENAME /usr/share/zabbix$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

---

## 🔄 4. Atualizar IP Dinâmico via DNS

**Arquivo:** `/usr/local/bin/update-zabbix-whitelist.sh`

```bash
#!/bin/bash
DNS="<DDNS-NAME>"
IP=$(getent hosts "$DNS" | awk '{print $1}')
[[ -n "$IP" ]] && echo "allow $IP;" > /etc/nginx/whitelist.dns-allow.conf && nginx -t && systemctl reload nginx
```

Agendar via cron:

```cron
*/5 * * * * /usr/local/bin/update-zabbix-whitelist.sh
```

---

## 📑 5. Configurar Logs de Tentativas Negadas

**Arquivo:** `/etc/nginx/nginx.conf` dentro do `http {}`:

```nginx
log_format blocked '$remote_addr - [$time_local] "$request" denied';
access_log /var/log/nginx/denied.log blocked;
```

---

## 🛡️ 6. Enganação Silenciosa de Atacantes

**Opções para bloqueio silencioso:**

* Com retorno HTTP 444 (sem resposta):

```nginx
location @log_denied {
    access_log /var/log/nginx/denied.log blocked;
    return 444;
}
```

* Ou com página falsa:

```nginx
location @log_denied {
    root /var/www/fake;
    index index.html;
    access_log /var/log/nginx/denied.log blocked;
    return 200;
}
```

---

## 🌐 Configuração do WAF na Cloudflare

### 🔥 Regras de Firewall

* **Permitir apenas IPs internos (ACL):**

  ```
  (http.host eq "zabbix.seudominio.com")
  AND NOT (ip.src in {10.200.30.0/24 177.23.XX.XX})
  ```

  Ação: Block

* **Bloquear outros países (opcional):**

  ```
  (http.host eq "zabbix.seudominio.com")
  AND NOT (ip.geoip.country eq "BR")
  ```

  Ação: Block

### 🤖 Bot Fight Mode

* Ative em `Security → Bots → Bot Fight Mode` (modo avançado)

### ⚠️ Proteção contra bruteforce

* Proteja `index.php` com challenge:

  ```
  (http.host eq "zabbix.seudominio.com")
  AND (http.request.uri.path contains "index.php")
  ```

  Ação: Managed Challenge

### 🚦 Rate Limiting (opcional)

* Limite acesso ao `/index.php` para evitar bruteforce:

  * 5 requisições/minuto
  * Ação: Block ou Challenge

---

## ✅ Conclusão

Agora, sua instância do Zabbix possui:

* Acesso protegido por HTTPS
* Proxy reverso com headers seguros
* Proteção WAF e rate-limit com Cloudflare
* Controle rígido por IP/DNS dinâmico
* Backend isolado e protegido
* Logs detalhados e bloqueio discreto de atacantes
