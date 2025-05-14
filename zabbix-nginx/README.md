# 🔒 Hardening de Acesso ao Zabbix com Nginx + HTTPS + Enganação de Atacantes

Este guia prático documenta como proteger seu Zabbix exposto na internet usando Nginx com HTTPS, cabeçalhos seguros, controle de acesso por IP e DNS, além de técnicas para disfarçar a presença do serviço.

---

## 📌 Índice

* [Requisitos](#requisitos)
* [Gerar Certificado HTTPS](#gerar-certificado-https)
* [Configuração Segura do Proxy Reverso no Nginx](#configuração-segura-do-proxy-reverso-no-nginx)
* [Isolar backend Zabbix](#isolar-backend-zabbix)
* [Atualizar IP Dinâmico via DNS](#atualizar-ip-dinâmico-via-dns)
* [Configurar Logs de Tentativas Negadas](#configurar-logs-de-tentativas-negadas)
* [Enganação Silenciosa de Atacantes](#enganação-silenciosa-de-atacantes)
* [Conclusão](#conclusão)

---

## 📌 Requisitos

* Ubuntu/Debian com Nginx
* Zabbix rodando localmente na porta `8080`
* Domínio válido (ex: `zabbix.seudominio.com`)
* Certbot para HTTPS (Let's Encrypt)

Consulte o [manual completo aqui]((https://github.com/danielselbachoficial/infra-hardening-guides/blob/main/zabbix-nginx/zabbix-nginx-hardening.md)).

