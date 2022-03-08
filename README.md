# SRE Project - Observing Cloud Resources

The purpose of this readme file is to document all steps and caveats to finish the project1 for this SRE nanodegree course.

## Project Description

This project showcase how to set up monitoring from scratch, adding IT assets and configure important alert to notify proper teams. Additionaly, this project reveals how to monitor not only host matrix but also an application, and send notification of the application endpoint becomes unavailable. Prior to motitoring setup, this project also include setting up dependencies and install Premitheus and Grafana using EKS, create aws cloud infrastructure using infrastructure as code (terraform).

## Main Steps

The main steps of this project are:

1. Provisioning the cloud resources
2. Installing node_exporter on the EC2 instance
3. Testing connectivity to the flask app
4. Installing the monitoring software in the EKS cluster
5. Creating dashboards for host metrics and endpoint health
6. Creating alerting rules and notification channels

## Enviroment and Dependencies

### Dependencies

- [Python](https://www.python.org/downloads/)
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- [VS Code](https://code.visualstudio.com/Download)
- [Postman](https://www.postman.com/downloads/)
- [Lens](https://k8slens.dev/)
- Clone the git repo to local enviroment

## Provision the Cloud Resources

1. Create IAM user and configure AWS CLI v2

- Create IAM user account in AWS
- Configure AWS CLI v2 to use named profile "udacity-admin" and set default region to us-east-1

2. Copy AMI to my account

- restore image

```shell
aws ec2 create-restore-image-task --object-key ami-08dff635fabae32e7.bin --bucket udacity-srend --name udacity-sre-project1-ami-image
```

The `udacity-sre-project1-ami-image` is s3 bucket name to match the AMI name.

- Take note of that AMI ID the script just output. Copy the AMI to `us-east-2`:

```shell
aws ec2 copy-image --source-image-id <your-ami-id-from-above> --source-region us-east-1 --region us-east-2 --name udacity-sre-project1-ami-image
```

3. Create a private key pair for your EC2 instance called `udacity` in the `us-east-2` region

4. Update configuration files in terraform folder:

- In `_data.tf` file: Replace the `owners` field with Amazon owner ID assigned on the AMI. Replace the `filter` values of the `name` to the AMI name ("udacity-sre-project1-ami-image").
- In `ec2.tf` file: Replace the `aws_ami` to the AMI ID from us-east-2.
- In `_config.tf` file: Replace `terraform.backend.s3.bucket` to bucket name used to store terraform backend state file, and make sure the region is set to "us-east-1". Also check the `provider.aws.region` is set to "us-east-2".

In this set up, two s3 buckets are created in us-east-1, one bucket is used to store AMI and the other one is used to store terraform backend state file.

5. Provision cloud infrastructure using Terraform.
   After updating configurations, run terraform `init`, `plan` and `apply` to provision each of the resource in AWS. EKS cluster and two EC2 instances will be ceated in us-east-2 region.

## Installing node_exporter on the EC2 instance

node_exporter is used to gather and export host matrix from EC2 instance to premethus.

1. SSH into the EC2 instance with username `ubuntu` and the udacity key created in a previous step.
   `ssh ubuntu@ec2-public-ip -i "path-to-udacity-pem"`

2. Install the node exporter on the EC2 instance.

```
sudo useradd --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.2.2.linux-amd64.tar.gz
sudo cp node_exporter-1.2.2.linux-amd64/node_exporter /usr/local/bin
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
sudo nano /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter

sudo ufw allow 9100/tcp
sudo systemctl restart ufw
```

## Testing connectivity to the flask app

1. Import the colletion and enviroment files to postman.
2. Change the following variables: `public-ip, username, email` then open the collection runner, choose the collection and environment, then Run the project. Check successful responses for each API endpoints.

The `/init` and `authorize/user` API call will initialize the database and register a user accordingly.

An example of success return to initialize a database:

```
{
    "dataset": {
        "created": "Mon, 07 Mar 2022 23:13:02 GMT",
        "description": "initialize the DB",
        "id": 6,
        "location": "home",
        "name": "init db"
    },
    "status": {
        "message": "101: Created.",
        "records": 1,
        "success": true
    }
}
```

## Installing the monitoring software in the EKS cluster

1. Update the `kube config file` by running `aws eks --region us-east-2 update-kubeconfig --name udacity-cluster`
2. create name space called monitoring: `kubectl create namespace monitoring`
3. Create the `prometheus-additional.yaml` file and set the `targets` accordingly for both Prometheus and Blackbox. Set `targets` to the public IP address of the EC2 instance ("e.g. http://18.216.124.163")
4. Modify the `values.yaml` (near line 2310) so that it matches:

```
additionalScrapeConfigsSecret:
      enabled: true
      name: additional-scrape-configs
      key: prometheus-additional.yaml
```

5. Create the kubernetes secret which references the above yaml file by `kubectl create secret generic additional-scrape-configs --from-file=prometheus-additional.yaml --namespace monitoring`

6. Add premetheus to helm repo
   `helm repo update`
   `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`

   Only need to run above command once, if premetheus is already in helm repo, it will skip.

7. Install the monitoring stack in Kubernetes using helm.
   `helm install prometheus prometheus-community/kube-prometheus-stack -f values.yaml --namespace monitoring`

   Make sure here the `-f` followed by `values.yaml` file path not the `prometheus-additional.yaml` file path.

## Creating dashboards for host metrics and endpoint health

### Host matrix monitroing

1. Open Lens, click monitoring namespace and open Grafana from network->services->premethos-grafana->connection->ports (Then login to Grafana using the credentials`user: admin & password: prom-operator`.)
2. Create dashboards for the CPU for the EC2 instance using `instance:node_cpu:rate:sum` query.
3. Make a dashboard for Available Memory in bytes. Use `node_memory_MemAvailable_bytes` in the prometheus query.
4. Make a for Disk I/O. Use `node_disk_io_now` in the prometheus query.
5. Make a for Network Received in bytes. Use `instance:node_network_receive_bytes:rate:sum` in the prometheus query.

### Blackbox Exporter

1. Copy the token from postman API calls, then in `blackbox-values.yaml` add the following:

```
config:
  modules:
    http_2xx:
      prober: http
      timeout: 5s
      http:
        valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
        follow_redirects: true
        preferred_ip_protocol: "ip4"
        valid_status_codes:
          - 200
        bearer_token: <Token from API call>
```

2. Install blackbox in the Kubernetes cluster by running command: `helm install prometheus-blackbox-exporter prometheus-community/prometheus-blackbox-exporter -f blackbox-values.yaml --namespace monitoring`

Make sure `-f` followed by the path of the `blackbox-values.yaml`

3. In Grafana, import dashboard 7587.

## Creating alerting rules and notification channels

1. Set up a notification channel to Slack using a webhook.
2. Create a dashboard for an API health check to check if the flask endpoint is online. Use `probe_http_status_code` for the prometheus query.
3. Configure alerts for:
   - One of the host metrics from above (CPU/Memory/Disk/Network)
   - Showing if the the flask endpoint is offline.
4. Cause the host metrics alerts to trigger.
5. Cause the flask endpoint to go offline. This can be achived by reboot ec2 instance.

## Submissions

1. A zip file containing screenshots from Grafana which include:
   - The dashboard for EC2 CPU utilization.
   - The dashboard for EC2 Memory utilization.
   - The dashboard for EC2 Disk I/O.
   - The dashboard for EC2 Network utilization.
   - The imported dashboard for Blackbox Exporter.
   - The dashboard showing that an alert triggered (could be one of CPU/memory/disk/network utilization).
   - The message from the alert in slack.
   - The alert showing that the flask app is offline.
   - The alert showing that the flask app is back online.
   - The list of alerting rules.
2. The screenshot of the node_exporter service running on the EC2 instance `sudo systemctl status node_exporter`
