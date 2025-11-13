#!/bin/bash

# =================================================================================
# Script com Menu Interativo para Gerenciar Regras do Firewall UFW
# =================================================================================
#
# DESCRIÇÃO:
# Este script fornece uma interface de menu para adicionar regras ao UFW
# de forma segura e intuitiva. É ideal para liberar acesso a serviços
# para endereços IP específicos.
#
# COMO USAR:
# 1. Salve este arquivo (ex: menu_ufw.sh).
# 2. Dê permissão de execução: chmod +x menu_ufw.sh
# 3. Execute com privilégios de administrador: sudo ./menu_ufw.sh
#
# =================================================================================

# --- CORES PARA MELHORAR A VISUALIZAÇÃO ---
VERDE='\033[0;32m'
AMARELO='\033[1;33m'
CIANO='\033[0;36m'
VERMELHO='\033[0;31m'
SEM_COR='\033[0m'

# --- FUNÇÃO PARA EXIBIR O CABEÇALHO ---
mostrar_cabecalho() {
    clear
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo -e "${CIANO}   Gerenciador de Regras do Firewall UFW          ${SEM_COR}"
    echo -e "${CIANO}=====================================================${SEM_COR}"
    echo
}

# --- FUNÇÃO PARA LIBERAR ACESSO SSH ---
liberar_ssh() {
    echo -e "${AMARELO}--- Liberação de Acesso SSH ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso: " ip_address
    read -p "Digite a porta do SSH (ou pressione Enter para usar a padrão '22'): " ssh_port

    # Define a porta padrão 22 se o usuário não digitar nada
    ssh_port=${ssh_port:-22}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$ssh_port/tcp${SEM_COR}..."
    sudo ufw allow from "$ip_address" to any port "$ssh_port" proto tcp
    echo -e "${VERDE}Regra para SSH adicionada com sucesso!${SEM_COR}"
}

# --- FUNÇÃO PARA LIBERAR OUTROS SERVIÇOS ---
liberar_outro_servico() {
    echo -e "${AMARELO}--- Liberação de Outro Serviço/Porta ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso: " ip_address
    read -p "Digite a porta do serviço (ex: 80, 443, 3306): " service_port
    read -p "Digite o protocolo (tcp ou udp, Enter para 'tcp'): " protocol

    # Define o protocolo padrão TCP se o usuário não digitar nada
    protocol=${protocol:-tcp}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$service_port/$protocol${SEM_COR}..."
    sudo ufw allow from "$ip_address" to any port "$service_port" proto "$protocol"
    echo -e "${VERDE}Regra para o serviço adicionada com sucesso!${SEM_COR}"
}

# --- FUNÇÃO PARA MOSTRAR STATUS ---
mostrar_status() {
    echo -e "${AMARELO}--- Status Atual do UFW ---${SEM_COR}"
    sudo ufw status numbered
    echo
    read -n 1 -s -r -p "Pressione qualquer tecla para voltar ao menu..."
}

# --- SCRIPT PRINCIPAL ---

# 1. Verifica se o script está sendo executado como root
if [[ $EUID -ne 0 ]]; then
   echo -e "${VERMELHO}ERRO: Este script precisa ser executado com privilégios de root (use 'sudo').${SEM_COR}"
   exit 1
fi

# 2. Verifica se o UFW está ativo e, se não, o ativa
if ! sudo ufw status | grep -q "active"; then
    echo -e "${AMARELO}UFW está inativo. Ativando agora...${SEM_COR}"
    sudo ufw enable
    echo -e "${VERDE}UFW ativado.${SEM_COR}"
    sleep 2
fi

# 3. Loop principal do menu
while true; do
    mostrar_cabecalho
    echo "Escolha uma opção:"
    echo -e "  ${AMARELO}1)${SEM_COR} Liberar acesso SSH para um IP específico"
    echo -e "  ${AMARELO}2)${SEM_COR} Liberar acesso a outro Serviço/Porta para um IP"
    echo -e "  ${AMARELO}3)${SEM_COR} Ver status atual do UFW"
    echo -e "  ${AMARELO}4)${SEM_COR} Sair"
    echo
    read -p "Opção: " choice

    case $choice in
        1)
            mostrar_cabecalho
            liberar_ssh
            sleep 3
            ;;
        2)
            mostrar_cabecalho
            liberar_outro_servico
            sleep 3
            ;;
        3)
            mostrar_cabecalho
            mostrar_status
            ;;
        4)
            echo -e "\n${CIANO}Saindo...${SEM_COR}"
            exit 0
            ;;
        *)
            echo -e "\n${VERMELHO}Opção inválida! Por favor, tente novamente.${SEM_COR}"
            sleep 2
            ;;
    esac
done
