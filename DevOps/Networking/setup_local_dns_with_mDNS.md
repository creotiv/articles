# Setup local DNS with mDNS

### Requirements
```bash
# some tools that we need
sudo apt install avahi-utils inotify-tools libnss-resolve
# script that will be notifying about dns changes on host
pip install mdns-publisher
```

Create **/etc/mdns.allow** with content:
```bash
.local.
.local
```

Edit **/etc/nsswitch.conf**, find `hosts:` and replace on

```bash
hosts: files mdns4_minimal mdns4 dns [NOTFOUND=return]
```

Create local config **local_domains.conf**
```bash
service1._your_machine_hostname_.local
...
service2._your_machine_hostname_.local
```

Set hostname for you machine

```bash
sudo hostnamectl set-hostname new-hostname
```

Set your network interface to support mDNS
```bash
sudo resolvectl mdns _your_network_intface yes
```

### Adding service that will look for changes in our config file

This script will rerun mdns_publisher on config changes

**publish-domains.sh**
```bash
#!/bin/bash

# The command to run
CMD="mdns-publish-cname -f"

# The configuration file to monitor
CONFIG_FILE=">>>PATH_TO_YOUR_CONFIG<<<"

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

Now we need to make service to run this script

**mdns-publisher.service**
```bash
[Unit]
Description=Avahi/mDNS CNAME publisher
After=network.target avahi-daemon.service

[Service]
User=_your_name_
Type=simple
WorkingDirectory=>>>/working/directory/where_your_script_will_run_from<<<
ExecStart=>>>/path/to/publish-domains.sh<<
Restart=yes
PrivateTmp=true
PrivateDevices=true

[Install]
WantedBy=multi-user.target
```

Now let's enable and run the service

```bash
chmod 755 mdns-publisher.service
sudo chown root:root mdns-publisher.service
sudo mv mdns-publisher.service /etc/systemd/system
sudo systemctl enable mdns-publisher
sudo systemctl start mdns-publisher.service
```

### Checking that all works
```bash
ping service1._your_machine_hostname_.local
nslookup service1._your_machine_hostname_.local
```

Let's try Nginx

**service.conf**
```bash
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
        worker_connections 768;
}

http {
  server {
      listen 80 default_server;
      server_name _;

      location / {
          return 404;
      }
  }

  server {
    listen 80;      # for IPv4
    listen [::]:80; # for IPv6

    server_name service1._your_machine_hostname_.local;
    access_log /var/log/nginx/service1.access.log;

    location / {
      proxy_pass http://localhost:8000;

      proxy_set_header Host            $host;
      proxy_set_header X-Real-IP       $remote_addr;
      proxy_set_header X-Forwarded-For $remote_addr;
      proxy_http_version 1.1;

      port_in_redirect on;
    }
  }
}
```

Run simple python http server
```bash
python3 -m http.server
```

And run Nginx
```bash
sudo systemctl stop nginx
nginx -c service.conf
```
