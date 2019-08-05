# Helm Chart for Frappe/ERPNext based applications
This project provides a helm chart for deploying a Frappe/ERPnext based application onto Kubernetes using 'helm' packaging tool.

## Pre-requisites
1. Working installation of Helm components - `Tiller` installed and configured (for `RBAC`) on target Kubernetes cluster and 'helm' client machine. For more details please refer to [Helm documentation](https://helm.sh/docs/using_helm/#installing-helm).
2. Storage provisioned using either `azure-file` or `azure-disk` (for AKS deployments). This involves creation of a storage account and a `file-share`, if using `azure-file` based persistent storage. For `azure-disk` based persistence, it setting up of [dynamic provisioning](https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv) is recommended. Create a blank `apps.txt` and `currentsite.txt` in the shared drive created for the application. Note that, without this, the application __will fail to deploy__.
3. Docker image created and available (i.e. `docker push`ed) in a shared repository.

## Installation Steps
1. `git clone` this chart (until a helm chart repository is setup) to any location locally.
2. Use one of the sample [values.yaml](../values.yaml) or create your own with same structure as the sample file. Update the values in the file as appropriate for your application.
3. Change to the directory where this chart has been cloned and deploy the application using `helm` command as follows -
<pre>helm install --name <release name> -f <path to values.yaml> .</pre>
For example -
<pre>helm install --name cas -f /workspace/matrix/values-matrix.yaml .</pre>

## Key Terms
* Release Name - Name provided in above command after `--name` parameter.

## What the chart does
The helm chart creates following kubernetes resources for the application -
* RBAC Configuration - Chart creates a binding for role `cluster-admin` to the `default` service account in target namespace for the deployment.
* Configuration Files - Configuration required for redis components and frappe components are created with following naming convention -

    * Server configurations - <code>_&lt;release-name&gt;_-server-config</code>: Contains configuration for 
        
        * Kafka
        * DB Host
        * DB Name to be used for the site to be setup in frappe
        * Gunicorn Connections per Worker
        * Number of gunicorn workers to start
    * Fluentd Configuration - <code>_&lt;release-name&gt;_-fluentd-config</code>: Contains configuration for fluentd _sidecar_ container that polls the log files generated in logs folder of the bench installation and publishes them to an elasticsearch host. Destination Elastic search host is configured through `etchosts` group in `values.yaml`.
    * Redis Configuration - <code>_&lt;release-name&gt;_-redis-config</code>: Contains configuration files for redis instances for cache, queue and socketio, used by the frappe installation. Persistence can be enabled for each of the redis servers, if required. (_Not tested yet_).

* Storage Components have following naming convention -

    * Storage class - <code>_&lt;release-name&gt;_-sites-sc</code>
    * Persistent Volume - <code>_&lt;release-name&gt;_-sites-pv</code>
    * Persistent Volume Claim - <code>_&lt;release-name&gt;_-sites-pvc</code>
    Storage capacity requested is picked up from the `values.yaml` file provided from the key `persistence.capacityInGi`. Other parameters that are specific to the Azure persistance provisioned, are also picked up from `persistence` group of values in `values.yaml`.
    * Secrets are created with name  <code>_&lt;release-name&gt;_-secrets</code> - This contains the authentication information required to connect to Azure's File shares created using storage account.
    
* Deployment of Redis and Frappe components

    * StatefulSet that runs 2 containers in a pod - one with Frappe and related applications and another sidecar that runs fluentd log collector. Number of replicas for the stateful set can be defined in the `values.yaml`.
    * Stateful set for Redis cache instance with configuration provided from configuration created above. This will be named as <code>_&lt;release-name&gt;_-redis-cache</code>
    * Stateful set for Redis queue instance with configuration provided from configuration created above. This will be named as <code>_&lt;release-name&gt;_-redis-queue</code>
    * Stateful set for Redis socketio instance with configuration provided from configuration created above. This will be named as <code>_&lt;release-name&gt;_-redis-socketio</code> 

* Services expose the applications deployed in the cluster. Following services are created -

    * Service with same name as the stateful set for frappe application is created that exposes the NGinx port (8000) in the frappe container
    * External Database is made available to the frappe application as an external name service which is used by the frappe application. The actual IP Address of the database is looked up using the entries in `etchosts`. The `branch` name provided in the `values.yaml` is used as prefix to `-db.ntex.com` and used as hostname for the database. This entry must be present in the `etchosts` group for frappe application to connect to the database. Application will then create/re-create database with name `site.dbname` value. Note that if the database exists while site is being initialized, it will be deleted and recreated - any data from the old database will be lost.
    * Redis cache setup is made available to the frappe application through a service named `er-frappe-redis-cache` and connects back to the deployment of redis cache created above. This service is exposed on port 13000.
    * Redis queue setup is made available to the frappe application through a service named `er-frappe-redis-queue` and connects back to the deployment of redis queue created above. This service is exposed on port 11000.
    * Redis socketio  setup is made available to the frappe application through a service named `er-frappe-redis-socketio` and connects back to the deployment of redis socketio created above. This service is exposed on port 12000.

## Configuration
Besides the above resources created, chart can be customized through the configuration parameters listed below.

|Parameter|Description|Default|
|-------------------------------------------|-----------------------------------------------------|-------------------------------------------------------------------|
|`replicaCount`| Number of replicas to be created for the frappe application | `1`|
|`targetNamespace`| Target namespace to which the application and related components should be deployed | `default`|
| TODO | TODO | TODO |
