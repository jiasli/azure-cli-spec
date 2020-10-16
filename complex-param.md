# Complex parameter support

This article talks about supporting complex parameters.

For example, the REST API we are translating to CLI command is [Time Series Insights Environments - Create Or Update](https://docs.microsoft.com/en-us/rest/api/time-series-insights/management(gen1/gen2)/environments/createorupdate).

We want to construct this JSON:

```json
  "properties": {
    "dataRetentionTime": "P31D",
    "partitionKeyProperties": [
      {
        "name": "DeviceId",
        "type": "String"
      },
      {
        "name": "Section",
        "type": "Int"
      }
    ]
```

This is not a basic type but a `list` of `dict`.

We have following options to translate `partitionKeyProperties` to CLI param `--partition-key-properties`:

## Possible solutions

### 1. Positional Argument Action, extending `argparse._AppendAction`

```sh
--partition-key-properties DeviceId String
--partition-key-properties Section Int
```

This form relies on the position(order) of arguments, so we can't reverse the order and use `String DeviceId` instead. This form is also not friendly to the reader of the command.


### 2. Keyword Argument Action, extending `argparse._AppendAction`

```sh
--partition-key-properties name=DeviceId type=String
--partition-key-properties name=Section type=Int
```

This form relies on the `key=value` structure of arguments. This form is friendly to the reader of the command, but more verbose.

> The above 2 forms are documented at [Collection Management with Actions](https://github.com/Azure/azure-cli/blob/dev/doc/command_guidelines.md#collection-management-with-actions). ⚠ Note that the sample code is wrong. It should use `setattr()` or call `super()`, instead of `return`. Please check [argparse doc](https://docs.python.org/3/library/argparse.html#action) or the [code in monitor module](https://github.com/Azure/azure-cli/blob/dev/src/azure-cli/azure/cli/command_modules/monitor/actions.py) instead.


### 3. Tag-like Action, extending `argparse.Action`

```sh
--partition-key-properties DeviceId=String Section=Int
```

The result of `az group show` has a `tags` property which is a dict by itself:

```json
  "tags": {
    "mykey1": "myvalue2",
    "mykey1": "myvalue2"
  },
```

When creating this JSON, use the form of `--tags key=value key=value`.

Tag-like Action use the same form but constructs a list instead a dict. This action won't be reasonable if more properties are added, for example, `DeviceId=String=foo=bar`.


### 4. Flatten everything, only use whitespace

This is the simplest form without all deliminators, but it highly relies on the order in which arguments are provided.

```sh
--partition-key-properties DeviceId String Section Int
```


### 5. Direct JSON

```sh
--partition-key-properties '[{"name":"DeviceId","type":"String"},{"name":"Section","type":"Int"}]'
```

This is the easiest form to implement and auto-gen, but difficult for the user to write.


### 6. AWS CLI shorthand syntax

```sh
--partition-key-properties name=DeviceId,type=String name=Section,type=Int
```

This is a simple form of JSON and is very easy for auto-gen. The argument can also be replaced with JSON directly. But, it is not compatible with **Keyword Argument Action**. The difference between **Shorthand syntax** and **Keyword Argument Action** is:

|Argument Form          |comma(`,`) separates |space(` `) separates |param name(`--partition-key-properties`) appears
|-                      |-                    | -                   |-
|Shorthand Syntax       |properties           |objects              |only once
|Keyword Argument Action|N/A                  |properties           |multiple times

This form can be extended to represent more complicated structures, like the one in [`aws ec2 run-instances`](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html):

```
Ipv6AddressCount=integer,Ipv6Addresses=[{Ipv6Address=string},{Ipv6Address=string}]
```

It corresponds to

```json
    "Ipv6AddressCount": integer,
    "Ipv6Addresses": [
      {
        "Ipv6Address": "string"
      }
```

#### Links for AWS CLI Shorthand Syntax

AWS CLI uses a term called **Shorthand Syntax** to deal with complex structures. See links below:

* Generating JSON skeleton and use JSON as input: https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-skeleton.html
* Using Shorthand Syntax: https://docs.aws.amazon.com/cli/latest/userguide/cli-usage-shorthand.html
* Example command `aws ec2 run-instances`: https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html
* `ParamShorthandParser` source code: https://github.com/aws/aws-cli/blob/bbde9b33381ee4010b2cc3e7fe27b9ba3871893d/awscli/argprocess.py#L274


### 7. Subcommands

```sh
az ... partition-key-properties add --name DeviceId --type String --defer
az ... partition-key-properties add --name Section --type Int
```

Subcommands are suitable if the object contains many properties, but difficult to auto-gen.


### 8. Construct objects, like Azure PowerShell

```sh
p1=$(az ... partition-key-property create --name DeviceId --type String)
# echo $p1
# {"name":"DeviceId","type":"String"}
p2=$(az ... partition-key-property create --name Section --type Int)
# echo $p2
# {"name":"Section","type":"Int"}
az ... --partition-key-properties "$p1" "$p2"
```

This is actually a variation of subcommands. Both this solution and subcommands can give better help message, because it can provide detailed information for parameters, like `--name` and `--type` in  `az ... partition-key-property create`.

⚠ This will hit the [quoting issues with PowerShell](https://github.com/Azure/azure-cli/blob/dev/doc/quoting-issues-with-powershell.md).
