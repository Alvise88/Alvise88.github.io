---
title: "Swarm Ingresses"
date: 2022-03-07T06:36:47Z
---

In Kubernetes, we would consider a “service” to be a network entity that makes it possible to reach individual containers.  
In Swarm, however, a “service” means something completely different.

A Swarm service is the equivalent of a container and all of the information needed to instantiate and run it.  

For example:  

```docker
version: '3'
  networks:
    default:
    external: false

  services:
    whoami:
       image: traefik/whoami
    networks:
      - default
    deploy:
      replicas: 3
```

There are two types of Swarm services. Replicated services are instantiated as many times as you’ve requested.  
If you ask for three of a replicated service, you get three.  
If you ask for 10, you get 10, and so on.  
Global services, on the other hand, are like Kubernetes DaemonSets, in that you have one instance running on each node.

As an orchestration engine, Docker Swarm decides where to put workloads, and then manages requests for those workloads. Swarm allows you to use multiple load balancing mechanisms, including Spread, which tries to keep the usage of each node as low as possible, and BinPack, which tries to use as few servers as possible to handle requests from within the cluster.  

Internally, Swarm assigns each service its own DNS entry, and when it receives a request for that entry, it load balances requests between different nodes hosting that service.  

Although Docker is included with environments aimed at developers, such as Docker Desktop, it’s also available for production environments.  

One of the advantages of working with Docker Swarm is that developers can use the exact same artifacts — including the stack definition — in both their development and production environments.  

In the last few years, there were some rumors in regards to Docker Swarm and the future of that technology. 
However, based on the news published recently we can assume that Docker Swarm will be still supported.  

I don’t tell you that you have to use Docker Swarm. This is one of the options you choose among a few available orchestration tools.  

You can leverage Docker Swarm by installing [Traefik](https://doc.traefik.io/traefik/) that is an open-source router for microservice architectures and that article shows how to use those tools together.

Basically speaking Traefik can be considered an ingress controller for Swarm.  

By the end of 2019, there was a release of Traefik 2 that introduces a few crucial changes. The most important is to change the naming of key features of the system. If you are familiar with Traefik v.1 you probably know to name such as fronted, backend and rules.  

In V 2 we have:

* router that replaced fronted 
* service that replaced backend
* middleware that replaced rules

Using traefik with swarm allows you to define ingresses dynamically.  

The idea is to have a main load balancer/proxy that covers all the Docker Swarm cluster and handles HTTPS certificates and requests for each domain.

But doing it in a way that allows you to have other Traefik services inside each stack without interfering with each other, to redirect based on path in the same stack (e.g. one container handles / for a web frontend and another handles /api for an API under the same domain), or to redirect from HTTP to HTTPS selectively.

<!--adsense-->

Preparation:

* Connect via SSH to a manager node in your cluster (you might have only one node) that will have the Traefik service.
* Create a network that will be shared with Traefik and the containers that should be accessible from the outside, with:
  ```shell
  docker network create --driver=overlay traefik-public
  ```
* Create an environment variable with a username (you will use it for the HTTP Basic Auth for Traefik and Consul UIs), for example:
  ```shell
  export USERNAME=admin
  ```
* Create an environment variable with the password, e.g.:
  ```shell
  export PASSWORD=s3cret
  ```
* Use openssl to generate the "hashed" version of the password and store it in an environment variable:
  ```shell
  export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
  ```

* Create a compose file called traefik.yml 

  ```docker
  version: "3.8"

  x-default-opts: &default-opts
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: 10

  x-constraints: &constraints
    # - node.role == worker
    - node.platform.os == linux
    # - node.labels.datacenter == ponzano

  x-preferences: &preferences
    - spread: node.labels.datacenter

  networks:
    traefik-public:
      external: true

  configs:
    routers-config:
      name: routers-config-${CONFIG:-1}
      file: ./conf.d/routers.toml
    middlewares-config:
      name: middlewares-config-${CONFIG:-1}
      file: ./conf.d/middlewares.toml
    tls-config:
      name: tls-config-${CONFIG:-1}
      file: ./conf.d/tls.toml
    canary-config:
      name: canary-config-${CONFIG:-1}
      file: ./conf.d/canary.yml
    traefik-config:
      name: traefik-config-${CONFIG:-1}
      file: ./traefik.yml

  services:

    traefik:
      <<: *default-opts
      # The official v2.6.1 Traefik docker image
      image: traefik:v2.6.1
      healthcheck:
        test: traefik healthcheck --ping
        interval: 10s
        timeout: 5s
        retries: 3
        start_period: 45s
      ports:
        # The HTTP port
        - "80:80"
        - "443:443" #Docker sends requests on port 443 to Traefik on port 443
        - "8080:8080"
        - "8082:8082" #Traefik metrics
      networks:
        - traefik-public
    configs:
        # Dynamic config
        - source: routers-config
          target: /conf.d/routers.yml
          mode: 0400 #Read only
        - source: middlewares-config
          target: /conf.d/middlewares.yml
          mode: 0400 #Read only
        - source: tls-config
          target: /conf.d/tls.yml
          mode: 0400 #Read only
        - source: canary-config
          target: /conf.d/canary.yml
          mode: 0400 #Read only
        # Static config
        - source: traefik-config
          target: /traefik.yml
          mode: 0400 #Read only
      deploy:
        placement:
          preferences: *preferences
          constraints: *constraints
        update_config:
          # https://docs.docker.com/compose/compose-file/#update_config
          parallelism: 1
          delay: 30s
          monitor: 60s
          failure_action: rollback
          max_failure_ratio: 0
          order: stop-first
        rollback_config:
          parallelism: 1
          delay: 30s
          failure_action: continue
          monitor: 60s
          max_failure_ratio: 0
          order: stop-first
        labels:
          # Enable Traefik for this service, to make it available in the public network
          - traefik.enable=true
          - traefik.docker.lbswarm=true
          # Use the traefik-public network (declared below)
          - traefik.docker.network=traefik-public
          # Use the custom label "traefik.constraint-label=traefik-public"
          # This public Traefik will only use services with this label
          # That way you can add other internal Traefik instances per stack if needed
          # - traefik.constraint-label=traefik-edge
          # admin-auth middleware with HTTP Basic auth
          # Using the environment variables USERNAME and HASHED_PASSWORD
          traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
          # https-redirect middleware to redirect HTTP to HTTPS
          # It can be re-used by other stacks in other Docker Compose files
          ##- traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
          ##- traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
          # traefik-http set up only to use the middleware to redirect to https
          # Uses the environment variable DOMAIN
          - traefik.http.routers.traefik-edge-http.rule=Host(`traefik.swarm.dev`)
          - traefik.http.routers.traefik-edge-http.entrypoints=http
          - traefik.http.routers.traefik-edge-http.middlewares=admin-auth
          - traefik.http.routers.traefik-edge.service=api@internal
          ##- traefik.http.routers.traefik-edge-http.middlewares=https-redirect
          # traefik-https the actual router using HTTPS
          # Uses the environment variable DOMAIN
          ##- traefik.http.routers.traefik-edge-https.rule=Host(`traefik.swarm.dev`)
          ##- traefik.http.routers.traefik-edge-https.entrypoints=https
          ##- traefik.http.routers.traefik-edge-https.tls=true
          # Use the special Traefik service api@internal with the web UI/Dashboard
          ##- traefik.http.routers.traefik-edge-https.service=api@internal
          # Use the "le" (Let's Encrypt) resolver created below
          ##- traefik.http.routers.traefik-edge-https.tls.certresolver=le
          # Enable HTTP Basic auth, using the middleware created above
          ##- traefik.http.routers.traefik-edge-https.middlewares=admin-auth
          # Define the port inside of the Docker service to use
          - traefik.http.services.traefik-edge.loadbalancer.server.port=8080
          - traefik.http.services.traefik-edge.loadbalancer.sticky=true
          - traefik.http.services.traefik-edge.loadbalancer.sticky.cookie.name=traefik_xxx_cookie
          - traefik.http.services.traefik-edge.loadbalancer.sticky.cookie.httpOnly=true
          - traefik.http.services.traefik-edge.loadbalancer.sticky.cookie.secure=false
  ```
* Deploy the stack with:
  ```shell
  docker stack deploy -c traefik.yml traefik
  ```

* Check if the stack was deployed with:
  ```bash
  docker stack ps traefik
  ```

* It will output something like:
  ```
  ID             NAME                IMAGE          NODE              DESIRED STATE   CURRENT STATE          ERROR   PORTS
  w5o6fmmln8ni   traefik_traefik.1   traefik:v2.6.1 dog.example.com   Running         Running 1 minute ago
  ```

* You can check the Traefik logs with:
  ```bash
  docker service logs traefik_traefik
  ```

The next thing would be to deploy a stack (a complete web application, with backend, frontend, database, etc) using this Docker Swarm mode cluster.

It's actually very simple, as you can use Docker Compose for local development and then use the same files for deployment in the Docker Swarm mode cluster.

Example:

```docker
networks:
  traefik-public:
    external: true

services:
  whoami:
    image: traefik/whoami
    networks:
      - traefik-public
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.docker.lbswarm=true"
        - "traefik.docker.network=traefik-public"
        - "traefik.http.routers.whoami_whoami.rule=Host(`whoami.swarm.dev`)"
        - "traefik.http.routers.whoami_whoami.entrypoints=http"
        - "traefik.http.routers.whoami_whoami.service=whoami_whoami"
        - "traefik.http.services.whoami_whoami.loadbalancer.server.port=80"
```

Once you have made the decision to make the leap towards k8s you can continue to use traefik installing it using the [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)