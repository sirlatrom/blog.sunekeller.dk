---
redirect_from: /docker-18-03-config-and-secret-templating/
title: Using the new config and secret templating in Docker CE 18.03
tags: [docker,swarm,configs,secrets,intermediate,features,what's new,docker ce]
description: Since Docker CE 18.03, you can have the best of both worlds. Using the new --template-driver option to docker config create and docker secret create, you can insert secret references as well as other templating placeholders directly in your Swarm configs, evaluated at task creation time.
excerpt: Since Docker CE 18.03, you can have the best of both worlds. Using the new --template-driver option to docker config create and docker secret create, you can insert secret references as well as other templating placeholders directly in your Swarm configs, evaluated at task creation time.
image:
uuid: 2d5c1232-9daa-11e8-804c-0b1be24ee66c
---

# Background

A really helpful new feature just landed in Docker CE 18.03. Since [Swarm Secrets](https://docs.docker.com/engine/swarm/secrets/) were introduced in Docker 1.13 (January 2017) and [Swarm Configs](https://docs.docker.com/engine/swarm/configs/) came around in 17.06, getting sensitive data and configuration files securely distributed to your Swarm service containers is now more or less a solved problem. Using Swarm configs, you no longer have to bake configuration files into a custom image, or use external means to distribute the configuration files to all nodes in your Swarm cluster.

# The problem

However, if dense config files also contain secret data, what are your options? Putting the configuration file in a Swarm secret will make it cumbersome for operators to inspect the non-sensitive parts of the configuration, and using a Swarm config will make it too easy to reveal any secrets as part of daily operations.

[Some](https://store.docker.com/images/mariadb) [of](https://store.docker.com/images/wordpress) [the](https://store.docker.com/images/postgres) official images on Docker Store have built-in entrypoint scripts to help specifying the file containing e.g. the MariaDB root password by pointing an environment variable to a file, which could very well be a Swarm secret located in the `/run/secrets/` directory. However, most images have not been adapted to support Swarm secrets natively.

A simple example is [the official Redis image](https://store.docker.com/images/redis). Given that you wanted to set other configuration options than the password, you would have to do one of the following things:

* **Expose the password to operators:** Create a `redis.conf` Swarm config including the password in plaintext, visible to everyone who can inspect the config through the Docker API or who looks over the shoulders of such a person.
* **Hide all config from operators:** Put the entire `redis.conf` file inside a Swarm secret, making it hard for operators to see the non-sensitive contents of the Redis configuration. If they copy out the contents from the container, they will still have unwillingly exfiltrated the secret password and risked its exposure.
* **Hack together some custom templating:**
  1. Create a `redis.conf` Swarm config with a placeholder for the password,
  2. Create a Swarm secret with the password,
  3. Add a custom entrypoint script to replace it with the contents of the correct secret file inside `/run/secrets/`,
  4. Write a Dockerfile which includes the entrypoint script, distribute the resulting image to your Swarm through a public or private registry and keep track of official image updates forever, repeating this step for every important feature or security update,
  5. Become miserable because of point 4.

As you can see, the choices are less than ideal.

# The improvement

Since [Docker CE 18.03](https://docs.docker.com/release-notes/docker-ce/#18030-ce-2018-03-21), however, you can have the best of both worlds. Using the new `--template-driver` option to `docker config create` and `docker secret create`, you can now use the Golang templating language to insert secret references as well as other templating placeholders directly in your Swarm configs and have them rendered only when each service task is created.

The possibilities listed in the relevant pull request on the [Moby project](https://github.com/moby/moby/pull/33702) are:

* {% raw %}`{{ env "VAR" }}`{% endraw %}
* {% raw %}`{{ .Task.ID }}`{% endraw %}
* {% raw %}`{{ secret "sometarget" }}`{% endraw %}[^1]

I've discovered that you can also use {% raw %}`{{ config "sometarget" }}`{% endraw %}, though I'd be careful with introducing obscure dependencies between different Swarm configs.

Adding documentation is tracked in [this GitHub issue](https://github.com/docker/docker.github.io/issues/6207).

# Example

Here's a step-by-step example to illustrate the feature:

1. Create a Swarm secret with the intended password for Redis:

   ```console
   $ openssl rand -hex 32 | tr -d '\n' | docker secret create redis_pw -
   s3agzeg68rvf2nhimk20mjm05
   ```

2. Create a `redis.conf` file e.g. using the [default Redis config](http://download.redis.io/redis-stable/redis.conf),

3. Change the TCP port in `redis.conf` to show we're changing the defaults, something we'd like to be able to see as an operator:

   ```text
   port 2100
   ```

4. Set the `requirepass` option in `redis.conf` as a _templated secret reference_, the value of which we'd like to _avoid_ be able to read during daily operations:

   ```go
   {% raw %}requirepass {{ secret "redis_pw" }}{% endraw %}
   ```

5. Create a Swarm config for `redis.conf` using the new `--template-driver` option, and inspect its contents to show that the `redis_pw` secret is not revealed:

   ```console
   {% raw %}$ docker config create --template-driver golang redis.conf ./redis.conf
   $ docker config inspect --pretty redis.conf | grep '^requirepass'
   requirepass {{ secret "redis_pw" }}{% endraw %}
   ```

6. Create an attachable overlay network and a Swarm service using the official Redis image:

   ```console
   $ docker network create --driver overlay --attachable redis_network
   $ docker service create \
     --name redis \
     --secret redis_pw \
     --network redis_network \
     --config source=redis.conf,target=/usr/local/etc/redis/redis.conf \
     redis:alpine \
     redis-server /usr/local/etc/redis/redis.conf
   ```

7. Try running a Redis CLI against the Redis service, and you should be rejected:

   ```console
   $ docker run --rm --network redis_network redis:alpine redis-cli -h redis -p 2100 set x "I'm in"
   NOAUTH Authentication required.
   ```

8. Try running a CLI in a service with access to the secret:

   ```console
   $ docker service create \
     --detach \
     --name redis-cli-test \
     --secret redis_pw \
     --network redis_network \
     --restart-condition on-failure \
     redis:alpine \
     sh -c 'redis-cli -h redis -p 2100 -a "$(cat /run/secrets/redis_pw)" set x "I'"'"'m in"; redis-cli -h redis -p 2100 -a "$(cat /run/secrets/redis_pw)" get x'
   ```

9. Wait a couple of seconds - currently we have to specify `--detach` with `--restart-condition on-failure` since the synchronous `docker service create` command (without `--detach`) interprets a task in state `Completed` to be a failure, and will not return you to the command line.[^2]

10. Inspect the logs to see it working, and you should get the following result:

    ```console
    $ docker service logs --raw redis-cli-test
    OK
    I'm in
    ```

[^1]: The example says ```"some_target"``` because it is possible to specify a source and a target for a secret when adding it to a Swarm service like so: ```--secret source=prod_redis_pw,target=redis_pw```, and you have to use the _target_ name rather than the _source_ name in the templated reference in your Swarm config.
[^2]: Note to self: I should create an issue in [docker/cli](https://github.com/docker/cli) for this.
