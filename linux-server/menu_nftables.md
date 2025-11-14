#!/bin/bash

# =================================================================================
# Script com Menu Interativo para Gerenciar o Firewall nftables
# =================================================================================
#
# DESCRIÇÃO:
# Este script fornece uma interface de menu para gerenciar o nftables, o framework
# de firewall moderno do Linux. Ele facilita a adição e remoção de regras,
# o controle do serviço e o salvamento das configurações de forma persistente.
#
# =================================================================================

# --- CORES PARA MELHORAR A VISUALIZAÇÃO ---
VERDE='\033[0;32m'
AMARELO='\033[1;33m'
CIANO='\033[0;36m'
VERMELHO='\033[0;31m'
SEM_COR='\033[0m'

# --- Nomes padrão para tabela e chains ---
TABELA="filter"
FAMILY="inet"

# --- FUNÇÃO PARA EXIBIR O CABEÇALHO DINÂMICO ---
mostrar_cabecalho() {
    local NFT_STATUS=$(systemctl is-active nftables)
    
    local STATUS_DISPLAY=""
    if [ "$NFT_STATUS" = "active" ]; then
        STATUS_DISPLAY="${VERDE}Ativo${SEM_COR}"
    else
        STATUS_DISPLAY="${VERMELHO}Inativo${SEM_COR}"
    fi

    clear
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "${CIANO}      Gerenciador de Regras do Firewall nftables     ${SEM_COR}"
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "  ${AMARELO}Status Atual:${SEM_COR} $STATUS_DISPLAY  |  ${AMARELO}Regras Ativas:${SEM_COR} ${RULE_COUNT:-0}"
    echo
}

# --- FUNÇÃO PARA CRIAR A ESTRUTURA BÁSICA (tabela e chains) SE NÃO EXISTIR ---
verificar_estrutura_base() {
    if ! sudo nft list tables | grep -q "table $FAMILY $TABELA"; then
        echo -e "${AMARELO}Estrutura base do nftables não encontrada. Criando...${SEM_COR}"
        sudo nft add table "$FAMILY" "$TABELA"
        
        # Sintaxe da chain corrigida para usar 'priority filter'
        sudo nft add chain "$FAMILY" "$TABELA" input { type filter hook input priority filter \; policy accept \; }
        sudo nft add chain "$FAMILY" "$TABELA" forward { type filter hook forward priority filter \; policy accept \; }
        sudo nft add chain "$FAMILY" "$TABELA" output { type filter hook output priority filter \; policy accept \; }
        
        sudo nft add rule "$FAMILY" "$TABELA" input iif lo accept
        sudo nft add rule "$FAMILY" "$TABELA" input ct state established,related accept
        echo -e "${VERDE}Estrutura base criada.${SEM_COR}"

        # Melhoria de segurança: Perguntar sobre regra SSH
        printf "${AMARELO}É altamente recomendado adicionar uma regra para permitir acesso SSH (porta 22). Deseja adicioná-la agora? [s/N]: ${SEM_COR}"
        read confirm_ssh
        case "$confirm_ssh" in
            s|S)
                sudo nft add rule "$FAMILY" "$TABELA" input tcp dport 22 accept
                echo -e "${VERDE}Regra padrão para SSH adicionada.${SEM_COR}"
                ;;
            *)
                echo -e "${AMARELO}Regra SSH não adicionada. Cuidado ao ativar o firewall!${SEM_COR}"
                ;;
        esac
        sleep 3
    fi
}

# --- FUNÇÃO PARA LIBERAR ACESSO SSH ---
liberar_ssh() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Liberação de Acesso SSH ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso (ex: 192.168.1.10): " ip_address
    read -p "Digite a porta do SSH (ou pressione Enter para usar a padrão '22'): " ssh_port
    ssh_port=${ssh_port:-22}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$ssh_port/tcp${SEM_COR}..."
    sudo nft add rule "$FAMILY" "$TABELA" input ip saddr "$ip_address" tcp dport "$ssh_port" accept
    echo -e "${VERDE}Regra para SSH adicionada com sucesso!${SEM_COR}"
    echo -e "${AMARELO}Lembre-se de salvar as regras para torná-las permanentes (Opção 4).${SEM_COR}"
}

# --- FUNÇÃO PARA LIBERAR OUTROS SERVIÇOS ---
liberar_outro_servico() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Liberação de Outro Serviço/Porta ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso (ex: 0.0.0.0/0 para todos): " ip_address
    read -p "Digite a porta do serviço (ex: 80, 443, 3306): " service_port
    read -p "Digite o protocolo (tcp ou udp, Enter para 'tcp'): " protocol
    protocol=${protocol:-tcp}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$service_port/$protocol${SEM_COR}..."
    sudo nft add rule "$FAMILY" "$TABELA" input ip saddr "$ip_address" "$protocol" dport "$service_port" accept
    echo -e "${VERDE}Regra para o serviço adicionada com sucesso!${SEM_COR}"
    echo -e "${AMARELO}Lembre-se de salvar as regras para torná-las permanentes (Opção 4).${SEM_COR}"
}

# --- FUNÇÃO PARA DELETAR UMA REGRA ---
deletar_regra() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Deletar Regra do Firewall ---${SEM_COR}"
    
    # Comando corrigido: -a antes do comando
    sudo nft -a list ruleset
    echo
    
    read -p "Digite o 'handle' da regra que você deseja deletar (ou '0' para cancelar): " rule_handle

    if ! [[ "$rule_handle" =~ ^[0-9]+$ ]]; then
        echo -e "\n${VERMELHO}Entrada inválida. Por favor, digite um número.${SEM_COR}"
        return
    fi
    
    if [[ "$rule_handle" -eq 0 ]]; then
        echo -e "\n${AMARELO}Operação cancelada.${SEM_COR}"
        return
    fi
    
    printf "Você tem certeza que deseja deletar a regra com handle ${VERMELHO}%s${SEM_COR}? [s/N]: " "$rule_handle"
    read confirm
    
    case "$confirm" in
        s|S)
            echo -e "\nDeletando regra com handle ${VERMELHO}$rule_handle${SEM_COR}..."
            sudo nft delete rule "$FAMILY" "$TABELA" input handle "$rule_handle"
            echo -e "${VERDE}Regra deletada com sucesso!${SEM_COR}"
            echo -e "${AMARELO}Lembre-se de salvar as regras para manter a deleção permanente (Opção 4).${SEM_COR}"
            ;;
        *)
            echo -e "\n${AMARELO}Deleção cancelada pelo usuário.${SEM_COR}"
            ;;
    esac
}

# --- FUNÇÃO PARA SALVAR AS REGRAS ---
salvar_regras() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Salvando Regras Permanentemente ---${SEM_COR}"
    echo "Isso irá sobrescrever o arquivo /etc/nftables.conf com as regras atuais."
    printf "Você tem certeza que deseja continuar? [s/N]: "
    read confirm
    
    case "$confirm" in
        s|S)
            echo -e "\nSalvando regras em /etc/nftables.conf..."
            sudo nft list ruleset > /etc/nftables.conf
            echo -e "${VERDE}Regras salvas com sucesso!${SEM_COR}"
            ;;
        *)
            echo -e "\n${AMARELO}Operação cancelada.${SEM_COR}"
            ;;
    esac
}

# --- FUNÇÕES DE CONTROLE DO SERVIÇO ---
ativar_nftables() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Ativando e Habilitando o Serviço nftables ---${SEM_COR}"
    sudo systemctl enable --now nftables
    echo -e "${VERDE}Serviço nftables iniciado e habilitado para iniciar com o sistema.${SEM_COR}"
}

desativar_nftables() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Desativando o Serviço nftables ---${SEM_COR}"
    sudo systemctl stop nftables
    echo -e "${VERDE}Serviço nftables parado.${SEM_COR}"
}

# --- SCRIPT PRINCIPAL ---

if [[ $EUID -ne 0 ]]; then
   echo -e "${VERMELHO}ERRO: Este script precisa ser executado com privilégios de root (use 'sudo').${SEM_COR}"
   exit 1
fi

verificar_estrutura_base

while true; do
    mostrar_cabecalho
    echo "Escolha uma opção:"
    echo -e "  ${CIANO}--- Gerenciamento de Regras ---${SEM_COR}"
    echo -e "  ${AMARELO}1)${SEM_COR} Liberar acesso SSH para um IP"
    echo -e "  ${AMARELO}2)${SEM_COR} Liberar acesso a outro Serviço/Porta para um IP"
    echo -e "  ${AMARELO}3)${SEM_COR} ${VERMELHO}Deletar uma regra por handle${SEM_COR}"
    echo -e "  ${AMARELO}4)${SEM_COR} ${VERDE}Salvar Regras Atuais (Tornar Permanente)${SEM_COR}"
    echo
    echo -e "  ${CIANO}--- Controle do Serviço ---${SEM_COR}"
    echo -e "  ${AMARELO}5)${SEM_COR} ${VERDE}Ativar${SEM_COR} Serviço nftables"
    echo -e "  ${AMARELO}6)${SEM_COR} ${VERMELHO}Desativar${SEM_COR} Serviço nftables"
    echo
    echo -e "  ${AMARELO}7)${SEM_COR} Sair"
    echo
    read -p "Opção: " choice

    case $choice in
        1) liberar_ssh; sleep 3 ;;
        2) liberar_outro_servico; sleep 3 ;;
        3) deletar_regra; sleep 3 ;;
        4) salvar_regras; sleep 3 ;;
        5) ativar_nftables; sleep 3 ;;
        6) desativar_nftables; sleep 3 ;;
        7) echo -e "\n${CIANO}Saindo...${SEM_COR}"; exit 0 ;;
        *) echo -e "\n${VERMELHO}Opção inválida! Por favor, tente novamente.${SEM_COR}"; sleep 2 ;;
    esac
done
