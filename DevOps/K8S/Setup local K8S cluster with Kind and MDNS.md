# Setup local K8S cluster with Kind and MDNS

## Requirements
1. [Install Docker](https://docs.docker.com/engine/install/ubuntu/)
2. [install Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
3. [Instal K8S tools](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
4. `sudo apt install avahi-utils inotify-tools`
5. `pip install mdns-publisher`

## Setup cluster
### Add MDNS domains
**Add `/etc/mdns.allow`:**
```
.local.
.local
```

**Edit `vi /etc/nsswitch.conf`:**

Set `hosts: files mdns4_minimal mdns4 dns [NOTFOUND=return] `

**Create domain config file `local_domains`:**
```
service1._your_machine_hostname_.local
...
service2._your_machine_hostname_.local
```

**Create script to publish domains `publish-domains.sh`:**
```
#!/bin/bash

# The command to run
CMD="mdns-publish-cname -f"

# The configuration file to monitor
CONFIG_FILE="/path/to/local_domains"

# Function to execute the command with the config file
run_command() {
    DOMAINS=$(awk '{printf "%s ", $0}' $CONFIG_FILE)

    # Kill the previous instance of the command if it's still running
    if [[ ! -z "$CMD_PID" ]]; then
        kill $CMD_PID 2>/dev/null
    fi

    # Run the command in the background and save its PID
    $CMD $DOMAINS &
    CMD_PID=$!
    echo "Command executed at $(date) with PID $CMD_PID"
}

# Make sure to kill the command when the script exits
trap 'kill $CMD_PID 2>/dev/null' EXIT

# Initial run
run_command

# Monitor the config file for changes and rerun the command when changes are detected
while inotifywait -e close_write,moved_to,create "$CONFIG_FILE"; do
    echo "Config file changed, rerunning command..."
    run_command
done
```

**Create service for our script `mdns-publisher.service`:**
```
[Unit]
Description=Avahi/mDNS CNAME publisher
After=network.target avahi-daemon.service

[Service]
User=_your_name_
Type=simple
WorkingDirectory=/home/_your_name_
ExecStart=/home/_your_name_/path/to/publish-domains.sh
Restart=no
PrivateTmp=true
PrivateDevices=true

[Install]
WantedBy=multi-user.target
```

**Move, enable and run the service:**
```
chmod 755 mdns-publisher.service
sudo chown root:root mdns-publisher.service
sudo mv mdns-publisher.service /etc/systemd/system
sudo systemctl enable mdns-publisher
sudo systemctl start mdns-publisher.service
```

**Check that domains working**

`ping service1._your_machine_hostname_.local`

### Setup Kind
**Add cluster config `kind.config.yml`:**

Docs here: https://kind.sigs.k8s.io/docs/user/configuration/
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster-name
networking:
  ipFamily: ipv4
  apiServerAddress: 0.0.0.0
  apiServerPort: 6660

nodes:
- role: control-plane
  image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
- role: worker
  image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
- role: worker
  image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
- role: worker
  image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
- role: worker
  image: kindest/node:v1.29.2@sha256:51a1434a5397193442f0be2a297b488b6c919ce8a3931be0ce822606ea5ca245
```

**Create cluster:**

`kind create cluster --config=kind.config.yml`

Use `kubectl cluster-info ...` from output to check

### Setup Lens

Run to get kubeconfig `kind get kubeconfig --name _your_cluster_name`
