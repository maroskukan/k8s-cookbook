# K8S environment

The `docker-compose.yml` contains definition for three Ubuntu 22.04 containers - one for control-plane and two to be used as worker nodes.

## Create

```bash
# Validate the docker-compose file

# Start and provision the environment
docker compose up -d
# Output
[+] Running 4/4
 ✔ Network k8snet     Created                                                                                     0.3s
 ✔ Container worker1  Started                                                                                     0.3s
 ✔ Container cp       Started                                                                                     0.3s
 ✔ Container worker2  Started                                                                                     0.3s

# Connect to container(s) via interactive terminal
docker exec -it [cp | worker1 | worker2 ] /bin/bash
```

## Cleanup

```bash
# Stop and delete the environment
docker compose down
```
