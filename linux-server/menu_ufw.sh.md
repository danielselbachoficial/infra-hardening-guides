#!/bin/bash

# =================================================================================
# Script com Menu Interativo para Gerenciar Regras do Firewall UFW
# =================================================================================
#
# DESCRIÇÃO:
# Este script fornece uma interface de menu para gerenciar o UFW, permitindo
# adicionar regras para IPs específicos, ativar, desativar e verificar o status
# do firewall de forma segura e intuitiva.
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

# --- FUNÇÃO PARA VERIFICAR SE O UFW ESTÁ ATIVO ANTES DE ADICIONAR REGRAS ---
checar_status_ufw() {
    if ! sudo ufw status | grep -q "active"; then
        echo -e "${VERMELHO}AVISO: O UFW está inativo.${SEM_COR}"
        echo -e "A regra será adicionada, mas não terá efeito até que o firewall seja ativado (Opção 5)."
        read -n 1 -s -r -p "Pressione qualquer tecla para continuar..."
        echo
    fi
}

# --- FUNÇÃO PARA LIBERAR ACESSO SSH ---
liberar_ssh() {
    mostrar_cabecalho
    checar_status_ufw
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
    checar_status_ufw
    echo -e "${AMARELO}--- Liberação de Outro Serviço/Porta ---${SEM_COR}"
    read -p "Digite o endereço IP que terá acesso: " ip_address
    read -p "Digite a porta do serviço (ex: 80, 443, 3306): " service_port
    read -p "Digite o protocolo (tcp ou udp, Enter para 'tcp'): " protocol

    protocol=${protocol:-tcp}

    echo -e "\nLiberando acesso para o IP ${VERDE}$ip_address${SEM_COR} na porta ${VERDE}$service_port/$protocol${SEM_COR}..."
    sudo ufw allow from "$ip_address" to any port "$service_port" proto "$protocol"
    echo -e "${VERDE}Regra para o serviço adicionada com sucesso!${SEM_COR}"
}

# --- FUNÇÃO PARA MOSTRAR STATUS ---
mostrar_status() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Status Atual do UFW ---${SEM_COR}"
    sudo ufw status numbered
    echo
    read -n 1 -s -r -p "Pressione qualquer tecla para voltar ao menu..."
}

# --- FUNÇÃO PARA ATIVAR O UFW ---
ativar_ufw() {
    mostrar_cabecalho
    echo -e "${AMARELO}--- Ativando o Firewall UFW ---${SEM_COR}"
    # O comando 'enable' pode interromper conexões SSH existentes e pede confirmação.
    # A flag '--force' evita a pergunta, mas é mais seguro deixar o usuário confirmar.
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

# Verifica se o script está sendo executado como root
if [[ $EUID -ne 0 ]]; then
   echo -e "${VERMELHO}ERRO: Este script precisa ser executado com privilégios de root (use 'sudo').${SEM_COR}"
   exit 1
fi

# Loop principal do menu
while true; do
    mostrar_cabecalho
    echo "Escolha uma opção:"
    echo -e "  ${CIANO}--- Adicionar Regras ---${SEM_COR}"
    echo -e "  ${AMARELO}1)${SEM_COR} Liberar acesso SSH para um IP"
    echo -e "  ${AMARELO}2)${SEM_COR} Liberar acesso a outro Serviço/Porta para um IP"
    echo
    echo -e "  ${CIANO}--- Gerenciamento do Firewall ---${SEM_COR}"
    echo -e "  ${AMARELO}3)${SEM_COR} Ver status atual do UFW"
    echo -e "  ${AMARELO}4)${SEM_COR} ${VERDE}Ativar${SEM_COR} Firewall UFW"
    echo -e "  ${AMARELO}5)${SEM_COR} ${VERMELHO}Desativar${SEM_COR} Firewall UFW"
    echo
    echo -e "  ${AMARELO}6)${SEM_COR} Sair"
    echo
    read -p "Opção: " choice

    case $choice in
        1)
            liberar_ssh
            sleep 2
            ;;
        2)
            liberar_outro_servico
            sleep 2
            ;;
        3)
            mostrar_status
            ;;
        4)
            ativar_ufw
            sleep 3
            ;;
        5)
            desativar_ufw
            sleep 3
            ;;
        6)
            echo -e "\n${CIANO}Saindo...${SEM_COR}"
            exit 0
            ;;
        *)
            echo -e "\n${VERMELHO}Opção inválida! Por favor, tente novamente.${SEM_COR}"
            sleep 2
            ;;
    esac
done
