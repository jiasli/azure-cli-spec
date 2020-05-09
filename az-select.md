# az select

The idea of `az select` is to select the prefix of an **Azure resource ID**, so that parameters like `--resource-group`, `--account-name`, `--vnet-name` can be automatically populated.

For instance, with prefix `/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1`,

- `--resource-group` will be populated as `rg1` 
- `--vnet-name` will be populated as `vnet1`

It can be considered as a manual alternative to `az local-context`. Difference:

- `az local-context` focuses on individual **argument context**, like `--resource-group`, `--vnet-name`
- `az select` focuses on **resource ID context** as a whole, like `/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1` and leaves **the parsing from resource ID to arguments** to the commands
  - It has good compatibility with **resource ID argument** (as shown below) and [JSON output passing](pass-json.md) which also focuses on resource ID

## Select a resource group

```sh
# Select by resource group name
> az group select --name rg1
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1

# Select by resource ID
> az select --id /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1
```

## Create a resource

Creating a resource doesn't by default switch the prefix, it is still `/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1`.

`--resource-group` will be automatically populated from the selected prefix as `rg1`.

```sh
> az vnet create --name vnet1
{
    "id": "/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1",
    "name": "vnet1",
    ...
}
WARNING: To create resource within this vnet, you may want to run `az vnet select --name vnet1`

# Without prefix
> az network vnet create --resource-group rg1 
                         --name vnet1

# Without prefix, use resource group id (not supported yet, only a proposal)
> az network vnet create --resource-group /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1 
                         --name vnet1

# Without prefix, use resource group JSON (not supported yet, only a proposal)
> az network vnet create --resource-group $rg
                         --name vnet1
```

## Select a resource

```sh
# Select by name, depending on the current prefix
> az vnet select --name vnet1
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1

# Select by absolute path
> az vnet select --resource-group rg1
                 --name vnet1
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1

# Select by resource ID
> az select --id /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1
```

## Create a sub-resource

Creating a sub-resource doesn't by default switch the prefix either. It is still `/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1`.

`--resource-group` will be automatically populated from the selected prefix as `rg1`. `--vnet-name` will be automatically populated from the selected prefix as `vnet1`.

```sh
> az network vnet subnet create --name subnet1
{
    "id": "/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1/subnets/subnet1",
    "name": "subnet1"
}

# Without prefix
> az network vnet subnet create --resource-group rg1 
                                --vnet-name vnet1 
                                --name subnet1

# Without prefix, use vnet id (not supported yet, only a proposal)
> az network vnet subnet create --vnet /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1
                                --name subnet1

# Without prefix, use vnet JSON (not supported yet, only a proposal)
> az network vnet subnet create --vnet $vnet
                                --name subnet1
```

## Show the selected prefix

```sh
> az select show
/subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1/providers/Microsoft.Network/virtualNetworks/vnet1
```

## Go up

```sh
> az select up
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590/resourceGroups/rg1

> az select up
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590
```

## Clear

Go back to `/subscriptions` (root). This may switch to `/tenant` in the future.
```sh
> az select clear
WARNING: Switched to /subscriptions/0b1f6471-1bf0-4dda-aec3-cb9272f09590
```
