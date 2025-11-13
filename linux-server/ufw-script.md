#!/bin/bash

# ==============================================================================
# Script para Liberar IPs Específicos no Firewall UFW
# ==============================================================================
#
# COMO USAR:
# 1. Adicione os IPs que você deseja liberar dentro dos parênteses da lista
#    IPS_PARA_LIBERAR.
# 2. Salve este arquivo (ex: liberar_ips.sh).
# 3. Dê permissão de execução ao script: chmod +x liberar_ips.sh
# 4. Execute o script com privilégios de administrador: sudo ./liberar_ips.sh
#
# ==============================================================================

# --- LISTA DE IPs ---
# Adicione todos os endereços IP que você deseja liberar aqui,
# separados por um espaço.
IPS_PARA_LIBERAR=("192.168.1.100" "203.0.113.55" "172.16.31.8")

echo "========================================="
echo "Iniciando a configuração do UFW..."
echo "========================================="
echo ""

# Verifica se o UFW está ativo. Se não, pergunta se o usuário deseja ativá-lo.
UFW_STATUS=$(sudo ufw status | grep -o "inactive")
if [ "$UFW_STATUS" = "inactive" ]; then
    read -p "O UFW está inativo. Deseja ativá-lo agora? (s/n): " choice
        sudo ufw enable
        echo "UFW ativado."
    else
        echo "AVISO: O UFW continua inativo. As regras não terão efeito."
    fi
    echo ""
fi

# Loop que percorre cada IP da lista e adiciona a regra no UFW
for ip in "${IPS_PARA_LIBERAR[@]}"; do
    echo "-> Liberando tráfego TOTAL para o IP: $ip"
    sudo ufw allow from "$ip"
done

echo ""
echo "========================================="
echo "Regras adicionadas com sucesso!"
echo "Status atual do UFW:"
echo "========================================="

# Exibe as regras do firewall de forma numerada para fácil visualização
sudo ufw status numbered
