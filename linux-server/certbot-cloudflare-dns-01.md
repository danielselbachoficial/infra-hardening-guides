# Guia: Configuração do Certbot com Desafio DNS-01 e Cloudflare

Este guia detalhado demonstra como configurar o Certbot para obter e renovar certificados SSL/TLS usando o desafio DNS-01, especificamente com o Cloudflare como provedor DNS. Esta abordagem é ideal para servidores que não expõem a porta 80 ou para cenários onde a validação HTTP-01 não é viável, como para instâncias do Grafana.

---

## Pré-requisitos

*   Uma conta ativa no Cloudflare com o seu domínio (`seudominio.com.br` no exemplo) gerenciado por ele.
*   Acesso SSH ao seu servidor Linux (onde o Certbot e Grafana estão instalados) com privilégios `sudo`.
*   Certbot já instalado no seu servidor.

---

## Passo 1: Gerar um Token de API Específico no Cloudflare

**IMPORTANTE:** Nunca use sua Chave de API Global do Cloudflare. Ela concede acesso irrestrito à sua conta. Em vez disso, crie um Token de API com permissões mínimas essenciais para o Certbot.

1.  **Acesse o Cloudflare:** Faça login na sua conta do Cloudflare.
2.  **Navegue até "My Profile" (Meu Perfil):** Clique no ícone do seu perfil no canto superior direito e selecione "My Profile".
3.  **Vá para "API Tokens":** Na barra lateral esquerda, clique em "API Tokens".
4.  **Crie um Token Personalizado:**
    *   Clique em "Create Token".
    *   Role para baixo até a seção "Custom token" e clique em "Get started".
5.  **Configure as Permissões do Token:**
    *   **Token name:** Dê um nome descritivo para o token, como `Certbot_Grafana_Renewal` ou `Certbot_DNS_afsimtech`.
    *   **Permissions:** Adicione as seguintes permissões para garantir que o Certbot possa ler e modificar registros DNS:
        *   `Zone` -> `Zone` -> `Read`
        *   `Zone` -> `DNS` -> `Edit`
    *   **Zone Resources:**
        *   Selecione `Specific zone` (Zona específica).
        *   No campo de busca, digite e selecione seu domínio, por exemplo, `seudominio.com.br`.
        *   **Isso é crucial:** Garante que o token só possa interagir com os registros DNS desse domínio específico.
    *   **Client IP Address Filtering (Opcional, mas Recomendado):** Se o seu servidor Grafana possui um IP estático, adicione o IP público do seu servidor aqui. Isso restringe o uso do token apenas a requisições originadas desse IP, aumentando a segurança.
6.  **Continue para Sumário:** Clique em "Continue to summary".
7.  **Crie o Token:** Clique em "Create token".
8.  **Copie o Token:** O Cloudflare exibirá o token. **COPIE-O IMEDIATAMENTE**, pois ele não será exibido novamente. Guarde-o em um local seguro temporariamente (um bloco de notas, por exemplo), você precisará dele no próximo passo.

---

## Passo 2: Configurar o Certbot no Servidor

Agora, vamos instalar o plugin do Certbot para Cloudflare e configurar as credenciais de API no seu servidor.

1.  **Instalar o Plugin Certbot DNS-Cloudflare:**
    ```bash
    sudo apt update
    sudo apt install python3-certbot-dns-cloudflare
    ```

2.  **Criar o Arquivo de Credenciais do Cloudflare:**
    Este arquivo armazenará seu token de API do Cloudflare. É **CRUCIAL** que ele tenha permissões restritas para que apenas o `root` (e o Certbot quando executado com `sudo`) possa lê-lo.

    ```bash
    sudo nano /etc/letsencrypt/cloudflare.ini
    ```
    Cole o seguinte conteúdo no arquivo, substituindo `SEU_CLOUDFLARE_API_TOKEN_AQUI` pelo token que você gerou no Passo 1:
    ```ini
    dns_cloudflare_api_token = SEU_CLOUDFLARE_API_TOKEN_AQUI
    ```
    Salve e feche o arquivo (`Ctrl+X`, depois `Y` para confirmar e `Enter`).

3.  **Definir Permissões Restritas para o Arquivo de Credenciais:**
    ```bash
    sudo chmod 600 /etc/letsencrypt/cloudflare.ini
    ```
    Este comando garante que apenas o usuário `root` (e o Certbot com `sudo`) possa ler este arquivo, protegendo seu token de API.

4.  **Atualizar a Configuração de Renovação Existente do Certificado:**
    Precisamos informar ao Certbot para usar o plugin DNS-Cloudflare para futuras renovações do certificado do seu domínio (ex: `grafana.seudominio.com.br`).

    ```bash
    sudo nano /etc/letsencrypt/renewal/grafana.seudominio.com.br.conf
    ```
    Procure a seção `[renewalparams]` e faça as seguintes alterações/adições:
    *   Altere a linha `authenticator = standalone` para `authenticator = dns-cloudflare`.
    *   Adicione a linha `dns_cloudflare_credentials = /etc/letsencrypt/cloudflare.ini`.
    *   **(Opcional, mas recomendado)** Adicione `dns_cloudflare_propagation_seconds = 30`. Embora o Cloudflare seja rápido, esta margem de 30 segundos garante que o registro TXT se propague antes do Let's Encrypt verificar.

    A seção `[renewalparams]` deverá ficar parecida com isto (ajuste se tiver outras linhas):
    ```ini
    [renewalparams]
    account = acbbd81ec9647d29bb63252a5192150d
    authenticator = dns-cloudflare
    server = https://acme-v02.api.letsencrypt.org/directory
    key_type = ecdsa
    dns_cloudflare_credentials = /etc/letsencrypt/cloudflare.ini
    dns_cloudflare_propagation_seconds = 30
    ```
    Salve e feche o arquivo.

---

## Passo 3: Testar a Renovação com Dry Run (DNS-01)

Com tudo configurado, realize um "dry-run" (simulação) para garantir que o processo de renovação funcione corretamente com o desafio DNS-01 usando o Cloudflare.

```bash
sudo certbot renew --dry-run
```

O que deve acontecer:

O Certbot tentará se comunicar com o Cloudflare via API, utilizando o token configurado em /etc/letsencrypt/cloudflare.ini.
Ele criará um registro TXT temporário no seu DNS no Cloudflare.
O Let's Encrypt verificará a presença desse registro TXT para validar a posse do domínio.
Após a validação (ou falha), o Certbot removerá o registro TXT.
Saída Esperada para Sucesso:

Se tudo correr bem, a saída final deverá ser similar a:

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations, all simulated renewals succeeded:
/etc/letsencrypt/live/grafana.seudominio.com.br/fullchain.pem (success)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

Se o dry-run for bem-sucedido, significa que sua configuração para renovação com DNS-01 via Cloudflare está funcionando perfeitamente! As renovações automáticas (geralmente via cron ou systemd timer configurado pelo Certbot) não terão mais problemas com a porta 80.

Próximos Passos
Após a validação bem-sucedida do dry-run, seu certificado será renovado automaticamente pelo Certbot.

Você pode verificar o cronjob ou systemd timer do Certbot para entender a frequência das renovações automáticas (geralmente duas vezes ao dia).
Para systemd: sudo systemctl list-timers | grep certbot
Para cron: sudo cat /etc/cron.d/certbot ou sudo crontab -l
Monitore os logs do Certbot para quaisquer erros futuros durante as renovações: sudo tail -f /var/log/letsencrypt/letsencrypt.log
Este guia fornece uma solução robusta e segura para gerenciar seus certificados SSL/TLS com o Certbot e Cloudflare.
