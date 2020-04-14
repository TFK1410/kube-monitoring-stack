# kube-monitoring-stack
This a bunch of yaml bundles and steps which can be used to setup a prometheus monitoring cluster with oAuth for authentication.

The setup has been tested on OpenShift 3.11. It contains:
* Prometheus-operator
* Prometheus in HA
* Alertmanager in HA
* Grafana

The steps for deployment are as follows:
## Basic project setup
1. Create a monitoring project / namespace
1. Deploy prometheus-opeartor/bundle.yml in the newly created namespace

## Grafana password setup
1. Generate plain password file and keep it safe. This will be used to access prometheus using the internal user by grafana:
   ```bash
   # head /dev/urandom | tr -dc A-Za-z0-9 | head -c255 > plain_password
   ```
1. Generate htpasswd using this password:
   ```bash
   # htpasswd -cbs internal.htpasswd internal $(cat plain_password)
   Adding password for user internal
   ```
1. Create the secret in the project from the file:
   ```bash
   # oc create secret generic --from-file=auth=internal.htpasswd prometheus-k8s-htpasswd
   ```

## Deploy Prometheus along with basic servicemonitors
1. Create oauth session secret:
   ```bash
   # oc create secret generic prometheus-k8s-custom-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Replace the "CLUSTER", "NAMESPACE" and "EXTERNAL_PROMETHEUS_URL" placeholders in the prometheus/prometheus.yml and prometheus/servicemonitors.yml files with the domain name of the cluster, the monitoring namespace and the full external URL for Prometheus which will correspond to the route object
1. Deploy the prometheus/prometheus.yml bundle
1. Deploy the prometheus/servicemonitors.yml bundle. 

## Deploy Alertmanager
1. Create oauth session secret:
   ```bash
   # oc create secret generic alertmanager-custom-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Replace the "CLUSTER", "NAMESPACE" and "ALERTMANAGER_FULL_EXTERNAL_URL" placeholders in the alertmanager/alertmanager.yml file with the domain name of the cluster, the monitoring namespace and the full external URL for Alertmanager which will correspond to the route object
1. Deploy the alertmanager/alertmanager.yml

The alertmanager bundle contains the secret with an example configuration for the receivers. I suggest adjusting that to your own needs.

## Deploy Grafana secrets
1. Create oauth session secret:
   ```bash
   # oc create secret generic grafana-custom-proxy --from-literal=session_secret=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c43)
   ```
1. Edit the grafana/grafana-datasource.yml file and add the previously created plain password as the value for basicAuthPassword. Also add the monitoring namespace in the placeholder
1. Create a secret file out of the modified grafana-datasource.yml
   ```bash
   # oc create secret generic --from-file=prometheus.yml=grafana/grafana-datasource.yml grafana-custom-datasources
   ```

## Deploy Grafana
1. Create a secret out of the grafana.ini file:
   ```bash
   # oc create secret generic --from-file=grafana/grafana.ini grafana-custom-config
   ```
1. Deploy grafana.yml (log in and make sure that Prometheus is set as the default datasource)
