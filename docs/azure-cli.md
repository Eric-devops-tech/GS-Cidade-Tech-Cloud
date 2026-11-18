# Azure CLI — provisionamento da rede e da VM

Este roteiro cria os recursos de infraestrutura para a demonstração. **Ele não
foi executado** e não cria a API ou instala o Compose. Criar uma VM pode gerar
custos; confirme assinatura, região, tamanho e IP permitido antes de executar.

Os comandos abaixo são PowerShell. Troque o IP de exemplo pelo IP público atual
do avaliador. O acesso externo à API será HTTP/TCP 8080, restrito a esse IP.
Não há regra SSH pública.

## 1. Login e variáveis

```powershell
az login
az account show --output table

$resourceGroup = "rg-cidade-tech"
$location = "chilecentral"
$vnetName = "vnet-cidade-tech"
$subnetName = "snet-app"
$nsgName = "nsg-cidade-tech"
$publicIpName = "pip-cidade-tech"
$nicName = "nic-cidade-tech"
$vmName = "vm-cidade-tech"
$allowedIp = "203.0.113.10/32" # Substitua pelo IP público autorizado
```

## 2. Resource Group, VNet e subnet

```powershell
az group create `
  --name $resourceGroup `
  --location $location `
  --tags project=CidadeTech purpose=academic-demo

az network vnet create `
  --resource-group $resourceGroup `
  --name $vnetName `
  --location $location `
  --address-prefixes 10.20.0.0/16 `
  --subnet-name $subnetName `
  --subnet-prefixes 10.20.1.0/24
```

## 3. NSG aplicada à subnet

```powershell
az network nsg create `
  --resource-group $resourceGroup `
  --name $nsgName `
  --location $location

az network nsg rule create `
  --resource-group $resourceGroup `
  --nsg-name $nsgName `
  --name AllowApi8080FromEvaluator `
  --priority 1100 `
  --direction Inbound `
  --access Allow `
  --protocol Tcp `
  --source-address-prefixes $allowedIp `
  --source-port-ranges "*" `
  --destination-address-prefixes "*" `
  --destination-port-ranges 8080

az network vnet subnet update `
  --resource-group $resourceGroup `
  --vnet-name $vnetName `
  --name $subnetName `
  --network-security-group $nsgName
```

Não crie uma regra de entrada para 1521 ou SSH. A regra de negação padrão da
NSG bloqueia as demais conexões de entrada.

## 4. Public IP, NIC e VM

```powershell
az network public-ip create `
  --resource-group $resourceGroup `
  --name $publicIpName `
  --location $location `
  --sku Standard `
  --allocation-method Static

az network nic create `
  --resource-group $resourceGroup `
  --name $nicName `
  --location $location `
  --vnet-name $vnetName `
  --subnet $subnetName `
  --public-ip-address $publicIpName

az vm create `
  --resource-group $resourceGroup `
  --name $vmName `
  --location $location `
  --size Standard_B2s_v2 `
  --image Ubuntu2204 `
  --admin-username azureuser `
  --generate-ssh-keys `
  --nics $nicName `
  --nsg-rule NONE `
  --tags project=CidadeTech purpose=academic-demo
```

O tamanho indicado (2 vCPU/8 GiB no ambiente usado anteriormente pelo grupo)
deve ser confirmado quanto à disponibilidade e ao custo na região antes de
executar. A Oracle Free e a API compartilham a memória da VM.

## 5. Conferir os recursos e o IP

```powershell
az vm list --resource-group $resourceGroup --show-details --output table
az network vnet show --resource-group $resourceGroup --name $vnetName --output table
az network nsg rule list --resource-group $resourceGroup --nsg-name $nsgName --output table
az network public-ip show `
  --resource-group $resourceGroup `
  --name $publicIpName `
  --query ipAddress `
  --output tsv
```

Depois que a API estiver implantada, valide o CRUD pelo endereço
`http://<IP-PUBLICO>:8080` e mostre `docker compose ps`, logs dos containers e
uma consulta SQL executada diretamente no Oracle.

## 6. Administração e limpeza

Use Azure Run Command para administrar a VM sem abrir SSH público:

```powershell
az vm run-command invoke `
  --resource-group $resourceGroup `
  --name $vmName `
  --command-id RunShellScript `
  --scripts "cd /opt/cidade-tech && docker compose ps"
```

Antes de remover os recursos, confira o nome do grupo e faça backup de qualquer
dado que queira conservar. A remoção do grupo exclui a VM, o disco, o IP, a
VNet, a subnet e a NSG do projeto:

```powershell
az group delete --name $resourceGroup --yes --no-wait
```
