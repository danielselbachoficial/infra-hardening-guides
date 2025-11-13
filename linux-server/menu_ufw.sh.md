#!/bin/bash

# =================================================================================
# Script com Menu Interativo e Cabeçalho Dinâmico para Gerenciar o Firewall UFW
# =================================================================================
#
# DESCRIÇÃO:
# Este script fornece uma interface de menu para gerenciar o UFW. O cabeçalho
# exibe dinamicamente o status do firewall e o número de regras ativas.
#
# =================================================================================

# --- CORES PARA MELHORAR A VISUALIZAÇÃO ---
VERDE='\033[0;32m'
AMARELO='\033[1;33m'
CIANO='\033[0;36m'
VERMELHO='\033[0;31m'
SEM_COR='\033[0m'

# --- FUNÇÃO PARA EXIBIR O CABEÇALHO DINÂMICO ---
mostrar_cabecalho() {
    local RULE_COUNT=$(sudo ufw status numbered | grep -c "^\[")

    local STATUS_DISPLAY=""
    if [ "$UFW_STATUS" = "active" ]; then
        STATUS_DISPLAY="${VERDE}Ativo${SEM_COR}"
    else
        STATUS_DISPLAY="${VERMELHO}Inativo${SEM_COR}"
    fi

    clear
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "${CIANO}   Gerenciador de Regras do Firewall UFW          ${SEM_COR}"
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "  ${AMARELO}Status Atual:${SEM_COR} $STATUS_DISPLAY  |  ${AMARELO}Regras Ativas:${SEM_COR} $RULE_COUNT"
    echo
}

# --- FUNÇÃO PARA LIBERAR ACESSO SSH ---
liberar_ssh() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Liberação de Acesso SSH ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso: " ip_address
    read -p "Digite a porta do SSH (ou pressione Enter para usar a padrão '22'): " ssh_port
    ssh_port=${ssh_port:-22}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$ssh_port/tcp${SEM_COR}..."
    sudo ufw allow from "$ip_address" to any port "$ssh_port" proto tcp
    echo -e "${VERDE}Regra para SSH adicionada com sucesso!${SEM_COR}"
}

# --- FUNÇÃO PARA LIBERAR OUTROS SERVIÇOS ---
liberar_outro_servico() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Liberação de Outro Serviço/Porta ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso: " ip_address
    read -p "Digite a porta do serviço (ex: 80, 443, 3306): " service_port
    read -p "Digite o protocolo (tcp ou udp, Enter para 'tcp'): " protocol
    protocol=${protocol:-tcp}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$service_port/$protocol${SEM_COR}..."
    sudo ufw allow from "$ip_address" to any port "$service_port" proto "$protocol"
    echo -e "${VERDE}Regra para o serviço adicionada com sucesso!${SEM_COR}"
}

# --- FUNÇÃO PARA DELETAR UMA REGRA ---
deletar_regra() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Deletar Regra do Firewall ---${SEM_COR}"
    
    sudo ufw status numbered
    echo
    
    read -p "Digite o número da regra que você deseja deletar (ou '0' para cancelar): " rule_number

    if ! [[ "$rule_number" =~ ^[0-9]+$ ]]; then
        echo -e "\n${VERMELHO}Entrada inválida. Por favor, digite um número.${SEM_COR}"
        return
    fi
    
    if [[ "$rule_number" -eq 0 ]]; then
        echo -e "\n${AMARELO}Operação cancelada.${SEM_COR}"
        return
    fi
    
    # CORREÇÃO APLICADA AQUI:
    # Usamos 'printf' para exibir o prompt colorido e depois 'read' para capturar a entrada.
    printf "Você tem certeza que deseja deletar a regra número ${VERMELHO}%s${SEM_COR}? [s/N]: " "$rule_number"
    read confirm
    
    case "$confirm" in
        s|S)
            echo -e "\nDeletando regra ${VERMELHO}$rule_number${SEM_COR}..."
            sudo ufw --force delete "$rule_number"
            echo -e "${VERDE}Regra deletada com sucesso!${SEM_COR}"
            ;;
        *)
            echo -e "\n${AMARELO}Deleção cancelada pelo usuário.${SEM_COR}"
            ;;
    esac
}

# --- FUNÇÃO PARA ATIVAR O UFW ---
ativar_ufw() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Ativando o Firewall UFW ---${SEM_COR}"
    sudo ufw enable
    echo -e "${VERDE}Comando para ativar o UFW executado.${SEM_COR}"
}

# --- FUNÇÃO PARA DESATIVAR O UFW ---
desativar_ufw() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Desativando o Firewall UFW ---${SEM_COR}"
    sudo ufw disable
    echo -e "${VERDE}Firewall UFW desativado com sucesso.${SEM_COR}"
}

# --- SCRIPT PRINCIPAL ---

if [[ $EUID -ne 0 ]]; then
   echo -e "${VERMELHO}ERRO: Este script precisa ser executado com privilégios de root (use 'sudo').${SEM_COR}"
   exit 1
fi

while true; do
    mostrar_cabecalho
    echo "Escolha uma opção:"
    echo -e "  ${CIANO}--- Gerenciamento de Regras ---${SEM_COR}"
    echo -e "  ${AMARELO}1)${SEM_COR} Liberar acesso SSH para um IP"
    echo -e "  ${AMARELO}2)${SEM_COR} Liberar acesso a outro Serviço/Porta para um IP"
    echo -e "  ${AMARELO}3)${SEM_COR} ${VERMELHO}Deletar uma regra por número${SEM_COR}"
    echo
    echo -e "  ${CIANO}--- Controle do Firewall ---${SEM_COR}"
    echo -e "  ${AMARELO}4)${SEM_COR} ${VERDE}Ativar${SEM_COR} Firewall UFW"
    echo -e "  ${AMARELO}5)${SEM_COR} ${VERMELHO}Desativar${SEM_COR} Firewall UFW"
    echo
    echo -e "  ${AMARELO}6)${SEM_COR} Sair"
    echo
    read -p "Opção: " choice

    case $choice in
        1)
            liberar_ssh; sleep 2 ;;
        2)
            liberar_outro_servico; sleep 2 ;;
        3)
            deletar_regra; sleep 3 ;;
        4)
            ativar_ufw; sleep 3 ;;
        5)
            desativar_ufw; sleep 3 ;;
        6)
            echo -e "\n${CIANO}Saindo...${SEM_COR}"; exit 0 ;;
        *)
            echo -e "\n${VERMELHO}Opção inválida! Por favor, tente novamente.${SEM_COR}"; sleep 2 ;;
    esac
done
