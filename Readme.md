# NPD Find & Explore Azure Resource Manager templates

## To do

The following have not yet been added to these templates, so should be configured as required for your service:

- [ ] Monitoring
- [ ] Aggregated logging
- [ ] Backups for PostgreSQL
- [ ] Azure Blob Store for ActiveStorage
- [ ] Azure Cache for Redis for background jobs/caching
- [ ] Bonus points: Azure Search integration

## Prerequisites

TODO

## Resources these templates create

TODO

## How to run the templates

The ARM templates are idempotent, so are safe to be run repeatedly. Re-running a template when the resources already exist will result in a no-op.

> Note: these templates do not destroy existing resources that don't match the definitions provided, so (for example) renaming a service in the template and re-running it would simply add a second copy of the resource with the new name.

First, you'll need to login:

```bash
az login
```

Then you can run each template in order. For each command you will want to pass:

1. The resource group
2. The ARM template file
3. The common parameters file (optional)
4. The parameters required by the template

The following script runs through each ARM template, using environment variables to save typing common strings repeatedly:

> Important: make sure to change the two PostgreSQL passwords and Rails master key at the top of this script before running.

```bash
POSTGRES_ADMIN_PASSWORD=CHANGE_THIS_TO_A_VERY_SECURE_PASSWORD_^100%
POSTGRES_ADMIN_PASSWORD_STAGING=CHANGE_THIS_TO_A_DIFFERENT_VERY_SECURE_PASSWORD_^100%
RAILS_MASTER_KEY=TODO_COPY_FROM_RAILS 

# Create the resource group to contain everything
az deployment create \
  --location westeurope \
  --template-file 0_resource_groups.json \
  --parameters @_common.parameters.json

RESOURCE_GROUP=s112p01-find-npd-data-persistent-resources

# Create the container registry
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 1_container_registry.json \
  --parameters @_common.parameters.json

# Retrieve Azure Container Registry credentials:
AZ_CR_CRED_JSON=`az acr credential show --name s112p01findnpddata`
AZ_CR_USERNAME=`echo $AZ_CR_CRED_JSON | jq -r '.["username"]'`
AZ_CR_PASSWORD=`echo $AZ_CR_CRED_JSON | jq -r '.["passwords"][0]["value"]'`

# Create the PostgreSQL database, Web App instance
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 5_web_app_service.json \
  --parameters @_common.parameters.json \
  --parameters administratorLogin=npd_admin \
  --parameters administratorLoginPassword=$POSTGRES_ADMIN_PASSWORD \
  --parameters stagingAdministratorLoginPassword=$POSTGRES_ADMIN_PASSWORD_STAGING \
  --parameters dockerRegistryUsername=$AZ_CR_USERNAME \
  --parameters dockerRegistryPassword=$AZ_CR_PASSWORD \
  --parameters railsMasterKey=$RAILS_MASTER_KEY \
  --parameters customHostname=find-npd-data.education.gov.uk

# Retrieve WebApp outbound IP addresses 
AZ_WEBAPP_IPS=`az webapp show --resource-group $RESOURCE_GROUP --name s112p01-find-npd-data | jq -r '.["possibleOutboundIpAddresses"]'`

# Lock down the PostgreSQL firewall
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 6_postgres_firewall.json \
  --parameters @_common.parameters.json \
  --parameters webAppIPAddresses=$AZ_WEBAPP_IPS
```

Don't be worried when your app fails to start. This is expected. You've created a container registry and pointed the Web App to use it, however there aren't any images in there yet. At this point, you will want to configure your CI service to deploy the production container it builds to the container registry, and trigger a deploymnt to the Web App instance.

## Important steps after running the templates

1. Enable diagnostic logging on both Web App slots. The templates aren't honoured for application logs, so these need to be set to retain 100Mb (the maximum) and 365 days.

## Connecting to the production system

> TODO This hung for me in testing... it needs an extra process in the Docker container :/

```bash
az webapp ssh --resource-group s112p01-find-npd-data-persistent-resources --name s112p01-find-npd-data
```

## Logs

The logs are available through the Azure Portal, or using the following CLI command:

```bash
az webapp log tail --resource-group s112p01-find-npd-data-persistent-resources --name s112p01-find-npd-data
```

## Contributing

Bug reports and pull requests are welcome on GitHub at
https://github.com/DfE-Digital/npd-find-and-explore-infra. This project is
intended to be a safe, welcoming space for collaboration, and contributors are
expected to adhere to the [Contributor Covenant](CODE_OF_CONDUCT.md)
code of conduct.

### Standards & principles

There are a handful of government-specific principles and standards we strive to adhere to:

1. [The DfE Coding Principles](https://dfe-digital.github.io/technology-guidance/principles/coding-principles/#coding-principles)
2. [The GDS Service Standard](https://www.gov.uk/service-manual/service-standard), points 8, 9, and 10 are particularly pertinent. It's also worth referring to the [GDS Service Manual](https://www.gov.uk/service-manual/technology).

## License

This project is made available under the terms of the [MIT License](LICENCE.md).
