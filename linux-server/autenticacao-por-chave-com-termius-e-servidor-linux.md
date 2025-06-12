# 🚀 Manual de Configuração SSH: Autenticação por Chave com Termius e Servidor Linux

Este manual detalha o processo de configuração da autenticação SSH baseada em chaves, uma alternativa mais segura e conveniente do que a autenticação por senha. Abordaremos a geração de chaves, sua instalação em um servidor Linux e a configuração no Termius, um cliente SSH popular, incluindo a resolução de problemas comuns de permissão e propriedade de arquivos.

---

## 📋 Sumário

Um sumário é crucial para documentos longos, permitindo que os leitores naveguem facilmente. Lembre-se de que os links devem corresponder exatamente aos títulos (minúsculas, hífens para espaços, sem acentos).

*   [Introdução](#introdução)
*   [Pré-requisitos](#pré-requisitos)
*   [Passo 1: Gerar o Par de Chaves SSH (Máquina Local)](#passo-1-gerar-o-par-de-chaves-ssh-máquina-local)
*   [Passo 2: Adicionar a Chave Pública ao Servidor (Servidor Remoto)](#passo-2-adicionar-a-chave-pública-ao-servidor-servidor-remoto)
*   [Passo 3: Configurar Permissões e Propriedade no Servidor (Crucial!)](#passo-3-configurar-permissões-e-propriedade-no-servidor-crucial)
*   [Passo 4: Adicionar a Chave Privada ao Termius (Máquina Local)](#passo-4-adicionar-a-chave-privada-ao-termius-máquina-local)
*   [Passo 5: Configurar o Host no Termius](#passo-5-configurar-o-host-no-termius)
*   [Passo 6: Testar a Conexão](#passo-6-testar-a-conexão)
*   [Resolução de Problemas Comuns](#resolução-de-problemas-comuns)
*   [Boas Práticas de Segurança](#boas-práticas-de-segurança)
*   [Conclusão](#conclusão)

---

## Introdução

A autenticação por chave SSH é o método preferido para conectar-se a servidores remotos por sua segurança e conveniência. Em vez de uma senha, que pode ser adivinhada ou forçada, você usa um par de chaves criptográficas: uma **chave privada** (mantida em segredo na sua máquina local) e uma **chave pública** (armazenada no servidor).

Este guia irá cobrir:
*   A geração segura de chaves SSH.
*   A instalação correta da chave pública no servidor Linux.
*   A configuração de permissões e propriedade de arquivos, que são frequentemente a causa de falhas na autenticação por chave.
*   A configuração do cliente Termius para usar a chave privada.

---

## Pré-requisitos

Para seguir este manual, você precisará dos seguintes itens:

*   **Máquina Local:** Um computador com Termius instalado (ou outro cliente SSH que suporte chaves) e acesso a um terminal (PowerShell no Windows, Terminal no macOS/Linux).
*   **Servidor Remoto:** Um servidor Linux com acesso SSH (inicialmente, por senha ou console).
*   **Acesso `sudo`:** Capacidade de executar comandos com privilégios de superusuário no servidor remoto.

---

## Passo 1: Gerar o Par de Chaves SSH (Máquina Local)

Se você já possui um par de chaves e deseja reutilizá-lo, pode pular para o [Passo 4](#passo-4-adicionar-a-chave-privada-ao-termius-máquina-local).

1.  **Abra o Terminal** (ou PowerShell, CMD) na sua máquina local.

2.  Execute o comando `ssh-keygen` para gerar um novo par de chaves:
    ```bash
    ssh-keygen -t rsa -b 4096 -C "seu_email@example.com"
    ```
    *   `-t rsa`: Especifica o tipo de algoritmo da chave (RSA é um padrão seguro).
    *   `-b 4096`: Define o tamanho da chave em bits (4096 é recomendado para maior segurança).
    *   `-C "seu_email@example.com"`: Adiciona um comentário à chave, útil para identificá-la posteriormente.

3.  **Local para salvar a chave:** O `ssh-keygen` perguntará onde salvar a chave. O local padrão (`~/.ssh/id_rsa`) é geralmente o melhor. Pressione `Enter` para aceitar o padrão.
    ```
    Enter file in which to save the key (/home/seu_usuario/.ssh/id_rsa): [Pressione Enter]
    ```

4.  **Frase de segurança (passphrase):** Você será solicitado a criar uma frase de segurança. **É ALTAMENTE RECOMENDADO** que você use uma passphrase forte. Ela adiciona uma camada extra de segurança à sua chave privada, protegendo-a mesmo que ela caia em mãos erradas.
    ```
    Enter passphrase (empty for no passphrase): [Digite sua passphrase]
    Enter same passphrase again: [Digite novamente]
    ```
    > **Aviso:** Se você não digitar nada e apenas pressionar `Enter`, a chave privada não terá passphrase (menos seguro).

Após a geração, você terá dois arquivos no diretório `~/.ssh/`:
*   `id_rsa` (ou o nome que você deu): **Sua chave privada.** Mantenha-a em sigilo absoluto e **NUNCA a compartilhe**.
*   `id_rsa.pub` (ou o nome que você deu com `.pub`): **Sua chave pública.** Esta é a chave que você copiará para o servidor.

---

## Passo 2: Adicionar a Chave Pública ao Servidor (Servidor Remoto)

Agora, você precisa copiar o conteúdo da sua chave pública (`id_rsa.pub`) para o servidor remoto, no arquivo `~/.ssh/authorized_keys` do usuário com o qual você deseja se conectar.

1.  **Conecte-se ao seu servidor remoto** usando SSH, via senha ou console (pela primeira vez ou se a chave ainda não funcionar).
    ```bash
    ssh seu_usuario@endereco_ip_do_servidor
    ```

2.  **Crie o diretório `.ssh` no servidor (se não existir):**
    ```bash
    mkdir -p ~/.ssh
    ```
    *   O `-p` garante que o diretório será criado se não existir, e não dará erro se já existir.

3.  **Adicione sua chave pública ao arquivo `authorized_keys`:**

    *   **Método 1: Copiar e Colar (recomendado se `ssh-copy-id` não estiver disponível):**
        1.  Na **sua máquina local**, visualize o conteúdo da sua chave pública:
            ```bash
            cat ~/.ssh/id_rsa.pub
            ```
        2.  Copie a saída **completa** (tudo que aparecer, começando com `ssh-rsa AAAA...` e terminando com seu comentário de e-mail).
        3.  No **servidor remoto**, edite o arquivo `authorized_keys` (ou crie-o se não existir).
            ```bash
            nano ~/.ssh/authorized_keys
            # Ou use vi, vim, etc.
            ```
        4.  Cole a chave pública copiada. Certifique-se de que a chave esteja em **uma única linha**, sem quebras de linha. Se já houver outras chaves, adicione a sua em uma nova linha.
        5.  Salve o arquivo (No `nano`: `Ctrl+O`, `Enter`, `Ctrl+X`).

    *   **Método 2: Usar `ssh-copy-id` (mais fácil e recomendado):**
        *   Na **sua máquina local**, se você tiver o `ssh-copy-id` instalado (comum em Linux/macOS):
            ```bash
            ssh-copy-id -i ~/.ssh/id_rsa.pub seu_usuario@endereco_ip_do_servidor
            ```
        *   Ele pedirá a senha do usuário no servidor e copiará a chave com as permissões corretas.

---

## Passo 3: Configurar Permissões e Propriedade no Servidor (Crucial!)

Este é um dos pontos mais comuns de falha na autenticação por chave. O SSH é **extremamente rigoroso** com as permissões e a propriedade do diretório `~/.ssh` e do arquivo `authorized_keys`. Se elas estiverem incorretas, o SSH se recusará a usar a chave por motivos de segurança, resultando em mensagens como "Permission denied".

Conecte-se ao servidor (via senha ou console) e execute os seguintes comandos como o usuário que você está configurando (ou com `sudo`):

1.  **Garantir a propriedade correta:**
    O diretório `.ssh` e todos os arquivos dentro dele devem pertencer ao usuário que está tentando se conectar (e **NÃO** ao `root` ou a outro usuário).
    ```bash
    sudo chown -R seu_usuario:seu_usuario /home/seu_usuario/.ssh
    ```
    *   Substitua `seu_usuario` pelo nome de usuário real no servidor (ex: `afsim-tech`).

2.  **Definir permissões restritivas para o diretório `.ssh`:**
    Apenas o proprietário (`seu_usuario`) deve ter permissão total (leitura, escrita e execução).
    ```bash
    chmod 700 ~/.ssh
    ```

3.  **Definir permissões restritivas para o arquivo `authorized_keys`:**
    Apenas o proprietário (`seu_usuario`) deve ter permissão de leitura e escrita.
    ```bash
    chmod 600 ~/.ssh/authorized_keys
    ```

4.  **Definir permissões para o diretório home do usuário (opcional, mas recomendado):**
    O diretório home do usuário não deve ter permissões de escrita para outros. `755` é um padrão comum e seguro.
    ```bash
    chmod 755 ~
    ```

5.  **Verificar as permissões e propriedade (para confirmar):**
    É sempre uma boa prática verificar se as permissões foram aplicadas corretamente.
    ```bash
    ls -ld ~ ~/.ssh ~/.ssh/authorized_keys
    ```
    A saída deve ser similar a:
    ```
    drwxr-xr-x 5 seu_usuario seu_usuario 4096 Mar 10 10:00 /home/seu_usuario
    drwx------ 2 seu_usuario seu_usuario 4096 Mar 10 10:00 /home/seu_usuario/.ssh
    -rw------- 1 seu_usuario seu_usuario  400 Mar 10 10:00 /home/seu_usuario/.ssh/authorized_keys
    ```
    Note que agora `.ssh` e `authorized_keys` devem ter `seu_usuario seu_usuario` como proprietário e grupo, com as permissões restritivas indicadas.

6.  **Reiniciar o serviço SSH no servidor:**
    Após qualquer alteração de permissão ou configuração, é essencial reiniciar o serviço SSH para que ele releia as configurações e as novas permissões.
    ```bash
    sudo systemctl restart sshd # Para sistemas baseados em systemd (Ubuntu, CentOS 7+, Debian 8+)
    # Ou
    sudo service sshd restart   # Para sistemas mais antigos
    ```

---

## Passo 4: Adicionar a Chave Privada ao Termius (Máquina Local)

Agora que sua chave pública está no servidor e as permissões estão corretas, você precisa adicionar a chave privada ao Termius.

1.  **Abra o Termius.**
2.  No menu lateral esquerdo, vá para a seção **"Keys"**.
3.  Clique no botão **"+"** para adicionar uma nova chave.
4.  Selecione a opção **"Import"**.
5.  Navegue até o local onde sua chave privada foi salva (geralmente `~/.ssh/id_rsa`).
6.  Selecione o arquivo da chave privada (o que **NÃO** tem a extensão `.pub`).
7.  Se sua chave privada tiver uma passphrase, o Termius pedirá que você a digite para desbloqueá-la.
8.  Dê um nome amigável para sua chave no Termius (ex: "Minha Chave Pessoal", "Chave do Servidor Grafana").
9.  Clique em "Save" (ou botão equivalente).

---

## Passo 5: Configurar o Host no Termius

Com a chave privada importada, vamos configurar o Termius para usá-la ao conectar-se ao servidor.

1.  No Termius, vá para a seção **"Hosts"**.
2.  **Edite um host existente** (clicando no nome dele) ou crie um **"New Host"**.
3.  **Preencha os detalhes básicos do seu servidor:**
    *   **Alias:** Um nome fácil de lembrar para o host (ex: "Servidor Grafana").
    *   **Address:** O endereço IP ou nome de domínio do seu servidor (ex: `192.168.1.100` ou `grafana.meudominio.com`).
    *   **Port:** A porta SSH (o padrão é `22`). Se você usa uma porta diferente no servidor, especifique-a aqui.
    *   **Username:** O nome de usuário no servidor remoto (ex: `afsim-tech`).

4.  **Configurar Autenticação:**
    *   Role para baixo até a seção **"Authentication"**.

    *   **Método para Teste Inicial (Recomendado):**
        *   Selecione o método **"Password"**.
        *   Digite a senha do seu usuário no servidor no campo "Password".
        *   **Crucial:** No campo **"Key"** (ou "Identity Key"), clique e selecione a chave privada que você acabou de importar (ex: "Minha Chave Pessoal").
        *   *Com esta configuração, o Termius tentará primeiro a autenticação por chave. Se essa falhar por qualquer motivo (como configuração incorreta no servidor), ele então tentará a autenticação por senha.*

    *   **Método para Produção (Após Teste com Sucesso):**
        *   Uma vez que a autenticação por chave esteja funcionando perfeitamente, e você tenha **desabilitado a senha no servidor** (veja [Boas Práticas de Segurança](#boas-práticas-de-segurança)), você pode mudar o método para **"Key"** e apenas selecionar a chave, sem precisar preencher a senha.

5.  **Salve** as alterações no host.

---

## Passo 6: Testar a Conexão

Agora é a hora da verdade!

1.  No Termius, clique no host que você acabou de configurar para tentar a conexão.
2.  **Se tudo estiver correto e as chaves funcionarem:** Você deverá ser conectado ao servidor sem ser solicitado a digitar a senha (se sua chave tiver uma passphrase, o Termius pedirá a passphrase da chave, e não a senha do servidor).
3.  **Se ainda pedir senha (e a chave tem passphrase):** Certifique-se de que você está digitando a **passphrase da sua chave**, e e não a senha do usuário do servidor.
4.  **Se ainda pedir senha (e a chave NÃO tem passphrase) ou der algum erro:** Prossiga para a próxima seção de resolução de problemas.

---

## Resolução de Problemas Comuns

Se a autenticação por chave ainda estiver falhando, os logs do servidor SSH são seus melhores amigos para diagnosticar o problema.

1.  **Verificar Logs do Servidor SSH:**
    *   Conecte-se ao servidor via senha ou console.
    *   Execute o comando para monitorar os logs em tempo real:
        ```bash
        sudo journalctl -u ssh.service -f
        # Ou, para sistemas mais antigos ou fallback:
        tail -f /var/log/auth.log
        ```
    *   Mantenha essa janela do terminal aberta.
    *   Na sua máquina local (Termius), tente conectar-se novamente.
    *   **Observe as mensagens que aparecem no terminal do servidor.** Elas indicarão o motivo exato da falha (ex: `Permission denied`, `Authentication failed`, `Invalid format`, etc.). Esta é a dica mais valiosa!

2.  **`Permission denied`:**
    *   **Causa:** Quase sempre um problema de permissões ou propriedade de arquivos. Isso significa que o servidor SSH não pode ler o arquivo `authorized_keys` ou o diretório `.ssh` porque as permissões são muito abertas ou o proprietário não é o usuário correto.
    *   **Solução:** Volte ao [Passo 3](#passo-3-configurar-permissões-e-propriedade-no-servidor-crucial) e **re-verifique/re-aplique** os comandos `chown` e `chmod`. As permissões devem ser `700` para `~/.ssh` e `600` para `~/.ssh/authorized_keys`, e ambos devem pertencer ao usuário que está tentando se conectar.

3.  **Chave Pública Incorreta ou Mal Formatada:**
    *   **Causa:** A chave pública no `authorized_keys` pode estar incompleta, ter quebras de linha, espaços extras, ou não corresponder à chave privada que você está usando.
    *   **Solução:**
        *   Na sua máquina local, execute `cat ~/.ssh/id_rsa.pub` e copie a saída **exatamente como está**.
        *   No servidor, abra `~/.ssh/authorized_keys` e certifique-se de que a chave está em **uma única linha**, sem caracteres extras. Se houver dúvidas, apague a linha existente e cole a chave novamente.

4.  **Chave Privada Incorreta no Termius:**
    *   **Causa:** Você pode ter selecionado a chave errada no Termius, ou importou a chave pública em vez da privada para o cliente.
    *   **Solução:** No Termius, em "Keys", verifique se você importou o arquivo **sem** `.pub` na extensão. No host, em "Authentication", certifique-se de que a chave correta está selecionada.

5.  **Configuração do Servidor SSH (`sshd_config`):**
    *   **Causa:** Em casos raros, a autenticação por chave pode estar desabilitada no próprio servidor ou as configurações são muito restritivas.
    *   **Solução:** No servidor, edite `/etc/ssh/sshd_config` (com `sudo`) e certifique-se de que as seguintes linhas estão presentes e descomentadas (sem `#` no início):
        ```
        PubkeyAuthentication yes
        AuthorizedKeysFile     .ssh/authorized_keys
        ```
    *   Após alterar, reinicie o serviço SSH: `sudo systemctl restart sshd`.

---

## Boas Práticas de Segurança

Adotar as seguintes boas práticas elevará ainda mais a segurança do seu acesso SSH:

*   **Sempre use uma Passphrase:** Adicionar uma passphrase à sua chave privada oferece uma camada extra de segurança fundamental. Mesmo que alguém obtenha sua chave privada, ela será inútil sem a passphrase.
*   **Desabilitar Autenticação por Senha (para ambientes de produção):** Uma vez que a autenticação por chave SSH esteja funcionando perfeitamente, é altamente recomendável desabilitar a autenticação por senha no servidor. Isso impede ataques de força bruta.
    *   Edite `/etc/ssh/sshd_config` no servidor e altere: `PasswordAuthentication yes` para `PasswordAuthentication no`.
    *   Reinicie o serviço SSH (`sudo systemctl restart sshd`).
    *   **ATENÇÃO:** Só faça isso após testar exaustivamente a conexão por chave! Tenha um método de acesso alternativo (como console de nuvem) caso algo dê errado e você perca o acesso.
*   **Gerencie Suas Chaves:** Não use a mesma chave para todos os servidores. Considere ter chaves diferentes para diferentes contextos ou clientes, facilitando a revogação de acesso se uma chave for comprometida.
*   **`ssh-agent`:** Em sistemas Linux/macOS, o `ssh-agent` pode armazenar suas chaves descriptografadas na memória, permitindo que você digite a passphrase apenas uma vez por sessão.

---

## Conclusão

A autenticação por chave SSH é um pilar da segurança em servidores Linux. Dominar sua configuração, incluindo o entendimento das permissões e propriedade de arquivos, é uma habilidade fundamental para qualquer administrador de sistemas ou desenvolvedor. Com este guia, você deve estar apto a configurar suas conexões SSH de forma segura e eficiente.

---

## 📚 Recursos Adicionais

*   [Documentação Oficial do GitHub Markdown](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
*   [Documentação Oficial do SSH](https://www.ssh.com/ssh/protocol/)
*   [Termius - Documentação Oficial](https://termius.com/docs/)

---

**Desenvolvido por *Daniel Selbach da AFSIM TECH" (https://github.com/danielselbachoficial)**
