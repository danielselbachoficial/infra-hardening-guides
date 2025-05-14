# 🔒 Hardening de Acesso ao Zabbix com Nginx + HTTPS + Enganação de Atacantes

Este guia prático documenta como proteger seu Zabbix exposto na internet usando Nginx com HTTPS, cabeçalhos seguros, controle de acesso por IP e DNS, além de técnicas para disfarçar a presença do serviço.

---

## 📌 Requisitos

* Ubuntu/Debian com Nginx
* Zabbix rodando localmente na porta `8080`
* Domínio válido (ex: `zabbix.seudominio.com`)
* Certbot para HTTPS (Let's Encrypt)

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

    location / {
        satisfy any;
        allow <IP-PUBLICO>;
        allow <IP-PUBLICO>;
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

## ✅ Conclusão

Agora, sua instância do Zabbix possui:

* Acesso protegido por HTTPS
* Controle rígido por IP/DNS dinâmico
* Backend isolado e protegido
* Logs detalhados e bloqueio discreto de atacantes
