---
title: Modify default Instana Agent Docker image to include Zscaler cert
date: 2023-07-12 10:47:30 -0700
categories: [Docker,APM]
tags: [docker,zscaler,cert,jvm,instana agent]
---

## Custom Docker Image of Instana Agent with Zscaler Cert

### Reasoning

Whenever running Instana agent Docker containers behind Zscaler proxy, I got the following errors in Docker logs:

```bash
"instana agent bootstrapping failed"
"instana: ERROR: Cannot connect to the agent through default gateway. Scheduling retry"
```

In order to solve the problem, I created a custom image built on top of the Instana agent image. Basically, I add the Zscaler certificate to JVM trust store and run the agent with the customized image(tagged as "agent-zs" below).

### Instana Agent base image

- Pull the latest image

```bash
docker pull icr.io/instana/agent
```

- Run a container

```bash
docker run --name agent icr.io/instana/agent
```

- Login to the container using bash

```bash
docker exec -it agent bash
```

### JVM TrustStore

- Make a copy of the current trust store cacerts

```bash
cp /opt/instana/agent/jvm/jre/lib/security/cacerts ./trust-proxy.jks
```

- Add Zscaler cert to the trust store bundle

```bash
/opt/instana/agent/jvm/jre/bin/keytool -importcert -file zscaler.crt \
  -alias zscaler -keystore ./trust-proxy.jks -storepass changeit
```

- Set a new password to the trust store if needed

```bash
/opt/instana/agent/jvm/jre/bin/keytool.exe -storepasswd -new <Your New Password> \
  -keystore ./trust-proxy.jks -storepass changeit
```

- Replace the system trust store with the modified bundle

```bash
cp ./trust-proxy.jks /opt/instana/agent/jvm/jre/lib/security/cacerts
```

### Create a new Docker image with the modification above

- Save the container as an image

```bash
docker commit agent
```

- Tag the image

```bash
docker images -a
docker tag f4f835cc9de0 agent-zs
```

## References

- [How to Import Public Certificates into Java’s Truststore from a Browser](https://medium.com/expedia-group-tech/how-to-import-public-certificates-into-javas-truststore-from-a-browser-a35e49a806dc)
- [Instana Self-signed certificates](https://www.ibm.com/docs/en/instana-observability/251?topic=agents-self-signed-certificates)
- [How to Create a Docker Image From a Container](https://www.dataset.com/blog/create-docker-image/)
- Examples using docker-compose with Instana agent:
  - [Instana Node.js SDK Demo](https://github.com/instana/instana-nodejs-demos/tree/master/sdk): image URI is obsolete, see the next demo for current one.

    ```yaml
    version: '3'
    services:

        agent:
            image: instana/agent:latest
            pid: "host"
            privileged: true
            volumes:
            - /var/run/docker.sock:/var/run/docker.sock
            - /dev:/dev
            - /sys:/sys
            - /var/log:/var/log
            networks:
            demomesh:
                aliases:
                - instana-agent
            environment:
            - INSTANA_AGENT_ENDPOINT=${agent_endpoint:?No agent endpoint provided}
            - INSTANA_AGENT_ENDPOINT_PORT=${agent_endpoint_port:-443}
            - INSTANA_AGENT_KEY=${agent_key:?No agent key provided}
            - INSTANA_DOWNLOAD_KEY=${download_key:-}
            - INSTANA_AGENT_ZONE=${agent_zone:-sdk-demo}
            expose:
            - 42699

        receiver-app:
            build:
            context: ./receiver-app
            networks:
            demomesh:
                aliases:
                - receiver-app
            environment:
            - INSTANA_AGENT_HOST=agent
            - BIND_ADDRESS=0.0.0.0
            expose:
            - 3216
            ports:
            - 3216:3216
            depends_on:
            - agent

        sender-app:
            build:
            context: ./sender-app
            networks:
            demomesh:
                aliases:
                - sender-app
            environment:
            - DOWNSTREAM_HOST=receiver-app
            depends_on:
            - receiver-app

    networks:
        demomesh: {}
    ```

  - [Instana Envoy Tracing Demo](https://github.com/instana/envoy-tracing/tree/main)

    ```yaml
    version: '3'
    services:

        client-app:
            build:
            context: ./client-app
            networks:
            - envoymesh
            environment:
            - INSTANA_DEV=1
            - target_url=http://envoy-gateway:8000/envoy-demo

        envoy:
            build:
            context: ./envoy
            args:
                agent_key: ${agent_key}
                download_key: ${download_key}
            volumes:
            - ./envoy/envoy-gateway.yaml:/etc/envoy-gateway.yaml
            networks:
            envoymesh:
                aliases:
                - envoy-gateway
            environment:
            - INSTANA_DEV=1
            - INSTANA_AGENT_HOST=instana-agent
            - INSTANA_AGENT_PORT=42699
            ports:
            - "8000:8000"
            - "8001:8001"

        server-app:
            build:
            context: ./server-app
            args:
                agent_key: ${agent_key}
                download_key: ${download_key}
            networks:
            envoymesh:
                aliases:
                - server-app
            environment:
            - INSTANA_DEV=1
            - INSTANA_AGENT_HOST=instana-agent
            - INSTANA_AGENT_PORT=42699
            - SERVER_PORT=8080
            expose:
            - "8080"

        agent:
            image: icr.io/instana/agent
            pid: "host"
            privileged: true
            volumes:
            - /var/run:/var/run
            - /run:/run
            - /dev:/dev:ro
            - /sys:/sys:ro
            - /var/log:/var/log:ro
            networks:
            envoymesh:
                aliases:
                - instana-agent
            environment:
            - INSTANA_AGENT_ENDPOINT=${agent_endpoint:-ingress-red-saas.instana.io}
            - INSTANA_AGENT_ENDPOINT_PORT=${agent_endpoint_port:-443}
            - INSTANA_AGENT_KEY=${agent_key}
            - INSTANA_DOWNLOAD_KEY=${download_key}
            - INSTANA_AGENT_ZONE=${agent_zone:-envoy-tracing-demo}
            expose:
            - "42699"

    networks:
        envoymesh: {}
    ```

### [Install CA certificates on Linux systems](https://www.hs-schmalkalden.de/en/university/faculties/faculty-of-electrical-engineering/studium/use-of-it/install-ca-certificates-on-linux-systems)

The following is for Redhat, which Instana Agent Docker image is based on:

- Create the /etc/pki/ca-trust/source/anchors directory if not yet present,
- Copy the .crt files into the directory,
- Run "update-ca-trust".
