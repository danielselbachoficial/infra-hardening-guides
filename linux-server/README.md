# 🔒 Hardening de Acesso ao Servidor via SSH com hosts.deny + Logs + UFW

Este repositório contém um guia prático para proteger servidores Linux (Ubuntu/Debian) com controle de acesso baseado em IP, usando:

- `/etc/hosts.allow` e `/etc/hosts.deny` (TCP Wrappers)
- Logging personalizado via `rsyslog`
- Firewall UFW (Uncomplicated Firewall)
- Monitoramento com `lnav`

> **Ideal para:** Ambientes onde o acesso remoto via SSH precisa ser **restrito, rastreável e discreto**, como laboratórios, redes internas, servidores em DMZ ou expostos à internet.

---

## 📁 Conteúdo

- `manual.md` — Guia completo passo a passo
- Exemplos de configuração (`hosts.allow`, `hosts.deny`)
- Comandos para instalação, logging e firewall
- Boas práticas de segurança

---

## 📌 Requisitos

- Ubuntu Server 24.04 LTS ou Debian 12
- Acesso root ou sudo
- SSH ativo
- UFW instalado e habilitado
- IP confiável para liberar no acesso

---

## 🧱 Arquitetura Resumida

Usuário autorizado (IP confiável)
↓
[🔒 /etc/hosts.allow — IPs permitidos]
↓
[🚫 /etc/hosts.deny — bloqueio e log dos demais]
↓
[🔥 UFW — só porta 22 liberada]
↓
[🎯 OpenSSH ativo e monitorado]
