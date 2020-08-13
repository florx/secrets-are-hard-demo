# secrets-are-hard-demo
A talk demo I've given, using Vault to manage dynamic credentials of a Postgres database.

This demo is also available in video format: 

[![Youtube Video](https://img.youtube.com/vi/dJuMYpgLeYA/0.jpg)](https://www.youtube.com/watch?v=dJuMYpgLeYA)


## The premise

As part of my talk titled "Secrets are hard to manage. How to manage your secrets and keep them safe", I describe Secret Sprawl, and Secret Management. One of the tools I recommend (though I have no affiliation to Hashicorp), is Hashicorp Vault.

See the slides here: https://www.slideshare.net/secret/3FBA3P7ZYSpN5A

## The demo

### Prerequisites 
* Vault: Ensure you have Vault installed. I prefer getting it direct (as then it includes the UI component): https://www.vaultproject.io/downloads.html
* PostgreSQL CLI tools: I use MacOSX so snagged this from Homebrew: `brew install postgresql`
* Docker Desktop: Have docker running locally so you can spin up containers easily. https://www.docker.com/products/docker-desktop (you have to login to download it, annoyingly)
* These instructions have been tested on MacOSX, so your mileage may vary.

### Let's run a thing

```bash
# Setup local a Vault server, in dev mode.
# This has basically no security enabled, so should *never* be run in prod.
vault server -dev

# Spin up a PostgreSQL database, with a specified default password
# It will make the network of this container available to our host machine too.
docker-compose -f postgres.yml up

# Put some random data in, we want to see if it works after all!
psql -h localhost -U admin -d users -a -f users.sql

# Point the Vault CLI at the right place, so we can use it.
export VAULT_ADDR='http://127.0.0.1:8200'

# Write a secret into Vault using a well known path:
# secret/app/users-service/dev
# I would recommend a password more secure than this in production...
vault kv put secret/app/users-service/dev \
    username=admin password=supersecret

# Get the value back out of Vault, to ensure it worked properly
vault kv get secret/app/users-service/dev

# Have a look at a glossy UI provided by Vault
# You'll need the root token, which is echoed into the terminal on starting the vault server
open http://127.0.0.1:8200/ui

# Begin the journey to enabling a database secrets backend.
vault secrets enable database

# Give Vault instructions on how to contact our database
# Note that both allowed_roles=* and sslmode=disable are not secure, and both should not be used in production.
vault write database/config/users-database \
    plugin_name=postgresql-database-plugin \
    allowed_roles="*" \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/users?sslmode=disable" \
    username="admin" \
    password="supersecret"

# Create Vault database role for limited read only access to all tables
vault write database/roles/read-only-users-database-human-role \
    db_name=users-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN \
    PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="10h"

# Get some credentials, Vault will automatically connect to our database, then create a brand new username and password for us
# This credential will be time limited to 1h initially, but allowed to renew the lease for up to 10h.
vault read database/creds/read-only-users-database-human-role 

# Renew our access to this credential, this is allowable up to 10h (based on our configuration) and will fail after 10h have passed.
# The lease_id comes from the above command
vault lease renew <lease_id>

# Roll the root database password, once we run this, we will personally lose root access to our own database
# We can never get the new password out of vault, resorting to using role only. This means no human eyes will ever be able to see this credential.
vault write -force \
    database/rotate-root/users-database
```

