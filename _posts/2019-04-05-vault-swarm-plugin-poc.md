---
redirect_from: /2019/03/vault-swarm-plugin-poc/
title: 'Vault + Swarm Docker secrets plugin (proof of concept)'
tags: [docker,stack,deploy,swarm,configs,secrets,vault]
description: In this post, I show how to write and deploy a secrets plugin for Docker Swarm that will fetch its values from HashiCorp Vault.
excerpt: In this post, I show how to write and deploy a secrets plugin for Docker Swarm that will fetch its values from HashiCorp Vault.
uuid: e3c7fbd8-4d6f-11e9-af68-6b8b9002e2a1
---

# Background

Secrets have been part of Swarm Mode since its inception, making it trivial to provide generic, static secrets to your distributed services. However, not all secrets are equal, and some use cases call for a more dynamic approach. Docker Engine allows installing a plugin and using it as a driver when creating secrets, letting the value of the secret be determined at runtime, thus enabling dynamic use cases. My [talk at DockerCon 2019 in San Fransisco](https://dockercon19.smarteventscloud.com/connect/sessionDetail.ww?SESSION_ID=282000) will cover how to write a secrets plugin that fetches dynamic secret values from [HashiCorp Vault](https://www.vaultproject.io), and how to deploy it as a Swarm service.

## Static vs. dynamic secrets

Generic, static secrets will only get you so far. Once you get to a large enough number of secrets, you'll either need a very good naming convention, or make sure you label secrets very carefully. Even then they might become cumbersome to manage, which risks either creating too broad policies, or drowning yourself in bureaucracy. And there are other secret management solutions out there, and in this post I will discuss a specific use case with HashiCorp Vault.

## Basic example of _static_ secrets

Here's a basic but complete example (adapted from the [official documentation](https://docs.docker.com/engine/swarm/secrets/#use-secrets-in-compose)) of using the built-in secrets feature:

First, create passwords for a database:

```console
$ cat /dev/urandom | tr -dc '0-9a-zA-Z!@#$%^&*_+-' | head -c 15 | docker secret create db_password -
$ cat /dev/urandom | tr -dc '0-9a-zA-Z!@#$%^&*_+-' | head -c 15 | docker secret create db_root_password -
```

Then write a Docker Compose file:

```yaml
version: "3.7"
services:
  db:
    image: mysql:latest
    command: "--default-authentication-plugin=mysql_native_password" # See https://github.com/docker-library/wordpress/issues/313#issuecomment-400836783
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_password
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    ports:
      - published: 8000
        target: 80
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
secrets:
  db_password:
    external: true
  db_root_password:
    external: true
volumes:
  db_data:
```

Finally, deploy the stack:

```console
$ docker stack deploy --compose-file docker-compose.yml example1
```

Wait a while for the database and blog app to start up, and you'll be able to visit [http://localhost:8000](http://localhost:8000) and see the working Wordpress site, all without you ever seeing the password.

As you can see in the Docker Compose file, both MySQL and WordPress are instructed to use files inside the `/run/secrets` directory for the database passwords. This is the delivery method selected for secrets.

## How static secrets work inside Swarm

When you use Swarm secrets _without_ a plugin, secret data and metadata is saved in the Raft store of the Swarm managers. By design, secret data cannot be updated, and the Docker CLI offers no commands to update secret metadata (i.e. labels). You _can_, however, update secret _labels_ through the [Docker Engine API](https://docs.docker.com/engine/api/v1.39/#operation/SecretUpdate).

Once you create a service with a secret attached, the secret values are placed as files on a private tmpfs (i.e. an in-memory file-system) mounted inside the container, rather than in environment variables, which are [too easily divulged to unconcerned parties](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/).

# The problem

You can configure Vault with e.g. your database's sysadmin credentials, and then use a combination of policies and authentication mechanisms to have Vault dynamically create time-limited user accounts with role-based grants. The most basic authentication method Vault offers based on [opaque tokens](https://www.vaultproject.io/docs/concepts/tokens.html). Tokens can have policies attached to them, indicating what areas in Vault they give access to. But how do you get them to your containers?

You _could_ of course create a static, long-lived token with the right policies attached, and then type that in as a static secret in Swarm. Then you could attach it to the relevant service, and the service could then communicate directly with Vault to read the secret data, e.g. database credentials. But then you get the problem of rotating the token you typed into Swarm, which either becomes a bureaucratic, repetitive task, or else you risk having to put that secret into your CI/CD system, which can lead to [secret sprawl](https://www.hashicorp.com/resources/what-is-secret-sprawl-why-is-it-harmful). If you _don't_ rotate the token, you instead run the risk of the token some day being intercepted, and then you _have_ to rotate it - if you notice, that is.

Ideally, you would give a new token to every instance of your service, and use the use-limit feature of Vault to make sure you can detect interception and ensure a stolen token cannot be reused. You can read more about this concept, which Vault calls [Response Wrapping](https://www.vaultproject.io/docs/concepts/response-wrapping.html#overview). However, with static Swarm secrets, there is no way of making use of response wrapping. If you typed a response wrapped token into Swarm and made use of it in a service, only the first instance would be able to make use of it, which is by design.

# A solution

In order to solve this challenge in a satisfying way, you'll need to use one of the several extension points of Docker Swarm.

## Introduction to the pluggable secrets backend

The _pluggable secrets backend_ allows you to specify a "driver" when creating a secret, e.g. `docker secret create --driver <driver_name> <name> <file|->`. The plugin must advertise that it implements the `"secretprovider"` interface, and Docker provides a [helpful repository](https://github.com/docker/go-plugins-helpers/blob/master/secrets/api.go) for getting started with writing such a plugin in Go.

When a driver is chosen for a secret, the Swarm manager still looks up the _metadata_ in the raft store, but will request the _data_ from the plugin with the given driver name. The corresponding plugin _must_ be installed on the _managers_ in the Swarm.

[Liron Levin](https://github.com/liron-l) from TwistLock contributed the pluggable secrets backend back in [2017](https://github.com/docker/swarmkit/pull/2239).

## A proof-of-concept plugin

My idea was to write a plugin that when used will call out to Vault to deliver secret values to Swarm service tasks. One of the requirements was that it should support response wrapping. It was not hard to write, given the [go-plugin-helpers](https://github.com/docker/go-plugins-helpers/blob/master/secrets/api.go) repo and the excellent official [Vault Go client](https://www.vaultproject.io/api/libraries.html#go).

The plugin works like this:

1. Receive request, including the secret name and labels, service name, ID and labels, the task name and ID
2. Based on the secret labels, the plugin will then
3. Create a token on behalf of the service task, with a Vault policy on the token with the same name as the service, and then, _optionally_:
    1. Use that token to read a [generic key/value secret](https://www.vaultproject.io/api/secret/kv/kv-v2.html) from a specified path, and _optionally_:
        1. Return a specific field inside that path
        2. JSON-encode the returned value
    2. Optionally use response wrapping to deliver the returned value

The source code for the proof of concept plugin is available on [GitLab](https://gitlab.com/sirlatrom/docker-secretprovider-plugin-vault/).

### A complication

Now, when I first tried out the plugin, it worked as intended. However, when I scaled up the service to 2 replicas, I noticed two things:

1. When setting the secret label that indicated that a generic token should be returned, sometimes the two replicas got the same value, and
2. When setting the label to use response wrapping, sometimes only one of the tasks would succeed, whereas the other would be told by Vault that the response wrapping token had already been used.

When investigating this puzzling finding, I read through some of the code that assigns secrets and other resources to Swarm nodes. The code responsibly caches the values of secrets such that for each node, any given secret is only requested once from either the raft store or the secret plugin. However, that does not help if the plugin returns values that are supposed to be individual for each task, e.g. when using response wrapping.

I set about writing the necessary changes as PRs [#2735](https://github.com/docker/swarmkit/pull/2735/) and [#2735](https://github.com/docker/swarmkit/pull/2735/) for docker/swarmkit. They are currently merged/vendored in [moby/moby](https://github.com/moby/moby/pull/38123)'s master branch, and _should_ release with Docker 19.03.

### Limitations

Until Docker 19.03, you cannot use response wrapping with the plugin. It goes without saying that I do _not_ recommend or even suggest using this plugin anywhere near production, or even in daily use. Rather see it as an example of what _can_ be done with Swarm and its extension points.

Also, the plugin currently relies on getting its _own_ access to Vault (a more privileged token that can perform the plugin's functions) through suboptimal means. Because plugins in Docker, even if run as containers, are currently very different from regular containers, there are several features you cannot make use of. Namely, even though you can have a special type of Swarm service to install the plugin in your Swarm, there is currently no way for you to attach Swarm secrets to such a service, static or otherwise. This leaves you with the problem of safely bootstrapping the plugin itself. The only method I could think of, which really is half-baked, but works in this POC, is to give the plugin access to the Docker socket of the manager node it is installed on, and use a helper service to hold the bootstrapping token. The plugin then uses the Docker API to find the helper service's container and reads the bootstrapping token from there.

# Future work

Configs can be created with the `--template-driver` option, allowing you to insert placeholders for secrets (as described [here]({% post_url 2018-04-01-docker-18-03-config-and-secret-templating %})) in your config file and have those be resolved each time a task (container) for a service is created. There will be a `template_driver` equivalent in the Docker Compose file format eventually (here are the pull requests: [docker/cli#1746](https://github.com/docker/cli/pull/1746)+[docker/compose#6530](https://github.com/docker/compose/issues/6530)). Once that is in place (tentatively set for Compose file format version 3.8), you'll be able to combine configs _and_ secrets _and_ secrets plugins to build a powerful and expressive config management solution, while keeping the concerns of the systems involved neatly separated.

My dream is to be able to write a docker-compose file like this (obviously made-up and not realistic):
```yaml
version: "3.8"
services:
  app:
    image: ...
    configs:
      - source: config.yml
        target: /etc/app/config.yml
    secrets:
      - vault_token
configs:
  config.yml:
    template_driver: golang
    file: config.yml.tmpl
secrets:
  vault_token:
    name: vault_token
    driver: sirlatrom/docker-secretprovider-plugin-vault
    labels:
      dk.almbrand.docker.plugin.secretprovider.vault.type: "vault_token" # Secret will contain a Vault token
      dk.almbrand.docker.plugin.secretprovider.vault.wrap: "true"        # Enable response wrapping
```

and a config.yml file like this:

```yaml
vault_addr: https://vault.example.com:8200
vault_token: {%raw%}{{ secret "password" }}{%endraw%}
```

and the config file would contain a response wrapped long-lived token that the app could then use.

**Wouldn't that be great?**
