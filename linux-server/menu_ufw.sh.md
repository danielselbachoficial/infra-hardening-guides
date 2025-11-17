# Script de Gerenciamento Interativo para UFW

Este repositório contém um script de shell (`menu_ufw.sh`) projetado para simplificar e agilizar o gerenciamento do **UFW (Uncomplicated Firewall)** em servidores baseados em Debian/Ubuntu.

O script oferece uma interface de menu amigável e segura, ideal tanto para iniciantes que precisam de uma forma mais segura de interagir com o firewall, quanto para administradores de sistemas que buscam acelerar tarefas do dia a dia.

## Recursos Principais

-   ✅ **Interface de Menu Intuitiva**: Navegue por opções claras para adicionar/deletar regras e controlar o firewall.
-   🚀 **Cabeçalho Dinâmico**: O status do firewall (**Ativo**/**Inativo**) e a **contagem de regras** são exibidos em tempo real no topo da tela.
-   🛡️ **Segurança em Primeiro Lugar**: Pede confirmação para ações destrutivas (como deletar regras) para evitar erros.
-   ⚙️ **Gerenciamento Completo**: Permite não apenas gerenciar regras, mas também ativar e desativar o serviço do UFW.
-   🎨 **Saída Colorida**: Utiliza cores para diferenciar avisos, sucessos e erros, melhorando a legibilidade.

## Pré-requisitos

-   Um shell `bash` (padrão na maioria dos sistemas Linux).
-   Privilégios de `sudo`.
-   O pacote `ufw` instalado no sistema. Se não tiver, instale com:
    ```bash
    sudo apt update && sudo apt install ufw
    ```

## Como Usar

1.  **Crie o arquivo do script** chamado `menu_ufw.sh` e cole o script que está em `Aparência do Menu`.
    ```bash
    sudo nano menu_ufw.sh
    ```

2.  **Dê permissão de execução** ao arquivo:
    ```bash
    sudo chmod +x menu_ufw.sh
    ```

3.  **Execute o script** com `sudo`:
    ```bash
    sudo ./menu_ufw.sh
    ```

### Aparência do Menu
```
#!/bin/bash

# =================================================================================
# Script com Menu Interativo e Cabeçalho Dinâmico para Gerenciar o Firewall UFW
# =================================================================================
#
# DESCRIÇÃO:
# Este script fornece uma interface de menu para gerenciar o UFW. 
#
# =================================================================================

VERDE='\033[0;32m'
AMARELO='\033[1;33m'
CIANO='\033[0;36m'
VERMELHO='\033[0;31m'
SEM_COR='\033[0m'

cabecalho() {
    clear
    local status="Inativo"
    local cor_status="${VERMELHO}"
    if command -v ufw >/dev/null 2>&1; then
        if sudo ufw status | grep -q "Status: active"; then
            status="Ativo"
            cor_status="${VERDE}"
        fi
    fi
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "${CIANO}   Gerenciador de Regras do Firewall UFW          ${SEM_COR}"
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "  Status: ${cor_status}${status}${SEM_COR}"
    echo
}

validar_ip_ou_subnet() {
    local entrada="$1"
    if [[ "$entrada" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
        return 0
    fi
    if [[ "$entrada" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}/[0-9]{1,2}$ ]]; then
        return 0
    fi
    return 1
}

validar_porta() {
    local porta="$1"
    if [[ "$porta" =~ ^[0-9]+$ ]] && [ "$porta" -ge 1 ] && [ "$porta" -le 65535 ]; then
        return 0
    fi
    return 1
}

opcao_1() {
    clear
    echo -e "${AMARELO}--- Liberar acesso SSH ---${SEM_COR}"
    echo
    echo -n "IP ou Sub-rede (ex: 192.168.1.100 ou 192.168.1.0/24): "
    read -r ip_subnet
    if ! validar_ip_ou_subnet "$ip_subnet"; then
        echo -e "${VERMELHO}IP ou sub-rede inválido${SEM_COR}"
        return 1
    fi
    echo -n "Porta (padrão 22): "
    read -r porta
    porta=${porta:-22}
    if ! validar_porta "$porta"; then
        echo -e "${VERMELHO}Porta inválida${SEM_COR}"
        return 1
    fi
    sudo ufw allow from "$ip_subnet" to any port "$porta" proto tcp
    echo -e "${VERDE}Regra adicionada para $ip_subnet!${SEM_COR}"
}

opcao_2() {
    clear
    echo -e "${AMARELO}--- Liberar acesso a Serviço ---${SEM_COR}"
    echo
    echo -n "IP ou Sub-rede (ex: 192.168.1.100 ou 192.168.1.0/24): "
    read -r ip_subnet
    if ! validar_ip_ou_subnet "$ip_subnet"; then
        echo -e "${VERMELHO}IP ou sub-rede inválido${SEM_COR}"
        return 1
    fi
    echo -n "Porta: "
    read -r porta
    if ! validar_porta "$porta"; then
        echo -e "${VERMELHO}Porta inválida${SEM_COR}"
        return 1
    fi
    echo -n "Protocolo (tcp/udp): "
    read -r proto
    proto=${proto:-tcp}
    sudo ufw allow from "$ip_subnet" to any port "$porta" proto "$proto"
    echo -e "${VERDE}Regra adicionada para $ip_subnet!${SEM_COR}"
}

opcao_3() {
    clear
    echo -e "${AMARELO}--- Deletar regra ---${SEM_COR}"
    echo
    sudo ufw status numbered
    echo
    echo -n "Número (0=cancelar): "
    read -r num
    if [ "$num" -eq 0 ] 2>/dev/null; then
        return 0
    fi
    echo -n "Confirma? [s/N]: "
    read -r conf
    if [ "$conf" = "s" ]; then
        sudo ufw --force delete "$num"
        echo -e "${VERDE}Deletado!${SEM_COR}"
    fi
}

opcao_4() {
    while true; do
        clear
        echo -e "${AMARELO}--- Verificar Instalação do UFW ---${SEM_COR}"
        echo

        if command -v ufw >/dev/null 2>&1; then
            echo -e "${VERDE}✓ UFW está instalado${SEM_COR}"
            echo
            echo -e "${CIANO}Informações do UFW:${SEM_COR}"
            echo
            sudo ufw status verbose
            echo
            echo -e "${CIANO}Versão:${SEM_COR}"
            echo
            echo -e "${CIANO}Localização:${SEM_COR}"
            command -v ufw
            echo
        else
            echo -e "${VERMELHO}✗ UFW não está instalado${SEM_COR}"
            echo
            echo -e "${AMARELO}Opções de instalação:${SEM_COR}"
            echo

            if command -v apt-get >/dev/null 2>&1; then
                echo -e "  Sistema: ${CIANO}Debian/Ubuntu${SEM_COR}"
                echo -e "  Comando: ${VERDE}sudo apt-get install ufw${SEM_COR}"
                echo
            elif command -v yum >/dev/null 2>&1; then
                echo -e "  Sistema: ${CIANO}RedHat/CentOS${SEM_COR}"
                echo -e "  Comando: ${VERDE}sudo yum install ufw${SEM_COR}"
                echo
            elif command -v dnf >/dev/null 2>&1; then
                echo -e "  Sistema: ${CIANO}Fedora${SEM_COR}"
                echo -e "  Comando: ${VERDE}sudo dnf install ufw${SEM_COR}"
                echo
            else
                echo -e "${AMARELO}Instale UFW através do gerenciador de pacotes da sua distribuição${SEM_COR}"
                echo
            fi
        fi

        echo -e "${AMARELO}======== Menu de Verificação ========${SEM_COR}"
        echo -e "  ${VERDE}I${SEM_COR}) Instalar UFW"
        echo -e "  ${VERDE}R${SEM_COR}) Recarregar/Atualizar"
        echo -e "  ${VERDE}V${SEM_COR}) Voltar ao menu principal"
        echo -e "${AMARELO}=====================================${SEM_COR}"
        echo
        echo -n "Opção: "
        read -r opcao_inst

        case "$opcao_inst" in
            i|I)
                echo
                echo -e "${CIANO}[*] Tentando instalar UFW...${SEM_COR}"
                echo

                if command -v apt-get >/dev/null 2>&1; then
                    echo -e "${CIANO}[*] Atualizando cache de pacotes...${SEM_COR}"
                    sudo apt-get update
                    echo
                    echo -e "${CIANO}[*] Instalando UFW...${SEM_COR}"
                    sudo apt-get install -y ufw
                    echo
                    echo -e "${VERDE}✓ UFW instalado com sucesso!${SEM_COR}"
                    sleep 2
                elif command -v yum >/dev/null 2>&1; then
                    echo -e "${CIANO}[*] Instalando UFW...${SEM_COR}"
                    sudo yum install -y ufw
                    echo
                    echo -e "${VERDE}✓ UFW instalado com sucesso!${SEM_COR}"
                    sleep 2
                elif command -v dnf >/dev/null 2>&1; then
                    echo -e "${CIANO}[*] Instalando UFW...${SEM_COR}"
                    sudo dnf install -y ufw
                    echo
                    echo -e "${VERDE}✓ UFW instalado com sucesso!${SEM_COR}"
                    sleep 2
                else
                    echo -e "${VERMELHO}✗ Gerenciador de pacotes não suportado${SEM_COR}"
                    sleep 2
                fi
                ;;
            r|R)
                continue
                ;;
            v|V)
                return 0
                ;;
            *)
                echo -e "${VERMELHO}Opção inválida${SEM_COR}"
                sleep 1
                ;;
        esac
    done
}

opcao_5() {
    clear
    echo -e "${AMARELO}--- Ativar UFW ---${SEM_COR}"
    echo
    sudo ufw --force enable
    echo -e "${VERDE}Ativado!${SEM_COR}"
}

opcao_6() {
    clear
    echo -e "${AMARELO}--- Desativar UFW ---${SEM_COR}"
    echo
    echo -e "${VERMELHO}ATENÇÃO: Isso expõe seu sistema!${SEM_COR}"
    echo -n "Confirma? [s/N]: "
    read -r resp
    if [ "$resp" = "s" ]; then
        sudo ufw disable
        echo -e "${VERDE}Desativado${SEM_COR}"
    fi
}

opcao_7() {
    while true; do
        clear
        echo -e "${AMARELO}--- Regras UFW Ativas ---${SEM_COR}"
        echo

        if ! command -v ufw >/dev/null 2>&1; then
            echo -e "${VERMELHO}UFW não está instalado${SEM_COR}"
            echo
            echo -n "Pressione ENTER para voltar ao menu principal..."
            read -r
            return 1
        fi

        echo -e "${CIANO}[Regras numeradas - para deletar]:${SEM_COR}"
        echo
        sudo ufw status numbered
        echo
        echo -e "${CIANO}[Regras em formato padrão]:${SEM_COR}"
        echo
        sudo ufw status
        echo
        echo -e "${CIANO}[Status verboso]:${SEM_COR}"
        echo
        sudo ufw status verbose
        echo
        echo -e "${AMARELO}======== Menu de Visualização ========${SEM_COR}"
        echo -e "  ${VERDE}R${SEM_COR}) Recarregar/Atualizar"
        echo -e "  ${VERDE}V${SEM_COR}) Voltar ao menu principal"
        echo -e "${AMARELO}=====================================${SEM_COR}"
        echo
        echo -n "Opção: "
        read -r opcao_viz

        case "$opcao_viz" in
            r|R)
                continue
                ;;
            v|V)
                return 0
                ;;
            *)
                echo -e "${VERMELHO}Opção inválida${SEM_COR}"
                sleep 1
                ;;
        esac
    done
}

main() {
    while true; do
        cabecalho
        echo "Menu:"
        echo -e "  ${AMARELO}1)${SEM_COR} Liberar SSH"
        echo -e "  ${AMARELO}2)${SEM_COR} Liberar Serviço"
        echo -e "  ${AMARELO}3)${SEM_COR} Deletar regra"
        echo -e "  ${AMARELO}4)${SEM_COR} Verificar instalação"
        echo -e "  ${AMARELO}5)${SEM_COR} Ativar UFW"
        echo -e "  ${AMARELO}6)${SEM_COR} Desativar UFW"
        echo -e "  ${AMARELO}7)${SEM_COR} Visualizar regras"
        echo -e "  ${AMARELO}8)${SEM_COR} Sair"
        echo
        echo -n "Opção [1-8]: "
        read -r opcao
        case "$opcao" in
            1) opcao_1; sleep 2 ;;
            2) opcao_2; sleep 2 ;;
            3) opcao_3; sleep 2 ;;
            4) opcao_4 ;;
            5) opcao_5; sleep 2 ;;
            6) opcao_6; sleep 2 ;;
            7) opcao_7 ;;
            8) echo -e "${VERDE}Saindo...${SEM_COR}"; exit 0 ;;
            *) echo -e "${VERMELHO}Inválido${SEM_COR}"; sleep 1 ;;
        esac
    done
}

main "$@"
```
