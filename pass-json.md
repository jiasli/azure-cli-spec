# Pass JSON output between commands

## Goal

The goal is to facilitate the interaction between commands, like how PowerShell [uses variables to store objects](https://docs.microsoft.com/en-us/powershell/scripting/learn/using-variables-to-store-objects).

For example, PowerShell CmdLet [Add-AzVirtualNetworkSubnetConfig](https://docs.microsoft.com/en-us/powershell/module/Az.Network/Add-AzVirtualNetworkSubnetConfig) takes `-VirtualNetwork <PSVirtualNetwork>`. See [Create a virtual network using PowerShell](https://docs.microsoft.com/en-us/azure/virtual-network/quick-create-powershell#create-a-resource-group-and-a-virtual-network). 

```powershell
$virtualNetwork = New-AzVirtualNetwork -ResourceGroupName rg1 -Name vnet1 -Location WestUS -AddressPrefix 10.0.0.0/16
$subnetConfig = Add-AzVirtualNetworkSubnetConfig -Name default -AddressPrefix 10.0.0.0/24 -VirtualNetwork $virtualNetwork
$virtualNetwork | Set-AzVirtualNetwork
```

## Current status

Currently, with Azure CLI users need to append `--query id --output tsv` to a command to extract the information from the output JSON and pass it to the next command, as documented in [Tips for using Azure CLI effectively](https://github.com/Azure/azure-cli/blob/dev/doc/use_cli_effectively.md#passing-values-from-one-command-to-the-other).

For example, we want to add a newly created subnet to a storage account's network rule. See [Configure Azure Storage firewalls and virtual networks](https://docs.microsoft.com/en-us/azure/storage/common/storage-network-security).

With CLI in Bash:

```bash
# Prepare resources
az group create -g rg1 -l westus
az network vnet create -g rg1 -n vnet1
az network vnet subnet create -g rg1 --vnet-name vnet1 -n subnet1 --address-prefixes 10.0.0.0/24 --service-endpoints  Microsoft.Storage
az storage account create -g rg1 -n st0507

# Add the created subnet to storage account's network rule
subnet=$(az network vnet subnet show -g rg1 --vnet-name vnet1 -n subnet1 --query id --output tsv)
az storage account network-rule add -g rg1 --account-name st0507 --subnet $subnet
```

With Azure PowerShell:

```powershell
$subnet = Get-AzVirtualNetwork -ResourceGroupName "myresourcegroup" -Name "myvnet" | Get-AzVirtualNetworkSubnetConfig -Name "mysubnet"
Add-AzStorageAccountNetworkRule -ResourceGroupName "myresourcegroup" -Name "mystorageaccount" -VirtualNetworkResourceId $subnet.Id
```

We want to eliminate the usage of `--query id --output tsv` and achieve the same effect.

## Changes

In this PR, a new factory `get_json_query_type(query='id')` is added for command arguments. It has one parameter as the JMESPath query string, which is used to query the input JSON string. If the input is not a valid JSON, the original value is used.

```py
c.argument('subnet', type=get_json_query_type(), help='Name or ID of subnet. If name is supplied, `--vnet-name` must be supplied.')
```

Then we can pipe the output JSON to the next command with either **variable style** or **pipeline style**.

### Variable style

#### Bash

https://www.gnu.org/software/bash/manual/html_node/Shell-Parameters.html

```bash
subnet=$(az network vnet subnet show -g rg1 --vnet-name vnet1 -n subnet1)
az storage account network-rule add -g rg1 --account-name st0507 --subnet "$subnet"
```

#### PowerShell

https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_variables

Variable style doesn't work well with PowerShell due to a bug in PowerShell. See appendix for more information.

### Pipeline style

#### Bash

https://www.gnu.org/software/bash/manual/html_node/Pipelines.html

```bash
az network vnet subnet show -g rg1 --vnet-name vnet1 -n subnet1 | az storage account network-rule add -g rg1 --account-name st0507 --subnet @-
```

#### PowerShell

https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines

As symbol `@` is interpreted by PowerShell as [splatting symbol](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_splatting), so quote it or escape it.

```powershell
az network vnet subnet show -g rg1 --vnet-name vnet1 -n subnet1 | az storage account network-rule add -g rg1 --account-name st0507 --subnet '@-'
az network vnet subnet show -g rg1 --vnet-name vnet1 -n subnet1 | az storage account network-rule add -g rg1 --account-name st0507 --subnet `@-
```

More information about quoting can be found at https://github.com/Azure/azure-cli/blob/dev/doc/use_cli_effectively.md#quoting-issues

## Problems

Some problems can emerge with this approach:

- The JSON can't be used to populate multiple parameters. For example, populating `--resource-group` and `--vnet-name` when a vnet JSON is provided:
    ```bash
    az network vnet subnet create --resource-group rg1 --vnet-name $vnet -n subnet1
    ```
    
    Possible solutions:

    1. Support `--vnet` which takes a resource ID by itself, like
        ```bash
        az network vnet subnet create --vnet /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1 -n subnet1
        ```
        This solution is perhaps the best.
    2. Pass the JSON twice to the following command, like
        ```bash
        az network vnet subnet create --resource-group $vnet --vnet-name $vnet -n subnet1
        ```
        This solution is verbose and counterintuitive. 
- This can conflict with the newly introduced local context which may automatically populate `--resource-group` and `--vnet-name`

## Appendix

### Quoting issue with PowerShell 

Due to issue https://github.com/PowerShell/PowerShell/issues/1995, double quotes within the JSON string are lost when calling a native `.exe` file. 

```powershell
# Note that the double quotes are lost
> python.exe -c "import sys; print(sys.argv)" '{"key": "value"}'
['-c', '{key: value}']

# Escape double quotes (") with backward-slashes (\), and quote the string with single quotes (')
> python.exe -c "import sys; print(sys.argv)" '{\"key": \"value\"}'
['-c', '{"key: "value"}']

# First escape double quotes with backticks (`), then escape double quotes with backward-slash (\)
> python.exe -c "import sys; print(sys.argv)" "{\`"key\`": \`"value\`"}"
['-c', '{"key": "value"}']
```

As you can see, the workaround makes the command awkward. Of course we can do some replacement to the JSON string to workaround the PowerShell issue but it takes more efforts, thus counteracting the benefit we gain from variable style.
