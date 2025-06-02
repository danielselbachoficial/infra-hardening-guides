# 🔒 Hardening de Acesso ao Servidor via SSH com hosts.deny + Logs + UFW

Este guia prático documenta como proteger servidores Linux com controle de acesso baseado em IP usando `hosts.deny`, logs personalizados com `rsyslog`, firewall UFW, e monitoramento com `lnav`.

Ideal para ambientes onde o acesso remoto via SSH precisa ser **restrito, rastreável e discreto**, como laboratórios, redes internas ou servidores expostos à internet.

---

## 📌 Requisitos

- Servidor Ubuntu Server 24.04 LTS ou Debian 12
- Acesso root ou sudo
- SSH habilitado
- UFW instalado e ativado
- IP público ou local confiável para liberação

---

## 🧱 Arquitetura Recomendada

Usuário autorizado (IP confiável)  
  ⬇  
[ 🔒 `/etc/hosts.allow` — IPs permitidos ]  
  ⬇  
[ 🚫 `/etc/hosts.deny` — bloqueia e loga todos os demais permitidos ]  
  ⬇  
[ 🔥 UFW — permite apenas porta 22 ]  
  ⬇  
[ 🎯 OpenSSH rodando no servidor ]

---

## ✅ 1. Instalar pacotes de log e análise

```bash
sudo apt update
sudo apt install -y rsyslog lnav vim ufw
sudo systemctl enable rsyslog
sudo systemctl restart rsyslog
```

## 📁 2. Criar arquivos de log personalizados
```bash
sudo touch /var/log/hosts-deny.log /var/log/hosts-allow.log
sudo chmod 644 /var/log/hosts-deny.log /var/log/hosts-allow.log
```

## 🔐 3. Configurar bloqueio padrão via /etc/hosts.deny
```bash
sudo vim /etc/hosts.deny
```

Adicione:
```bash
ALL: ALL: spawn /bin/echo "$(date) | Serviço Remoto %d | Host Remoto %c | Porta Remota %r | Processo Local %p" >> /var/log/hosts-deny.log
```

## 🔓 4. Liberar IPs confiáveis via /etc/hosts.allow
```bash
sudo vim /etc/hosts.allow
```

Exemplo para liberar um IP específico:


## 🔥 5. Configurar o firewall (UFW)
```bash
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw reload
```

Verifique as regras:
```bash
sudo ufw status verbose
```

## 🧪 6. Testar o serviço SSH
Garanta que o SSH está ativo:
```bash
sudo systemctl enable ssh
sudo systemctl restart ssh
sudo systemctl status ssh
```

## 🕵️ 7. Monitorar tentativas bloqueadas

Via tail:
```bash
tail -f /var/log/hosts-deny.log
```

Via lnav:
```bash
sudo lnav /var/log/hosts-deny.log
```

## 🛡️ 8. Considerações de Segurança
Apenas serviços compatíveis com TCP Wrappers (como o SSH) são afetados por hosts.allow / hosts.deny.

Esse hardening é compatível com iptables, nftables, fail2ban, ou Cloudflare Zero Trust.

Essa abordagem reduz ruído de scanners e bots, e ainda gera logs legíveis e acionáveis.

# “Auditar é proteger. Bloquear é preservar.” — Tradição SysAdmin #
