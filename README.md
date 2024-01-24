# sensorberg-task

To run the playbook, please install ansible in rocky linux 9 OS and run the below commands.

```
git clone https://github.com/jpadmin/sensorberg-task
cd sensorberg-task
ansible-playbook install_mosquitto.yml
```

## Motivation

The ansible playbook is organised in roles that has dependency in a top down approach as a stack such that it does the deployment in the order build -> configure -> runtime. Instead of designing roles per component, this design helps in rolling specific change in required phase, like only build tasks in build phase, configure tasks in configure phase and runtime tasks in runtime phase.

## Roles

### Build
`base` - Performs the OS and build utility installations

`mosquitto_build` - Performs the mosquitto server installation from epel-release rpm repository.

`mosquitto_exporter_build` - Performs the mosquitto_exporter build to generate a server compatible binary from a thirdparty git repository - https://github.com/sapcc/mosquitto-exporter

### Configure (Depends on Build)
`mosquitto_configure` - Configures the mosquitto server with tls certificates and authentication and creates exporter service file manually. Also opens the required ports(22, 8883 and 9234) to the firewall. Mosquitto port 1883(non SSL) only accepts authenticated localhost connections and is primarily for prometheus exporter. Mosquitto port 8883(SSL) is publicly available but with ca root certificate for SSL verification(One way SSL) and valid authentication.

### Runtime (Depends on Configure)
`mosquitto_runtime` - Reloads the firewall and start the mosquito and exporter servers. Mosquitto server is available on port 8883 and Prometheus exporter is available on port 9234.

## TLS and Client Authentication

![image](https://drive.google.com/uc?export=view&id=16Vft7NwIgP4PutS6hzRk1ANFH44BzwdL)

### Certificate Generation
For TLS configuration, a root CA is created and the server certificate and the server key is generated using the root CA.
Ref : https://mosquitto.org/man/mosquitto-tls-7.html

For this playbook, a self signed certificate is generated for this purpose with the sub domain : `mosquittodemo.sensorberg.com`.

### How to test the SSL connectivity for publish and subscribe

 As our SSL certificate is validated using the domain `mosquittodemo.sensorberg.com`, a host validation is accompanied with the mosquitto_pub or mosquitto_sub client calls. This would lead to SSL failure if we pass the refer the mosquitto broker by its IP address or otherwise we need to supply the `--insecure` flag with our connections. This can be resolved by adding a DNS entry for `mosquittodemo.sensorberg.com` to our server's IP address. Alternatively for test purposes, we can add temporary host file addition to our clients. If the client is linux machine, this can be added in the `/etc/hosts` file of the client. Say if 10.1.12.11 is our server's IP, run:
```
printf "\n10.1.12.11 mosquittodemo.sensorberg.com" >> /etc/hosts
```

Then to subscribe to the broker using SSL with client auth, run:
```
mosquitto_sub -h mosquittodemo.sensorberg.com -t <topic> -p 8883 --cafile roles/mosquitto_configure/files/certs/ca.crt -u "<username>" -P "<password>"
```
OR
```
mosquitto_sub -h 10.1.12.11 --insecure -t <topic> -p 8883 --cafile roles/mosquitto_configure/files/certs/ca.crt -u "<username>" -P "<password>"
```

And to publish a message, run:
```
mosquitto_pub -h mosquittodemo.sensorberg.com -t <topic> -p 8883 --cafile roles/mosquitto_configure/files/certs/ca.crt -u "<username>" -P "<password>" -m "<message>"
```
OR
```
mosquitto_pub -h 10.1.12.11 --insecure -t <topic> -p 8883 --cafile roles/mosquitto_configure/files/certs/ca.crt -u "<username>" -P "<password>" -m "<message>"
```

Here `<username>` refers to the client auth username, `<password>` refers to the client auth password, `<topic>` refers to the name of the topic and `<message>` refers to the message to be published.

## Prometheus Exporter

We use prometheus mqtt exporter to fetch the mosquitto related metrics to pass to prometheus. There are different exporters available and this demo uses the exporter from sapcc : https://github.com/sapcc/mosquitto-exporter

### Build

Follow the below steps to build the exporter specific to our server
```
git clone https://github.com/sapcc/mosquitto-exporter.git
cd mosquitto_exporter
make build
```

Run the below command to start the exporter server. This demo expects the exporter to share the same server for the mosquitto server and exporter server. Make sure mosquitto server is already running.
```
cd mosquitto_exporter/bin
./mosquitto_exporter --endpoint tcp://127.0.0.1:1883 -u <username> -p <password>
```

Verify the metrics by calling
```
curl -I localhost:9234/metrics
HTTP/1.1 200 OK
Content-Type: text/plain; version=0.0.4; charset=utf-8
Date: Wed, 24 Jan 2024 19:32:33 GMT
```

## Prometheus integration and PromQL

### Assumptions for the PromQL

For the PromQL to work properly, we assume following prometheus scrapping configuration.

```
global:
  scrape_interval: 15s

scrape_configs:
- job_name: node
  static_configs:
  - targets: ['localhost:9234']
```

Please note that instead of localhost you need to specify the private IP / publish IP address of the exporter server depending on where are the prometheus hosted.

### PromQL

```
rate(broker_messages_received{ job = "node" }[1h])
```
![image](https://drive.google.com/uc?export=view&id=1Z9sj5hj__GYXpGy18FRvrSGDJlecCKVb)

## Scope of Improvement.

### Improve resilience and bring on HA
- Features - keep-alive, client takeover or message queueing for failing/offline subscribers
- Add clean session and persistent storage to the Mosquitto server
- Associating bridge connections between Mosquitto servers as a cluster

### Technologies for HA
- Develop with containerisation approach
- A small scale app cluster can be managed using a docker swarm or public cloud solutions like Containerisation or AWS ECS or fargates, but upon hitting a large volume these are best fit with Kubernetes architecture(constructs like PDB, HPA, VPA, Replication controller and failure mechanisms thrive for HA).