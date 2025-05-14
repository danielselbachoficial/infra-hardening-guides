# Hardening de Acesso ao Zabbix com Nginx + HTTPS + Enganação de Atacantes

Este guia prático documenta como proteger sua instância do Zabbix exposta na internet usando Nginx com HTTPS, cabeçalhos de segurança, controle de acesso por IP e DNS, e técnicas para disfarçar a presença do serviço de forma eficaz.

---

## 🔐 Requisitos

* Ubuntu/Debian com Nginx
* Zabbix rodando localmente na porta `8080`
* Domínio válido (ex: `zabbix.seudominio.com`)
* Certbot para HTTPS com Let's Encrypt

---

## ⚖️ 1. Gerar Certificado HTTPS com Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d zabbix.seudominio.com
```

---

## 📂 2. Configurar Nginx com proxy reverso seguro

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

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; object-src 'none'; frame-ancestors 'none'; base-uri 'self';" always;

    location / {
        satisfy any;
        allow 138.97.35.201;
        allow 191.249.72.112;
        include /etc/nginx/whitelist.dns-allow.conf;
        deny all;

        error_page 403 = @log_denied;

        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location @log_denied {
        access_log /var/log/nginx/denied.log blocked;
        return 404;
    }
}
```

---

## 🔐 3. Isolar o backend (porta 8080)

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

    location ~ [^/]\.(php)(/|$) {
        fastcgi_pass unix:/var/run/php/zabbix.sock;
        fastcgi_param SCRIPT_FILENAME /usr/share/zabbix$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

---

## 🏡 4. Criar include com IP dinâmico

**Arquivo:** `/etc/nginx/whitelist.dns-allow.conf`

Atualizado por script:

```bash
#!/bin/bash
DNS="87d25470f4828151.sn.mynetname.net"
IP=$(getent hosts "$DNS" | awk '{print $1}')
[[ -n "$IP" ]] && echo "allow $IP;" > /etc/nginx/whitelist.dns-allow.conf && nginx -t && systemctl reload nginx
```

Agende com `crontab -e`:

```cron
*/5 * * * * /usr/local/bin/update-zabbix-whitelist.sh
```

---

## 📃 5. Ativar logs de IPs negados

**No `nginx.conf`** dentro de `http { ... }`:

```nginx
log_format blocked '$remote_addr - [$time_local] "$request" denied';
access_log /var/log/nginx/denied.log blocked;
```

---

## 🕵️‍♂️ 6. Enganar atacantes com 404 ou 444

**Exemplo de handler discreto:**

```nginx
location @log_denied {
    access_log /var/log/nginx/denied.log blocked;
    return 444; # ou 404 se preferir
}
```

Ou com uma página falsa:

```nginx
location @log_denied {
    root /var/www/fake;
    index index.html;
    access_log /var/log/nginx/denied.log blocked;
    return 200;
}
```

---

## 🚀 Conclusão

Com esse conjunto de medidas, você protege sua instância do Zabbix com:

* HTTPS com cabeçalhos modernos
* Acesso restrito por IP e DNS dinâmico
* Backend inacessível diretamente
* Logs e enganação para defesa silenciosa

Sinta-se à vontade para contribuir com melhorias ou automatizações.

> Pull requests e estrelas são bem-vindos :)
