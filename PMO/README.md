# Deploying a PMO Host with cloud-init

The [cloud-init script](PMO/pmo-cloud-config.sh) script can be used



## Steps

1. Deploy a Virtual Machine or Bare Metal Instance and apply the [cloud-init script](PMO/pmo-cloud-config.sh) script during boot time. (Important: Ensure Config Drive is Enabled)
2. Update the following options in the script: `DU_FQDN`, `OS_USERNAME`, `OS_PASSWORD`, `REGION`
3. (WIP)
