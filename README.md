# Installing Kong on IBM Cloud

## Step 1 - Provision Kubernetes Cluster

- Click the **Catalog** button on the top
- Select **Service** from the **Catalog**
- Search for **Kubernetes Service** and click on it

![kong_doc_html_46d1c04e26ba5eea](https://user-images.githubusercontent.com/5286796/106394187-4b013200-6421-11eb-92b7-6c825737765c.png)

- You are now at the Kubernetes deployment page. You need to specify some details about the cluster
- Choose a plan **standard** or **free**. The free plan only has one worker node and no subnet. To provision a standard cluster, you will need to upgrade your account to Pay-As-You-Go

To upgrade to a Pay-As-You-Go account, complete the following steps:

- In the console, go to Manage > Account.
- Select Account settings; and click Add credit card.
- Enter your payment information, click Next, and submit your information
- Choose **classic** or **VPC** , read the docs and choose the most suitable type for yourself

![kong_doc_html_4d3a968071544952](https://user-images.githubusercontent.com/5286796/106394203-62d8b600-6421-11eb-89d2-98dc1c439942.png)

- Now choose your location settings,
- Choose **Geography** (continent)

![kong_doc_html_72496e6b0b2c820d](https://user-images.githubusercontent.com/5286796/106394202-610ef280-6421-11eb-978f-04ac9b590083.png)

- Choose Single or Multizone. 
> In single zone, your data is only kept on the datacenter while on the other hand with Multizone, it is distributed to multiple zones, thus safer in an unforeseen zone failure
>
> If you wish to use Multizone, please set up your account with VRF

- If at your current location selection, there is no available Virtual LAN, a new VLAN will be created for you
- Choose a Worker node setup or use the preselected one. Set Worker node amount per zone
- Choose **Master Service Endpoint**. 

> In VRF-enabled accounts, you can choose private-only to make your master accessible on the private network or via VPN tunnel. Choose public-only to make your master publicly accessible. When you have a VRF-enabled account, your cluster is set up by default to use both private and public endpoints.
- Give desired **tags** to your cluster, click **create**
- Wait for your cluster to be provisioned
- Your cluster is ready for usage

## Step 2 Deploy IBM Cloud Block Storage plug-in

The Block Storage plug-in is a persistent, high-performance iSCSI storage that you can add to your apps by using Kubernetes Persistent Volumes (PVs).

- Click the **Catalog** button on the top
- Select **Software** from the catalog
- Search for **IBM Cloud Block Storage plug-in** and click on it
- On the application page, click in the dot next to the cluster you wish to use
- Click on Enter or Select Namespace and choose the default Namespace or use a custom one (if you get error please wait 30 minutes for the cluster to finalize)
- Give a **name** to this workspace
- Click **install** and wait for the deployment


## Step 3: Deploying Kong

To deploy Kong onto your Kubernetes cluster with Helm, use:

```sh
$ helm repo add kong https://charts.konghq.com
$ helm repo update

# Helm 2
$ helm install kong/kong

# Helm 3
$ helm install kong/kong --generate-name --set ingressController.installCRDs=false
```

### Production configuration

This will have a `value-production.yaml` file with some parameters oriented to production configuration in comparison to the regular values.yaml can be find. You can use this file instead of the default file.

### Enable exposing Prometheus metrics:

```yaml
- metrics.enabled: false 
+ metrics.enabled: true 
```
### Enable Pod Disruption Budget:

```yaml
- pdb.enabled: false
+ pdb.enabled: true
```
### Increase number of replicas to 4:

```yaml
- replicaCount: 2
+ replicaCount: 4
```

### Database backend

Deploy the PostgreSQL sub-chart (default)

```sh
helm install my-release bitnami-ibm/kong
```

Use an external PostgreSQL database

```sh
helm install my-release bitnami-ibm/kong \
   --set postgresql.enabled=false \
   --set postgresql.external.host=_HOST_OF_YOUR_POSTGRESQL_INSTALLATION_ \
   --set postgresql.external.password=_PASSWORD_OF_YOUR_POSTGRESQL_INSTALLATION_ \
   --set postgresql.external.user=_USER_OF_YOUR_POSTGRESQL_INSTALLATION_
```

Deploy the Cassandra sub-chart

```sh
helm install my-release bitnami-ibm/kong \
   --set database=cassandra \
   --set postgresql.enabled=false \
   --set cassandra.enabled=true
```


Use an existing Cassandra installation

```sh
helm install my-release bitnami-ibm/kong \
   --set database=cassandra \
   --set postgresql.enabled=false \
   --set cassandra.enabled=false \
   --set cassandra.external.hosts[0]=_CONTACT_POINT_0_OF_YOUR_CASSANDRA_CLUSTER_ \
   --set cassandra.external.hosts[1]=_CONTACT_POINT_1_OF_YOUR_CASSANDRA_CLUSTER_ \
   --set cassandra.external.user=_USER_OF_YOUR_CASSANDRA_INSTALLATION_ \
   --set cassandra.external.password=_PASSWORD_OF_YOUR_CASSANDRA_INSTALLATION_
```

### Sidecars and Init Containers

If you have a need for additional containers to run within the same pod as Kong (e.g. an additional metrics or logging exporter), you can do so via the sidecars config parameter. Simply define your container according to the Kubernetes container spec.

```yaml
sidecars:
   - name: your-image-name
      image: your-image
      imagePullPolicy: Always
      ports:
   ​     - name: portname
   ​      containerPort: 1234
```


Similarly, you can add extra init containers using the initContainers parameter.

```yaml
initContainers:
 - name: your-image-name
   image: your-image
   imagePullPolicy: Always
   ports:
​     - name: portname
​      containerPort: 1234
```

### Adding extra environment variables

In case you want to add extra environment variables (useful for advanced operations like custom init scripts), you can use the kong.extraEnvVars property.

```yaml
kong:
  extraEnvVars:
    - name: KONG_LOG_LEVEL
​      value: error
```


Alternatively, you can use a ConfigMap or a Secret with the environment variables. To do so, use the `kong.extraEnvVarsCM` or the `kong.extraEnvVarsSecret` values.


The Kong Ingress Controller and the Kong Migration job also allow these kinds of configuration. the `ingressController.extraEnvVars`, `ingressController.extraEnvVarsCM`, `ingressController.extraEnvVarsSecret`, `migration.extraEnvVars`, `migration.extraEnvVarsCM` and `migration.extraEnvVarsSecret` values.

### Using custom init scripts

For advanced operations, the Bitnami Kong charts allows using custom init scripts that will be mounted in `/docker-entrypoint.init-db`. You can use a ConfigMap or a Secret (in case of sensitive data) for mounting these extra scripts. Then use the `kong.initScriptsCM` and `kong.initScriptsSecret` values.

```sh
elasticsearch.hosts[0]=elasticsearch-host
elasticsearch.port=9200
initScriptsCM=special-scripts
initScriptsSecret=special-scripts-sensitive
```

### Deploying extra resources

There are cases where you may want to deploy extra objects, such as KongPlugins, KongConsumers, amongst others. For covering this case, the chart allows adding the full specification of other objects using the extraDeploy parameter. The following example would activate a plugin at deployment time.

```yaml
## Extra objects to deploy (value evaluated as a template)
##
extraDeploy: |-
 - apiVersion: configuration.konghq.com/v1
   kind: KongPlugin
   metadata:
​     name: {{ include "common.names.fullname" . }}-plugin-correlation
​     namespace: {{ .Release.Namespace }}
​     labels: {{- include "common.labels.standard" . | nindent 6 }}
   config:
​     header_name: my-request-id
   plugin: correlation-id
```

### Setting Pod's affinity 

This chart allows you to set your custom affinity using the affinity parameter.

As an alternative, you can use of the preset configurations for pod affinity, pod anti-affinity, and node affinity available at the [bitnami/common](https://github.com/bitnami/charts/tree/master/bitnami/common#affinities) chart. To do so, set the podAffinityPreset, podAntiAffinityPreset, or `nodeAffinityPreset` parameters.
