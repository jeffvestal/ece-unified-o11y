# Let's make ECE observable End-To-End
#### From underlying infra metrics to users' activity
######  ECE Observability Project from a few SAs and friends

[![ECE o11y Dashboard](https://github.com/elastic/ece_o11y/blob/master/visulizations/ECE%20Admin%20Observability%20Dashboard.png "ECE o11y Dashboard")](https://github.com/elastic/ece_o11y/blob/master/visulizations/ECE%20Admin%20Observability%20Dashboard.png "ECE o11y Dashboard")


------------
# Problem Statement
> As a ECE DevOps admin, I want to have one central dashboard that displays ECE infra logs and metrics, user cluster logs and metrics, and Cluster utilization metrics. I further want to use that one cluster to create anomaly detection and threshold alerts with the new Alerting Framework.

### Details
ECE currently does not provide an out of the box total observability cluster that covers ECE infra compoents, uaer created culster health monitoring, and cluster user metrics in one central cluster. 

1. ECE system clusters are pinned to previous major version (6.8.8 as of this writing)
1a. 6.x does not have newer o11y features including transforms, lens, alerting UI, and others
2. User clusters can not send their monitoring data to the *logging-and-metrics* cluster
3. A user created observability cluster in 7.x (current major version as of writing) can not:
3a. Collect cluster monitoring metrics from system clusters
3b. Utilize ECE OOTB metricbeat / filebeat data collected in the *logging-and-metrics*  cluster

------------
# Potential Solutions
## Upgrade ECE system clusters to match latest release of user clusters
### Blockers
1. ECE is pinned to 6.8.8 and can not be upgraded by the end users
1a. Elastic Dev team has to upgrade as part of the stack pack

## Create separate user o11y cluster
### Issues
1. ECE infra logs and metrics that are auto instrumented and collected are still separate from the user configured monitoring data []
2. System Health metrics can not be sent to the User o11y cluster due to major version mismatch.

## Use CCS
### Setup Steps
1. Create a *User o11y* cluster. 
2. Create a *6x Bridge* cluster 
3. Configure CCS on *User o11y* cluster to pull data from 
3a. The *logging-and-metrics*  system cluster.
3b. Configure CCS on *User o11y* cluster to pull data from the
4. Configure system clusters to send cluster health data to *6x Bridge* cluster
5. Reconfigure metricbeat dashboards to use *:metricbeat CSS index pattern

### Advantages
1. One cluster now can view all ECE cluster's (system and user) health metrics on the Health UI App as well as on custom dashboards 
1a. Can centralize creating anomaly detection jobs and alerts from monitoring data
2. One cluster to view ECE and user logs

### Issues
1. Lots of custom configuration by the end user to get up a base level o11y cluster
2. System clusters can not be set as a *remote_cluster* for ccs by default
2a. workaround is to go into each system cluster's Advanced Settings in the Elasticsearch -> Edit UI and temporarily setting `system_owned: false` while adding as *remote_cluster*
3. Requirement for the *6x Bridge* cluster to collect system cluster's health metrics
3a. system cluster can collect their own health data and it will ship to the *User o11y* cluster but this violates the best practice of don't monitor yourself.
4. OOTB beat dashboards need to be edited to use the CCS index patterns
5. 6x Bridge CCS has to be configured through the ECE API. Configuring with ECE Admin UI will not work
```https://elastic.ecedemo.com:12443/api/v1/clusters/elasticsearch
# Look for the cluster_id for the bridge cluster and your ECE o11y Cluster
"cluster_id": "dec469aa536549e2bb5082aaaaaaaaaa",
      "cluster_name": "6.8 Bridge",
 ...
       "cluster_id": "43109d2b2a04410e8725f2xxxxxxxxxx,
      "cluster_name": "ECE Observability",

# Use that cluster_ID to configure
https://elastic.ecedemo.com:12443/api/v1/clusters/elasticsearch/43109d2b2a04410e8725f2xxxxxxxxxx/ccs/settings
{
    "remote_clusters": {
        "68_bridge" : {
        	"cluster_id": "dec469aa536549e2bb5082aaaaaaaaaa",
        	"skip_unavailable": true
        }
    }
}

```

# Decision
Even though it is a lot of end-user work, the only workable solution to collect all o11y data on a single cluster is to use the CCS Solution.

------------


# Data Flow Diagram
[![ECE o11y Data Flow](https://github.com/elastic/ece_o11y/blob/master/ECE%20o11y%20Data%20Flow.png "ECE o11y Data Flow")](https://github.com/elastic/ece_o11y/blob/master/ECE%20o11y%20Data%20Flow.png "ECE o11y Data Flow")
