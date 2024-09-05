# K8S environment

## Create

```bash
# Validate the Vagrantfile

# Start and provision the environment
vagrant up --provider=libvirt

# Connect to machine via SSH
vagrant ssh [ cp | worker ]
```

## Cleanup

```bash
# Stop and delete the environment
vagrant destroy --force
```
