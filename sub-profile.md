# Sub-profile for command modules

This spec is intended to address https://github.com/Azure/azure-cli/issues/14102

Currently under each profile (like `latest`, `2019-03-01-hybrid`), the API version of each RP is fixed. For example, in the `latest` profile, `ResourceType.MGMT_RESOURCE_RESOURCES` is fixed to `2019-07-01`. **If  `vm` module relies on `ResourceType.MGMT_RESOURCE_RESOURCES`, it must use `2019-07-01` as well**:

https://github.com/Azure/azure-cli/blob/b5937aa49d572fe93e02bb2d30fa6b1cc575d224/src/azure-cli-core/azure/cli/core/profiles/_shared.py#L148


```py
        ResourceType.MGMT_RESOURCE_RESOURCES: '2019-07-01',
```

In order for each commond module to have its own API dependency of other RPs, a sub-profile must be established, such as `src/azure-cli/azure/cli/command_modules/vm/_dependency_profile.py` which contains a `dict`:

```py
DEPENDENCY_API_PROFILES = {
    'latest': {
        ResourceType.MGMT_STORAGE: '2019-06-01',
        ResourceType.MGMT_NETWORK: '2020-04-01', 
        ResourceType.MGMT_RESOURCE_RESOURCES: '2019-07-01'
    },
    '2019-03-01-hybrid': {
        ResourceType.MGMT_STORAGE: '2017-10-01',
        ResourceType.MGMT_NETWORK: '2017-10-01',
        ResourceType.MGMT_RESOURCE_RESOURCES: '2018-05-01',
    },
    ...
}
```

This `dict` indicates:

- If the current main profile is `latest`, `vm` command module will use `2019-07-01` for `ResourceType.MGMT_RESOURCE_RESOURCES`
- If the current main profile is `2019-03-01-hybrid`, `vm` command module will use `2018-05-01` for `ResourceType.MGMT_RESOURCE_RESOURCES`
