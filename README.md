# Parts Unlimited, Veridian Dynamics, Sparkly Sprokets, Weyland-Yutani

<img src="https://www.digitalonus.com/wp-content/uploads/2019/06/digital_on_us_logo.png" width="300"><br>
<img src="https://cdn.worldvectorlogo.com/logos/hashicorp.svg" width="150">
<br><br>
[![N|Solid](https://cldup.com/dTxpPi9lDf.thumb.png)](https://nodesource.com/products/nsolid)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

## Documentation Sections

### Provisioning and Architecture

| Asset Name | Region 
| ------ | ------ |
| Primary Cluster | us-east-2 | 
| PR Cluster 1 | us-west-1 | 
| PR Cluster 2 | eu-central-1 | 
| DR Cluster 1 | eu-west-1 |
| DR Cluster 2 | ap-southeast-1 |

From an architectural viewpoint us-east-1 was discarded as the primary region and the most practical decision was to use us-east-2.

### Basic Accsess Catalog

| Asset Name | IP/Hostname
| ------ | ------ 
| Flask App |  |

### Database Dynamic Secrets Configuration
Database Dynamic secrets were enabled in order to work with the flask application following the steps below, for the MySQL/MariaDB database.

Database secrets were enabled:
```sh
vault secrets enable database
```
MySQL database configuration:
```sh
vault write database/config/my-mysql-database \
    plugin_name=mysql-database-plugin \
    connection_url="foo:foobarbaz@tcp(terraform-20200206144544828600000005.ccj5pugkq6mw.sa-east-1.rds.amazonaws.com:3306)/" \
    allowed_roles="my-role"
```
Role Map configuration
```sh
vault write database/roles/my-role \
    db_name=my-mysql-database \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```
Testing can be done with the next command
```sh
vault read database/creds/my-role
```
### Vault Agent Configuration
Enable App Role on Vault Cluster
```sh
vault auth enable approle
vault policy write token_update token_update.hcl
vault write auth/approle/role/apps policies="token_update"
vault read -format=json auth/approle/role/apps/role-id \
         | jq  -r '.data.role_id' > roleID
vault write -f -format=json auth/approle/role/apps/secret-id \
         | jq -r '.data.secret_id' > secretID
Policy (token_update.hcl)
```
Define policy to allow token creation:
```sh
path "auth/token/create" {
  capabilities = ["update"]
}
path "database/creds/*" {
  capabilities = ["read"]
}
path "aws/creds/*" {
  capabilities = ["update", "read", "create"]
}
path "transit/+/orders" {
 capabilities = ["update", "read", "create", "delete"]
}
```
### Configure Vault Transit Engine
```sh
vault secrets enable transit
vault write -f transit/keys/orders
```


### Configure Vault Agent

Config File (agent-config.hcl)
```sh
xit_after_auth = false
pid_file = "./pidfile"
auto_auth {
   method "approle" {
       mount_path = "auth/approle"
       config = {
           role_id_file_path = "/opt/vault/config/roleID"
           secret_id_file_path = "/opt/vault/config/secretID"
           remove_secret_id_file_after_reading = false
       }
   }
   sink "file" {
       config = {
           path = "/opt/vault/config/approleToken"
       }
   }
}
listener "tcp" {
  address = "0.0.0.0:8007"
}
vault {
   address = "http://internal-a3970b95-vault-lb-1327631726.sa-east-1.elb.amazonaws.com:8200"
}
template {
  source      = "/opt/flask/mysqldbcreds.json.ctmpl"
  destination = "/opt/flask/mysqldbcreds.json"
}
```

Create ConsulTemplate
````
Template in /opt/flask/mysqldbcreds.json.ctmpl
{{ with secret "database/creds/my-role" }}
{
  "username": "{{ .Data.username }}",
  "password": "{{ .Data.password }}",
  "hostname": "terraform-20200206144544828600000005.ccj5pugkq6mw.sa-east-1.rds.amazonaws.com"
}
{{ end }}
````

Run agent
```sh
vault agent -config=agent-config.hcl -log-level=debug
```
