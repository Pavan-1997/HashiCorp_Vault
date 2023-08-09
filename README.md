HashiCorp Vault is an open-source tool designed to manage secrets and protect sensitive data in modern computing environments. It provides a secure and centralized solution for storing and accessing credentials, encryption keys, and other confidential information. Vault follows the principles of the "secrets as a service" concept, ensuring that secrets are securely stored and accessed only by authorized users and applications.

Below are the providers:

![image](https://github.com/Pavan-1997/HashiCorp_Vault/assets/32020205/859f25f0-77ce-4fdc-b3c4-6260d169bade)

---

Step1: Launch any server considering AWS EC2 Ubuntu here


Step2: Add PGP for the package signing key
```
sudo apt update && sudo apt install gpg
```


Step3: Download the signing key to a new keyring
```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```


Step4: Verify the key's fingerprint
```
gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint
```


Step5: Add Hashicorp Repo
```
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) test" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```


Step6: Update and Install Vault
```
sudo apt update && sudo apt install vault
```


Step7: To check the Vault Version
```
vault --version
```


---
### To start and stop the vault server can be done in dev mode and server mode(to run on a production)

---> To start the Vault server in Development Mode:
```
vault server -dev
```
- Need to check componenets -  Port, Storage, Unseal Key and Root Token
- Storage is inmem when using Vault in dev mode where all creds are stored locally


---> We need to export address and token 
```
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN="hvs.fPagmu29uyGNgnkBOD7lLJI2"
```

---> Check the status of the Vault Server
```
vault status
```


---
### To read, write and delete secrets:

---> WRITE:
```
vault kv put <custom-path> key-1=value-1
```

For enabling a custom path in the secret engine we use below
```
vault secrets enable -path=<custom-path> kv
```
`Eg: vault secrets enable -path=my kv`
	
`vault kv put my/path my-key-1=value-1`

---> READ:
```
vault kv get my/path
```
```
vault kv get -format=json  my/path
```
(To view the secret in JSON)

```
vault secrets list
```
(To get all secrets available at the particular path)

---> DELETE:
```
vault kv delete my/path
```

---

---> To verify all Secret Engine path availbe on Hashicorp Server
```
vault secrets list
```
---> To enable AWS Secret Engine path
```
vault secrets enable -path=aws aws
```
```
vault secrets list
```
(To verify)

---> To disable AWS Secret Engine path
```
vault secrets disable aws
```
```
vault secrets list
```
(To verify)
```
vault secrets disable <custom-path>
```
(To disable custom created secret engine path)



---

### AWS Dynamic Secrets Generation 

---> To enable AWS Secret Engine path
```
vault secrets enable -path=aws aws
```
```
vault secrets list
```
(To verify)

---> Set the root config using access-key and secret-key
```
vault write aws/config/root access-key=AKIAVS7NOQMV3LMDKZXB secret-key=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 region=us-east-2
```
---> Need to setup role
```
vault write aws/roles/my-ec2-role2 \
        credential_type=iam_user \
        policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
```

---> Generate access key and secret key for that role
```
vault read aws/creds/my-ec2-role1
```
---> To destroy the dynamic generated creds
```
vault lease revoke aws/creds/my-ec2-role1/<lease-id>
```


-----------------------------------------------------------------

VAULT POLICY

---> List the policies

vault policy list

---> Write your custom policy

vault policy write my-policy - <<EOF
# Dev servers have version 2 of KV secrets engine mounted by default, so will
# need these paths to grant permissions:
path "secret/data/*" {
  capabilities = ["create", "update"]
}

path "secret/data/foo" {
  capabilities = ["read"]
}
EOF

vault policy list
(To verify)

---> TO read any policy created

vault policy read my-policy

---> Delete Vault policy by policy name 

vault policy delete my-policy

--->  Attach token to policy 

export VAULT_TOKEN="$(vault token create -field token -policy=my-policy)"

---> Write a secret with the policy

vault kv put -mount=secret creds password="my-long-password"

---> Verify write operation on foo which should ideally restrict

vault kv put -mount=secret foo robot=beep



-----------------------------------------------------------------

AUTH METHODS AND POLICIES

---> To check the auth list 

vault auth list

---> To enable a auth method

vault auth enable appprole

---> Associate policy to auth method

vault write auth/approle/role/my-role \
   secret_id_ttl=10m \
   token_num_uses=10 \
   token_ttl=20m \
   token_max_ttl=30m \
   secret_id_num_uses=40 \
   token_policies=my-policy

---> Generate and Export Role ID

export ROLE_ID="$(vault read -field=role_id auth/approle/role/my-role/role-id)"

---> Generate and Export Secret ID

export SECRET_ID="$(vault write -f -field=secret_id auth/approle/role/my-role/secret-id)"

---> Now perform a write config 

vault write auth/approle/login role_id="$ROLE_ID" secret-id="$SECRET_ID"



-----------------------------------------------------------------

--->Vault login using root token

vault login 
(Provide the root token generated from the start of vault server in dev-mode)

---> To re-create a root token

vault token create 

---> To remove already generated root tokens

vault token revoke <token>



-----------------------------------------------------------------

---> To enable GitHub authentication

vault auth enable github


* If get errror Error enabling github auth: Post "https://127.0.0.1:8200/v1/sys/auth/github": http: server gave HTTP response to HTTPS client then perform below

export VAULT_ADDR='http://127.0.0.1:8200'

export VAULT_TOKEN="hvs.fPagmu29uyGNgnkBOD7lLJI2"

---> To verify the auth list

vault auth list

---> To create Orgazisation in vault

vault write auth/github/config organization=Sample-Test-Pavan

---> To create Team 

vault write auth/github/map/teams/Sample-Test-Pavan-Team value=default,application

---> Vault login using Github Method and give a token generated from GitHub

vault login -method=github

---> Revoke Github Authentication

vault token revoke -mode path auth/github

---> Disable Github Authentication

vault auth disable github



-----------------------------------------------------------------

Start the Vault in Production make sure you disable the dev mode prior

---> To stop the dev mode vault

Ctrl+C

---> Unset development token

unset VAULT_TOKEN

---> Vault's config.hcl (Before doing below make a directory mkdir -p ./vault/data and later create a touch config.hcl)

storage "raft" {
  path    = "./vault/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = "true"
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
ui = true

---> Starting vault server using config.hcl 

vault server -config=config.hcl

---> Export VAULT_ADDR

export VAULT_ADDR='http://127.0.0.1:8200'

---> Initialize vault

vault operator init


Sealed state means no R/W

Unsealed state means R/W operations

---> Unseal vault

vault operator unseal
