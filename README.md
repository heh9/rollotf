<h1 align="center">Welcome to rollotf 👋</h1>

> Terraform helper that allows you to apply rolling updates on resources with counts.

## 💡 How it works

First step is to partition the instances in cycles

```python
partition, resource = 2, [ 'node-1', 'node-2', 'node-3', 'node-4', 'node-5' ]
```

The update of this resource was split into 3 cycles where `len(cycle) <= partition`, meaning that no more than `len(cycle)` nodes can be down at any given time

```python
cycles = [ [ 'node-1', 'node-2' ], [ 'node-3', 'node-4' ], [ 'node-5' ] ]
```

In each cycle a `terraform apply` or `terraform destroy+apply` is ran, targeting only the instances from that cycle

```bash
terraform apply -target resource.name['node-1'] -target resource.name['node-2']

bash <<EOF
exit_code=$(healhcheck_command)
until [ $exit_code -eq 0 ]; do
    exit_code=$(healhcheck_command)
done
EOF

continue
```

A cycle waits for each instance to pass its health checks before proceeding

## ✨ Demo

Upgrading a Vault cluster:

*Placeholder for video*

Example of `config.yaml` with good meta data:

```yaml
# Override default command terraform or add flags to it
command: terraform -lock=true -no-color
# Name of the terraform resource to be updated
name: vsphere_virtual_machine.vault_server
# Maximum no. of instances to be updated in one cycle
partition: 1
# Force destroy of the instance, use where providers don't detect changes properly
recreate: yes
# Healtcheck condition that must be satisfied in order to proceed to next cycle
healthcheck:
  # Command used to check instance health, available environment variables are:
  # $INDEX $COUNT $INSTANCE_IP $INSTANCE_NAME
  exec: |
    #!/bin/bash

    http_code=$(curl -sw '%{http_code}' http://${INSTANCE_IP}:8200/v1/sys/health -o /dev/null)
    if [ ${http_code} -eq 200 ]; then
        exit 0
    fi

    exit 1
  # Or provide a script file instead of the exec, it must be executable,
  # have a shebang and be present in the root folder
  script: health.py
  # Intial delay after finishing an apply and before starting the checks
  delay: 5m
  # How much to wait between healthchecks
  period: 15s
```

## 🚀 Usage

Make sure you have [terraform](https://www.terraform.io/downloads.html) installed

Just run the following command at the root of your project:

```bash
rollotf apply -config vault.yaml
```

Or provide the config from stdin:

```bash
cat <<EOF | rollotf apply -
name: vsphere_virtual_machine.vault_server
partition: 1
recreate: yes
healthcheck:
  script: health.py
  delay: 1m
  period: 15s
EOF
```

Generate default config:

```bash
rollotf config > default.yaml
```

## 📝 License

Copyright © 2020 [Vladimir Iacob](https://github.com/heh9).<br />
This project is [MIT](https://github.com/heh9/rollotf/blob/master/LICENSE) licensed.
