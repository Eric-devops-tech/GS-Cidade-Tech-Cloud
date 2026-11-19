# Azure CLI — provisionamento da rede e da VM

O ambiente foi provisionado e demonstrado em vídeo. Depois da gravação, o
Resource Group `rg-cidade-tech` foi excluído; portanto, **não há VM ou API
Azure ativa agora**. Para reproduzir a implantação, use os comandos em
[`azure-cli-comandos.txt`](../azure-cli-comandos.txt). Eles criam uma VM Ubuntu
22.04 `Standard_B2s_v2` em `chilecentral`, com Docker Compose para a API e o
PostgreSQL.

Para consultar e copiar a sequência completa de comandos, incluindo a instalação
do Docker pela Azure Run Command e as verificações da VM, use
[`azure-cli-comandos.txt`](../azure-cli-comandos.txt). Este documento explica as
etapas e a arquitetura do provisionamento. A criação requer uma assinatura Azure
ativa com permissões, cota, disponibilidade dos recursos e cobrança habilitada;
os custos serão cobrados na assinatura selecionada (créditos disponíveis podem
ser aplicados). Se o Resource Group já existir, inspecione-o antes de executar
novamente as etapas de criação.

Os comandos abaixo são PowerShell. Informe o IPv4 público atual que deve acessar
a aplicação; o sufixo `/32` restringe a regra a esse endereço. Não registre o IP
em arquivos públicos. O acesso externo à API será HTTP/TCP 8080. Não há regra
SSH pública.

## 1. Login e variáveis

```powershell
az login
az account show --output table
# Selecione a assinatura que deve receber os recursos:
# az account set --subscription "<nome ou ID da assinatura>"

$resourceGroup = "rg-cidade-tech"
$location = "chilecentral"
$vnetName = "vnet-cidade-tech"
$subnetName = "snet-app"
$nsgName = "nsg-cidade-tech"
$publicIpName = "pip-cidade-tech"
$nicName = "nic-cidade-tech"
$vmName = "vm-cidade-tech"
$clientIp = Read-Host "IPv4 público autorizado (somente o endereço, sem /)"
$parsedIp = $null
if (-not [System.Net.IPAddress]::TryParse($clientIp, [ref]$parsedIp) -or
    $parsedIp.AddressFamily -ne [System.Net.Sockets.AddressFamily]::InterNetwork) {
  throw "Informe um endereço IPv4 válido."
}
$allowedIp = "$($parsedIp.IPAddressToString)/32"
```

As execuções de script Linux abaixo são enviadas codificadas em Base64 para
evitar problemas de aspas e quebras de linha do PowerShell com o Azure CLI. O
helper imprime a saída do Azure Run Command e interrompe o roteiro se o comando
falhar:

```powershell
function Invoke-CidadeTechVmScript {
  param([Parameter(Mandatory = $true)][string]$Script)

  $encodedScript = [Convert]::ToBase64String(
    [Text.Encoding]::UTF8.GetBytes($Script)
  )
  $remoteCommand = "echo $encodedScript | base64 -d | bash"
  $result = az vm run-command invoke `
    --resource-group $resourceGroup `
    --name $vmName `
    --command-id RunShellScript `
    --scripts $remoteCommand `
    --query "value[0].message" `
    --output tsv

  if ($LASTEXITCODE -ne 0) {
    throw "Azure Run Command falhou na VM $vmName."
  }
  $result
}
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

Não crie uma regra de entrada para 5432 ou SSH. A regra de negação padrão da
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
executar. O PostgreSQL e a API compartilham a memória da VM.

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

Depois de executar a implantação descrita em `azure-cli-comandos.txt`, obtenha
o IP público atual da VM e valide o CRUD em
`http://<IP-PUBLICO>:8080/swagger-ui/index.html`. A NSG permite acesso somente
do IPv4 informado na variável `$clientIp`. Para outro avaliador ou rede, atualize
a origem da regra para o IPv4 atual antes de testar. Mostre também o estado e os
logs dos containers e consultas SQL executadas diretamente no PostgreSQL.

## 6. Administração e limpeza

Use Azure Run Command para administrar a VM sem abrir SSH público:

```powershell
Invoke-CidadeTechVmScript -Script "cd /opt/cidade-tech && docker compose ps"
```

As credenciais aleatórias do gestor inicial ficam em
`/root/cidade-tech-manager.txt`, legíveis somente por root. Consulte-as antes da
gravação usando Azure Run Command; não compartilhe nem grave a saída:

```powershell
Invoke-CidadeTechVmScript -Script "cat /root/cidade-tech-manager.txt"
```

Quando terminar, os recursos podem permanecer ativos e gerar custos. Antes de
removê-los, confira o nome do grupo e faça backup de qualquer dado que queira
conservar. A remoção exclui a VM, o disco, o IP, a VNet, a subnet e a NSG do
projeto:

```powershell
az group delete --name $resourceGroup --yes --no-wait
```
