# Deploying a PMK Host with cloud-init

## Steps

### I. Create a Cluster Object 
Create a cluster object. Nodes can join this new cluster. The output of this command will contain the UUID of the cluster object, which will be referenced in a later step. 
1) Obtain an auth token. Set the `DU_FQDN`, `OS_USERNAME`, `OS_PASSWORD` options. The final command will save the authentication token as an environment variable.  
```
export DU_FQDN="example.platform9.net"
export OS_USERNAME="user@email.com"
export OS_PASSWORD="strongpass"

export TOKEN=$(curl -D - -sH "Content-Type:application/json" https://$DU_FQDN/keystone/v3/auth/tokens -d '{"auth":{"identity":{"methods":["password"],"password":{"user":{"name":"'$OS_USERNAME'","domain":{"name": "Default"},"password":"'$OS_PASSWORD'"}}},"scope":{"project":{"domain":{"name":"Default"},"name":"'service'"}}}}' | grep -Ei '^X-Subject-Token' | awk '{print $2}' | tr -d '\r')
```
2) Create an empty cluster

Prior to creating the cluster, you will have to run API calls to get values for the following:
* **project_uuid** to be used for the two subsequent commands: 
```curl -s -k -H "X-Auth-Token: $TOKEN" https://$DU_FQDN/keystone/v3/projects | jq```
* **kubeRoleVersion**: 
```curl -s -k -H "X-Auth-Token: $TOKEN" https://$DU_FQDN/qbert/v4/<YOUR_PROJECT_UUID>/clusters/supportedRoleVersions | jq -r '.roles[].roleVersion'```
* **nodePoolUuid**: 
```curl -s -k -H "X-Auth-Token: $TOKEN" https://$DU_FQDN/qbert/v4/<YOUR_PROJECT_ID>/nodePools | jq -r '.[] | select(.name == "defaultPool") | .uuid'```
```
curl --request POST --url https://$DU_FQDN/qbert/v4/<project_uuid>/clusters --header "X-Auth-Token: $TOKEN" --header 'content-type: application/json' --data '{
"allowWorkloadsOnMaster": false,
"calicoIPv4DetectionMethod": "first-found",
"calicoIpIpMode": "Never",
"calicoNatOutgoing": true,
"calicoV4BlockSize": 26,
"containersCidr": "10.180.232.0/21",
"mtuSize": 1440,
"name": "rs-test-01",
"networkPlugin": "calico",
"nodePoolUuid": "23fad4fe-01f4-4239-b47c-1d906290c567",
"privileged": true,
"kubeRoleVersion": "1.24.7-pmk.240",
"servicesCidr": "10.180.240.0/22"
}'
```
 

### II. Use cloud-init to deploy a PMK node
This script downloads and executes the Platform9 CLI, pf9ctl, in non-interactive mode. The CLI will prepare the node for onboarding (applying prerequisites and ensuring all prerequisite are satisfied), downloads the Platform9 Hostagent service, and associates the node with the cluster object created in the previous step. 

1. Apply the following cloud-config script to the virtual machine or bare metal instance you are provisioning. 
Update the `pf9ctl config set` command with the management plane FQDN, an admin username and password, project name, and region. (Default project name is `service`, and default region name is `RegionOne`)
Update the `attach-node` command with the UUID of the cluster object created in the previous step. 
Ensure config drive option is enabled. 
```
#cloud-config
password: winterwonderland
chpasswd: { expire: False }
ssh_pwauth: True
manage_etc_hosts: true

write_files:
  - content: |
      #!/bin/bash
      OUTPUT=/root/register_node.log
      setenforce 0 || true
      curl -sL https://pmkft-assets.s3-us-west-1.amazonaws.com/pf9ctl_setup -o- | bash >$OUTPUT 2>&1
      pf9ctl config set -u "https://examplecustomer.platform9.net" -e "user@email.com" -p "strongpass" -t "service" -r "RegionOne" --no-prompt >$OUTPUT 2>&1
      nice -5 pf9ctl prep-node --skip-checks --no-prompt >$OUTPUT 2>&1
      nice -5 pf9ctl attach-node --no-prompt --uuid cc0314c1-696b-4cca-b730-d0250ecb11a2 --master-ip $(hostname --all-ip-addresses | cut -d " " -f1) >$OUTPUT 2>&1
    path: /root/register_node.sh
    permissions: "0755"

runcmd:
  - /root/register_node.sh
```
