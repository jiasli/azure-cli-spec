# `az logout` with MSAL integration

ℹ **Cache** is short for **credential cache**

Cartesian product: `all_use_cases` = `auth_lib` * `level` * `if_clear_cache` * `if_current`

## auth_lib

Which underlying authentication library Azure CLI uses:

- ADAL (current)
- MSAL (future, SSO with PowerShell, VS, etc)
  
## level

Levels for `az logout`, controlled by `--user`, `--cloud`, `--all`/`az account clear`:

- Clear cache of one user in one cloud for CLI
- Clear cache of all users in one cloud for CLI
- Clear cache of all users in all clouds for CLI
- Delete cache file, including other app's credentials

## if_current

Whether to logout the current user/cloud, controlled by the existence of `--username` and `--cloud`:

- ~~`--username`~~ ~~`--cloud`~~: Current user
- `--username` `--cloud`: Specific user

## if_clear_cache

Whether to clear cache during `az logout`, controlled by `--clear-credential`:

- ~~`--clear-credential`~~: Persist cache
- `--clear-credential`: Clear cache

## auth_lib * level

**`auth_lib`** * **`level`** * `if_clear_cache=True` * `if_current=False`:

| Level | CLI + ADAL | MSAL Support ? | CLI + MSAL (prototype)
| - | - | - | -
| Clear cache of one user in one cloud for CLI| `az logout --username`  | `ClientApplication.remove_account` | `az logout --username --clear-credential`
| Clear cache of all users in one cloud for CLI | ❌ | `loop ClientApplication.get_accounts:` `ClientApplication.remove_account` | `az logout --cloud --clear-credential`?
| Clear cache of all users in all clouds for CLI | `az account clear` | ❌ (1) | `az account clear --clear-credential` / `az logout -all --clear-credential`?
| Delete cache file, including other app's credentials | `az account clear` | ❌ (2) | ❌ (Not implement)

1. In [discussion](https://teams.microsoft.com/l/message/19:85462ed2792e424bb248555e238caa8c@thread.skype/1588037256654?tenantId=72f988bf-86f1-41af-91ab-2d7cd011db47&groupId=3e17dcb0-4257-4a30-b843-77f47f1d4121&parentMessageId=1587982973837&teamName=Azure%20SDK&channelName=Service%20-%20Azure.Identity&createdTime=1588037256654) with MSAL team for cross-cloud support. **Workaround** is to loop through clouds, but has **cloud leak** issue. MSAL team doesn't think this is a problem, though CLI team think there may be security concerns
2. Cache file is not managed by MSAL, but by CLI. MSAL only manipulates the cache fed in

## auth_lib * if_clear_cache

**`auth_lib`** * `level=Any` * **`if_clear_cache`** * `if_current=Any`

| `auth_lib` | ADAL (current) | MSAL (future, SSO) | 
| - | - | - 
| File | `accessToken.json` | `masl.cache`
| Persist cache | N/A | Default with a warning
| Clear cache | Default | `--clear-credential` | 


# Get operations

| Operation | CLI + ADAL | MSAL Support ? | CLI + MSAL (prototype)
| - | - | - | -
| Get accounts in CLI profile | `az account list` | N/A | `az account list`
| Get accounts in cache | `az account list` (1) | `ClientApplication.get_accounts` | `az account credential list`?

1. Currently **CLI profile** and **cred cache** are synced
