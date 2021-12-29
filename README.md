# Setting Up Prometheus and Adding Endpoints
Here we will setup a monitoring solution by installing **Prometheus, Alertmanager, and Grafana**, and ensure metrics from all three services are being fed into Prometheus. 
As we install Prometheus and Alertmanager, we'll also configure our environment to easily and effectively change our configurations without any added work.

# Set Up Prometheus
- Create a monitoring server for **Prometheus** and **Alert Manager**. 
- On the monitoring server, create a user for `Prometheus`:
  - sudo useradd --no-create-home --shell /bin/false prometheus
- Create the needed directories:
  - mkdir -p /etc/prometheus /var/lib/prometheus
- Set the ownership of the /var/lib/prometheus directory:
  - chown prometheus:prometheus /var/lib/prometheus
- Download Prometheus:
  - cd /tmp/
  - wget https://github.com/prometheus/prometheus/releases/download/v2.7.1/prometheus-2.7.1.linux-amd64.tar.gz 
- Extract the files:
  - tar -xvf prometheus-2.7.1.linux-amd64.tar.gz
- Move the configuration file and set the owner to the prometheus user:
  - cd prometheus-2.7.1.linux-amd64/
  - mv console* prometheus.yml /etc/prometheus
  - chown -R prometheus:prometheus /etc/prometheus
- Move the binaries and set the owner:
  - mv prometheus promtool /usr/local/bin/
  - chown prometheus:prometheus /usr/local/bin/prom*

- Change back to your main directory, and then create the service file `/etc/systemd/system/prometheus.service`:
  - sudo vim /etc/systemd/system/prometheus.service
  - Add the following to the file:
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

- Start Prometheus and make sure it automatically starts on boot:
  - sudo systemctl start prometheus
  - sudo systemctl enable prometheus
- Check Prometheus's status to ensure everything is working correctly:
  - sudo systemctl status prometheus

# Set Up Alertmanager
- On the same monitoring server, create the alertmanager system user:
  - sudo useradd --no-create-home --shell /bin/false alertmanager
- Create the needed directories:
  - sudo mkdir /etc/alertmanager
- Download Alertmanager:
  - cd /tmp/
  - wget "https://github.com/prometheus/alertmanager/releases/download/v0.16.1/alertmanager-0.16.1.linux-amd64.tar.gz"
- Extract the files:
  - tar -xvf alertmanager-0.16.1.linux-amd64.tar.gz
  - cd alertmanager-0.16.1.linux-amd64/
- Move the binaries:
  - sudo mv alertmanager amtool /usr/local/bin/
- Set the ownership of the binaries:
  - sudo chown -R alertmanager:alertmanager /usr/local/bin/alertmanager /usr/local/bin/amtool
- Move the configuration file into the /etc/alertmanager directory:
  - sudo mv alertmanager.yml /etc/alertmanager/
- Set the ownership of the /etc/alertmanager directory:
  - sudo chown -R alertmanager:alertmanager /etc/alertmanager/
- Create the alertmanager.service file for systemd:
  - sudo vim /etc/systemd/system/alertmanager.service
  - Add the following to the file, save and exit:

```
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
WorkingDirectory=/etc/alertmanager/
ExecStart=/usr/local/bin/alertmanager \
    --config.file=/etc/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target
```

- Stop Prometheus and update the Prometheus configuration file to use Alertmanager:
  - sudo systemctl stop prometheus
  - cd
  - sudo vim /etc/prometheus/prometheus.yml
    - Un-comment alertmanager, and change alertmanager to localhost (so it will look like the following):

```
alerting:
  alertmanagers:
    - static_configs:
      - targets:
        - localhost:9093
 ```

- Reload systemd, and then start the prometheus and alertmanager services:
  - sudo systemctl daemon-reload
  - sudo systemctl start prometheus
  - sudo systemctl start alertmanager
  
- Make sure alertmanager starts on boot:
  - sudo systemctl enable alertmanager

# Set Up Grafana
Create a new server for `Grafana`, log in to the grafana server via SSH:

- Install the prerequisite package:
  - sudo apt-get install libfontconfig
- Download and install Grafana using the .deb package provided on the Grafana download page:
  - cd /tmp/
  - wget https://dl.grafana.com/oss/release/grafana_5.4.3_amd64.deb
  - sudo dpkg -i grafana_5.4.3_amd64.deb
- Start and enable Grafana service:
  - sudo systemctl start grafana-server
  - sudo systemctl enable grafana-server
- Access Grafana's web UI by navigating to `<GRAFANA_IP_ADDRESS>:3000` in a new browser tab.
- Log in with the username `admin` and the password `admin`. Reset the password when prompted.

- Add a Data Source
  - Click `Add data source` on the home page. Select `Prometheus`.
  - In the `HTTP` section, set the URL to `http://<MONITORING_SERVER_IP_ADDRESS>:9090`
  - Click Save & Test.

- Add Alert Manager and Grafana Endpoints in the Prometheus configuration file:
  `sudo vim /etc/prometheus/prometheus.yml`
  - At the end of the file, at the bottom of the `scrape_configs` section, add the Alertmanager endpoint (make sure it aligns with the - job_name: 'prometheus' line):
    
```
- job_name: 'alertmanager'
  static_configs:
  - targets: ['localhost:9093']

- job_name: 'grafana'
  static_configs:
  - targets: ['<GRAFANA_IP_ADDRESS>:3000']
```

- Restart Prometheus:
  - sudo systemctl restart prometheus
  - sudo systemctl status prometheus
- Navigate to the Prometheus web UI in a new browser tab: `http://<MONITORING_IP_ADDRESS>:9090`
  - Click `Status` > `Targets`.

Ensure all three endpoints are listed on the Targets page.
