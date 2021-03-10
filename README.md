# AppRole authentication method with Vault tutorial

## Instructions

- Clone this repository:
```shell
$ git clone git@github.com:dlavric/approle-vault.git
```

- Start vault dev server with root as root token:
```shell
$ vault server -dev -dev-root-token-id root
```

- Open a new terminal, Export the env variable:
```shell
$ export VAULT_ADDR="http://127.0.0.1:8200"
```

- Enable the AppRole authentication method at the default path:
```shell
$ vault auth enable approle
```

- Create the hcl file for the jenkins policy:
```shell
$ tee jenkins-pol.hcl <<EOF
# Read-only permission on 'secret/data/mysql/*' path
path "secret/data/mysql/*" {
  capabilities = [ "read", "update" ]
}
EOF
```

- Write the jenkins policy:
```shell
$ vault policy write jenkins jenkins-pol.hcl
```

- Create the 'jenkins' role with the attached policy:
```shell
$ vault write auth/approle/role/jenkins token_policies="jenkins" \
        token_ttl=1h token_max_ttl=4h
```
Output:
```
Success! Data written to: auth/approle/role/jenkins
```

- Read the jenkins role to verify it:
```shell
$ vault read auth/approle/role/jenkins
```
Output:
```
Key                        Value
---                        -----
bind_secret_id             true
local_secret_ids           false
secret_id_bound_cidrs      <nil>
secret_id_num_uses         0
secret_id_ttl              0s
token_bound_cidrs          []
token_explicit_max_ttl     0s
token_max_ttl              4h
token_no_default_policy    false
token_num_uses             0
token_period               0s
token_policies             [jenkins]
token_ttl                  1h
token_type                 default
```

- Get the RoleID:
```shell
$ vault read auth/approle/role/jenkins/role-id
```

- Get the SecretID:
```shell
$ vault write -f auth/approle/role/jenkins/secret-id
```

- Login with the RoleID and the SecretID:
```shell
$ vault write auth/approle/login role_id="7c056db0-9c10-0933-94f2-622786ff3a97" \
  secret_id="5180abc3-3a63-f36c-c64b-a5895b66ba7a"
```
Output:
```
Key                     Value
---                     -----
token                   s.MrTV5py363LGTNIPlatiZIjs
token_accessor          K244iNYRZDIUWG0YCR0Kna1s
token_duration          1h
token_renewable         true
token_policies          ["default" "jenkins"]
identity_policies       []
policies                ["default" "jenkins"]
token_meta_role_name    jenkins
```

- Pass the client token:
```shell
$ VAULT_TOKEN=s.MrTV5py363LGTNIPlatiZIjs vault kv get secret/mysql/webapp
```
Output:
```
No value found at secret/data/mysql/webapp
```

- Login with the approle token:
```shell
$ vault login s.MrTV5py363LGTNIPlatiZIjs
```
Output:
```
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                     Value
---                     -----
token                   s.MrTV5py363LGTNIPlatiZIjs
token_accessor          J4AYriU03DvG2KxaB1TrJcE9
token_duration          59m18s
token_renewable         true
token_policies          ["default" "jenkins"]
identity_policies       []
policies                ["default" "jenkins"]
token_meta_role_name    jenkins
```

- Get the secrets from the path 'secret/mysql/webapp':
```shell
$ vault kv get secret/mysql/webapp
```
Ouput:
```
No value found at secret/data/mysql/webapp
```
- Create a JSON file containing the data you wish to store:
```shell
$ tee mysqldb.json <<EOF
{
  "url": "foo.example.com:35533",
  "db_name": "users",
  "username": "admin",
  "password": "pa$$w0rd"
}
EOF
```

- Login as root:
```shell
$ vault login root
```

- Create the secrets at the path 'secret/mysql/webapp'. 
The input data will be read from the mysql.json file:
```shell
$ vault kv put secret/mysql/webapp @mysqldb.json
```
Output:
```
Key              Value
---              -----
created_time     2021-03-10T13:56:02.550578Z
deletion_time    n/a
destroyed        false
version          1
```

- Get the secrets from the path:
```shell
$ vault kv get secret/mysql/webapp
```
Output:
```
====== Metadata ======
Key              Value
---              -----
created_time     2021-03-10T13:56:02.550578Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
db_name     users
password    pa26939w0rd
url         foo.example.com:35533
username    admin
```
