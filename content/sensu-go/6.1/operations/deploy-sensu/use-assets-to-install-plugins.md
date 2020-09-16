---
title: "Use dynamic runtime assets to install plugins"
linkTitle: "Use Assets to Install Plugins"
description: "Dynamic runtime assets are shareable, reusable packages that make it easier to deploy Sensu plugins. You can use assets to provide the plugins, libraries, and runtimes you need to power your monitoring workflows. Read the guide to get started using dynamic runtime assets."
weight: 140
version: "6.1"
product: "Sensu Go"
platformContent: False
menu: 
  sensu-go-6.1:
    parent: deploy-sensu
---

Dynamic runtime assets are shareable, reusable packages that make it easier to deploy Sensu plugins.
You can use assets to provide the plugins, libraries, and runtimes you need to automate your monitoring workflows.
See the [asset reference][1] for more information about dynamic runtime assets.
This guide uses the [Sensu PagerDuty Handler dynamic runtime asset][7] as an example.

## Register the Sensu PagerDuty Handler asset

To add the [Sensu PagerDuty Handler dynamic runtime asset][7] to Sensu, use [`sensuctl asset add`][6]:

{{< code shell >}}
sensuctl asset add sensu/sensu-pagerduty-handler:1.2.0 -r pagerduty-handler
fetching bonsai asset: sensu/sensu-pagerduty-handler:1.2.0
added asset: sensu/sensu-pagerduty-handler:1.2.0

You have successfully added the Sensu asset resource, but the asset will not get downloaded until
it's invoked by another Sensu resource (ex. check). To add this runtime asset to the appropriate
resource, populate the "runtime_assets" field with ["pagerduty-handler"].
{{< /code >}}

This example uses the `-r` (rename) flag to specify a shorter name for the asset: `pagerduty-handler`.

You can also click the Download button on the asset page in [Bonsai][7] to download the asset definition for your Sensu backend platform and architecture.

{{% notice note %}}
**NOTE**: Sensu does not download and install asset builds onto the system until they are needed for command execution.
Read [the asset reference](../assets#dynamic-runtime-asset-builds) for more information about asset builds.
{{% /notice %}}

## Adjust the asset definition

Asset definitions tell Sensu how to download and verify the asset when required by a check, filter, mutator, or handler.

After you add or download the asset definition, open the file and adjust the `namespace` and `filters` for your Sensu instance.
Here's the asset definition for version 1.2.0 of the [Sensu PagerDuty Handler][7] asset for Linux AMD64:

{{< code yml >}}
---
type: Asset
api_version: core/v2
metadata:
  name: pagerduty-handler
  namespace: default
  labels: {}
  annotations: {}
spec:
  url: https://assets.bonsai.sensu.io/02fc48fb7cbfd27f36915489af2725034a046772/sensu-pagerduty-handler_1.2.0_linux_amd64.tar.gz
  sha512: 5be236b5b9ccceb10920d3a171ada4ac4f4caaf87f822475cd48bd7f2fab3235fa298f79ef6f97b0eb6498205740bb1af1120ca036fd3381edfebd9fb15aaa99
  filters:
  - entity.system.os == 'linux'
  - entity.system.arch == 'amd64'
{{< /code >}}

Filters for _check_ dynamic runtime assets should match entity platforms.
Filters for _handler and filter_ dynamic runtime assets should match your Sensu backend platform.
If the provided filters are too restrictive for your platform, replace `os` and `arch` with any supported [entity system attributes][4] (for example, `entity.system.platform_family == 'rhel'`).
You may also want to customize the asset `name` to reflect the supported platform (for example, `pagerduty-handler-linux`) and add custom attributes with [`labels` and `annotations`][5].

**Enterprise-tier dynamic runtime assets** (like the [ServiceNow][10] and [Jira][11] event handlers) require a Sensu commercial license.
For more information about commercial features and to activate your license, see [Get started with commercial features][12].

Use sensuctl to verify that the asset is registered and ready to use:

{{< code shell >}}
sensuctl asset list --format yaml
{{< /code >}}

## Create a workflow

With the asset downloaded and registered, you can use it in a monitoring workflow.
Dynamic runtime assets may provide executable plugins intended for use with a Sensu check, handler, mutator, or hook, or JavaScript libraries intended to provide functionality for use in event filters.
The details in Bonsai are the best resource for information about each asset's capabilities and configuration.

For example, to use the [Sensu PagerDuty Handler][7] asset, you would create a `pagerduty` handler that includes your PagerDuty service API key in place of `SECRET` and `pagerduty-handler` as a runtime asset:

{{< language-toggle >}}

{{< code yml >}}
type: Handler
api_version: core/v2
metadata:
  name: pagerduty
  namespace: default
spec:
  command: sensu-pagerduty-handler
  env_vars:
  - PAGERDUTY_TOKEN=SECRET
  filters:
  - is_incident
  runtime_assets:
  - pagerduty-handler
  timeout: 10
  type: pipe
{{< /code >}}

{{< code json >}}
{
    "api_version": "core/v2",
    "type": "Handler",
    "metadata": {
        "namespace": "default",
        "name": "pagerduty"
    },
    "spec": {
        "type": "pipe",
        "command": "sensu-pagerduty-handler",
        "env_vars": [
          "PAGERDUTY_TOKEN=SECRET"
        ],
        "runtime_assets": ["pagerduty-handler"],
        "timeout": 10,
        "filters": [
            "is_incident"
        ]
    }
}
{{< /code >}}

{{< /language-toggle >}}

Save the definition to a file (for example, `pagerduty-handler.json`), and add it to Sensu with sensuctl:

{{< code shell >}}
sensuctl create --file pagerduty-handler.json
{{< /code >}}

Now that Sensu can create incidents in PagerDuty, you can automate this workflow by adding the `pagerduty` handler to your Sensu service check definitions.
See [Monitor server resources][13] to learn more.

## Next steps

Read these resources for more information about using dynamic runtime assets in Sensu:

- [Assets reference][1]
- [Asset format specification][14]
- [Share assets on Bonsai][15]

You can also try our interactive tutorial to [send critical alerts to your PagerDuty account][8].


[1]: ../assets/
[2]: #create-an-asset
[3]: https://bonsai.sensu.io
[4]: ../../../observability-pipeline/observe-entities/entities/#system-attributes
[5]: ../assets/#metadata-attributes
[6]: ../../../sensuctl/sensuctl-bonsai/#install-dynamic-runtime-asset-definitions
[7]: https://bonsai.sensu.io/assets/sensu/sensu-pagerduty-handler
[8]: ../../../learn/sensu-pagerduty/
[10]: https://bonsai.sensu.io/assets/sensu/sensu-servicenow-handler
[11]: https://bonsai.sensu.io/assets/sensu/sensu-jira-handler
[12]: ../../../commercial/
[13]: ../../../observability-pipeline/observe-schedule/monitor-server-resources/
[14]: ../assets#dynamic-runtime-asset-format-specification
[15]: ../assets#share-an-asset-on-bonsai
[16]: https://bonsai.sensu.io