---
title: 'Docker stack deploy: update configs and secrets'
tags: [docker,stack,deploy,swarm,configs,secrets,ci,cd]
description: In this post, I explain how you can use CI variables or file digests to update configs and secrets during docker stack deploy.
excerpt: In this post, I explain how you can use CI variables or file digests to update configs and secrets during docker stack deploy.
uuid: 28f1e06e-2590-11e9-a2a7-33a1ce4af04f
---

# Background

If you've ever deployed a Docker stack using `docker stack deploy --compose-file docker-compose.yml <stack_name>`, and the `docker-compose.yml` file has configs in it referring to files in your working directory, you may have hit a snag when you wanted to update the contents of those configs.

# The problem

In Swarm Mode, configs and secrets are immutable objects with unique names, and there is no way to mutate their contents. You _can_ update a _service_, though, to make it refer to a different config or secret.

Say you have this docker-compose.yml file:

```yaml
version: '3.6'
services:
  app:
    image: nginx
    configs:
      - source: nginx.conf
        target: /etc/nginx/nginx.conf
    secrets:
      - cert.pem
configs:
  nginx.conf:
    file: nginx.conf
secrets:
  cert.pem:
    file: cert.pem
```

If you have the `nginx.conf` and `cert.pem` files in your working directory, Docker will read their contents and create the config and secret with their content, and add references to them to the spec of your `app` service.

However, if you change the contents of either file, and try and run `docker stack deploy ...` again, you will get an error message saying you cannot create the config or secret, because it already exists.

# A solution

If you expand the `configs:` and `secrets:` top-level sections by adding a name to each entry, and add an appropriate environment variable as part of that name, you'll get the desired result.

If you're working interactively, you can use the `${LINENO}` shell variable which will increase with every command you enter.

If on the other hand you're in a CI/CD type of situation, you can choose one of several methods:

1. Use a job variable such as `${CI_JOB_ID}` in GitLab or `${BUILD_NUMBER}` in Jenkins
2. Calculate a digest of each referenced file and use that to determine whether a config or secret should be updated

The result of method 1 would look like:

```yaml
version: '3.6'
services:
  app:
    image: nginx
    configs:
      - source: nginx.conf
        target: /etc/nginx/nginx.conf
    secrets:
      - cert.pem
configs:
  nginx.conf:
    name: nginx.conf-${CI_JOB_ID}
    file: nginx.conf
secrets:
  cert.pem:
    name: cert.pem-${CI_JOB_ID}
    file: cert.pem
```

As for method 2, to use digests, you could decide on a certain convention for the variable names like this:

```yaml
version: '3.6'
services:
  app:
    image: nginx
    configs:
      - source: nginx.conf
        target: /etc/nginx/nginx.conf
    secrets:
      - cert.pem
configs:
  nginx.conf:
    name: nginx.conf-${nginx_conf_DIGEST}
    file: nginx.conf
secrets:
  cert.pem:
    name: cert.pem-${cert_pem_DIGEST}
    file: cert.pem
```

Then you'd have to run a script along the lines of the following to generate the digests and deploy the stack. Notice that configs and secrets cannot have names exceeding 64 characters, which complicates the script a little. You'll also need the `json_xs` tool (found in the `libjson-xs-perl` package in Ubuntu) and the `jq` tool (in the `jq` package).

```bash
# Get a list of the keys, names and files of configs and secrets
json_xs -f yaml -t json < docker-compose.yml | jq --raw-output '(.configs,.secrets) | to_entries | map(select(.value | has("file")) | .key, .value.name, .value.file)[]' > configs_and_secrets.txt
# Iterate over each three-tuple
while read entry; read name; read file
do
  # Sanitize the variable name for the digest
  sanitized_filename="$(echo $file | sed 's/[./ ]/_/g')"
  echo "$entry.name: $sanitized_filename"
  # Get the part of the name without any variable references
  name_without_references="$(env -i envsubst <<< "${name}")"
  remainder=$(( 64 - ${#name_without_references} ))
  # Export a variable with the digest, truncate to a total of 64 characters
  export ${sanitized_filename}_DIGEST="$(sha512sum $file | awk '{print $1}' | cut -c -${remainder})"
  echo "Use variable ${sanitized_filename}_DIGEST for $entry"
done < configs_and_secrets.txt
# Deploy the stack
docker stack deploy --prune -c docker-compose.yml stack
# Clean up
rm configs_and_secrets.txt
```

Notice the `--prune` flag; it _should_ take care of removing the configs and secrets from the previous deployment _next_ time you deploy, since they'll no longer be referenced by any services.