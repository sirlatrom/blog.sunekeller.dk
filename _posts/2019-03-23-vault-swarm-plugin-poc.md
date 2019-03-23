---
title: 'Vault + Swarm Docker secrets plugin (proof of concept)'
tags: [docker,stack,deploy,swarm,configs,secrets,vault]
description: In this post, I show how to write and deploy a secrets plugin for Docker Swarm that will fetch its values from HashiCorp Vault.
excerpt: In this post, I show how to write and deploy a secrets plugin for Docker Swarm that will fetch its values from HashiCorp Vault.
uuid: e3c7fbd8-4d6f-11e9-af68-6b8b9002e2a1
---

# Background

Secrets have been part of Swarm Mode since its inception, making it trivial to provide generic, static secrets to your distributed services. However, not all secrets are equal, and some use cases call for a more dynamic approach. Docker Engine allows installing a plugin and using it as a driver when creating secrets, letting the value of the secret be determined at runtime, thus enabling dynamic use cases. My [talk at DockerCon 2019 in San Fransisco](https://dockercon19.smarteventscloud.com/connect/sessionDetail.ww?SESSION_ID=282000) will cover how to write a secrets plugin that fetches dynamic secret values from HashiCorp Vault, and how to deploy it as a Swarm service.

[Where I work](https://www.almbrand.dk), we deal with a complex mix of systems some of which are under our own control, and others over which we have none. If you consider the amount of environments in all the different systems involved all through the development and operations lifecycle of any individual service, you'll be right to guess that maintaining a coherent and up-to-date set of configurations is a daunting task. Some of this complexity can be handled with a careful amount of layering and separation of concerns, when a fitting set of tools is applied.

# The problem

## A short intermission about layered configuration in complex systems

Imagine you have your own system of records containing customer and product information. You likely have a staging environment and maybe a dev/test/QA environment in addition to your production environment. But so do most of the third-parties (service-providers, authorities, interest organizations, independent software vendors, etc.) that you integrate with. But more likely than not, they have a different set of environments made for different purposes. Cue the combinatorial explosion of environments.

Some configuration variables such as the endpoint URLs or credentials to the non-prod environments at other organizations will vary depending on what test data is available on their end, but many others will have the same values across multiple non-prod environments.

A naïve approach would be to keep a copy of the configuration for each environment, both for your own services and how they connect with third-parties. This will quickly become unmanageable, though, as the number of third-parties you integrate with increases and you add more services to your portfolio.

Another approach is to separate out the configuration variables that are different between the various environments into separate areas in your configuration management solution and then layer them on top of each other, choosing a suitable order of precedence. You could e.g. keep a "global" set of configuration variables at the lowest-precedence level and layer variables common for all non-prod environments on top of that, and finally sprinkle the variables specific to that snowflake environment on top as the final layer, the latter being given highest level of precedence. You could go even further and allow overriding certain variables for a specific deployment of a specific service.

All of the cases I just described have proven necessary at one point or another, and it reflects the approach we currently employ in the platform I work on on a daily basis.


# A solution

## Mixing in secrets and the principle of least privilege

Given the above approach, each service deployment should have access to only the areas of your configuration management solution that it needs to function, in order to pursue the principle of least privilege.

[HashiCorp Vault](https://www.vaultproject.io) is one of several available secrets management solutions, and to my personal taste it's probably the best one out there that can be deployed in a standalone scenario. It offers many different useful features, such as dynamic credentials generation for various systems (databases and others), with the notable core feature of "leases", meaning that access to secrets is always time-limited, and can be configured to be constrained to be limited to a certain number of reads.

Limiting what secrets each service can access reduces the impact of any one service being compromised. There are a bunch of things to consider, and I am by no means a security specialist. However, if you can [avoid passing secrets in environment variables](https://diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/), and audit (and if necessary revoke) secret access on a per-instance basis, I believe it's not a bad situation to be in, security wise.

## Configuration and secrets in Swarm

Docker Swarm can help distribute your configuration files and secrets to the individual service instances, and the required config and secret objects are distributed only to the nodes currently running service instances.

Meanwhile, Docker Compose offers a flexible and declarative way to create and update your secrets in Swarm, and, importantly, lets you specify a driver as well as labels for a secret, just like one can do on the command-line.

## How it works

When a task is scheduled to run on a node, the secrets attached to it are evaluated and sent together with the task spec. If a secrets driver plugin is set for the secret, the plugin will be asked what value the secret should have for that task. If the plugin returns the `DoNotReuse: true` value in its response to the secret value request, each separate task can possibly get its own individual value for the secret. This allows for multiple scenarios:

- Returning an enriched secret value (i.e. adding additional metadata as part of the returned secret value),
- Handing out single-use tokens to each task, allowing auditing of the individual service's use of secrets,
- Returning completely dynamic values, e.g. dynamically created database credentials, and more.

Specifically, if you create the secret and the service with the necessary labels on them, you'll be able to choose one or more Vault policies based on custom criteria and hand out a [response-wrapped](https://www.vaultproject.io/docs/concepts/response-wrapping.html) -token tailored to the service, giving access only to the secrets it absolutely needs to function.

## A proof of concept plugin

# Future work

Configs can be created with the `--template-driver` option, allowing you to insert placeholders for secrets in your config file and have those be resolved each time a task (container) for a service is created. There will be a `template_driver` equivalent in the Docker Compose file format eventually (here are the pull requests: [docker/cli#1746](https://github.com/docker/cli/pull/1746)+[docker/compose#6530](https://github.com/docker/compose/issues/6530)). Once that is in place (tentatively set for Compose file format version 3.8), you'll be able to combine configs and secrets _and_ secrets plugins to build a powerful and expressive config management solution, while keeping the concerns of the systems involved separated.

The source code for the proof of concept plugin is available at [GitLab](https://gitlab.com/sirlatrom/docker-secretprovider-plugin-vault/).
