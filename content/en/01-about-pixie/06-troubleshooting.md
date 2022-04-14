---
title: "Troubleshooting"
metaTitle: "About Pixie | Troubleshooting"
metaDescription: "Troubleshooting problems with Pixie."
order: 6
---

This page describes how to troubleshoot Pixie. We frequently answer questions on our [community Slack](https://slackin.px.dev/) channel and in response to [GitHub issues](https://github.com/pixie-io/pixie/issues). You can also check those two places to see if your question has already been addressed. To better understand how Pixie's various components interact, please see the [Architecture](/about-pixie/architecture/overview) overview.

#### Troubleshooting Deployment

- [How do I check the status of Pixie's components?](#troubleshooting-deployment-how-do-i-check-the-status-of-pixie's-components)
- [How do I get the Pixie debug logs?](#troubleshooting-deployment-how-do-i-get-the-pixie-debug-logs)
- [My deployment is stuck / fails.](#troubleshooting-deployment-my-deployment-is-stuck-fails)
- [Why does my cluster show as unavailable / unhealthy in the Live UI?](#troubleshooting-deployment-why-does-my-cluster-show-as-unavailable-unhealthy-in-the-live-ui)

#### Troubleshooting Operation

- [Why does my cluster show as disconnected in the Live UI?](#troubleshooting-operation-why-does-my-cluster-show-as-disconnected-in-the-live-ui)
- [Why can’t I see data?](#troubleshooting-operation-why-can't-i-see-data)
- [Why can’t I see application profiles / flamegraphs for my pod / node?](#troubleshooting-operation-why-can't-i-see-application-profiles-flamegraphs-for-my-pod-node)
- [Why is the vizier-pem pod’s memory increasing?](#troubleshooting-operation-why-is-the-vizier-pem-pod's-memory-increasing)
- [Troubleshooting tracepoint scripts.](#troubleshooting-operation-troubleshooting-pixie-tracepoint-scripts)

## Troubleshooting Deployment

### How do I check the status of Pixie's components?

An initial overview of Pixie can be retrieved by listing all `Vizier` pods to verify whether all pods have the status `Running`:

```shell
$ px debug pods
  NAME                                          PHASE    RESTARTS  MESSAGE  REASON  START TIME
  pl/pl-nats-0                                  RUNNING  0                          2022-04-08T13:17:15-07:00
  pl/vizier-certmgr-76f6f89ddf-6sm76            RUNNING  0                          2022-04-08T13:17:26-07:00
  pl/vizier-cloud-connector-57c7588c67-56p5j    RUNNING  0                          2022-04-08T13:17:26-07:00
  pl/vizier-metadata-0                          RUNNING  1                          2022-04-08T13:17:27-07:00
  pl/vizier-proxy-79bd7d9b55-w5zv5              RUNNING  0                          2022-04-08T13:17:27-07:00
  pl/vizier-query-broker-75478b59d4-smjt2       RUNNING  0                          2022-04-08T13:17:27-07:00
  px-operator/vizier-operator-7955d5669d-wbwzz  RUNNING  0                          2022-04-08T13:16:56-07:00
  pl/kelvin-8665676895-7dcgg                    RUNNING  0                          2022-04-08T13:17:26-07:00
  pl/vizier-pem-bjkbm                           RUNNING  0                          2022-04-08T13:17:27-07:00
  pl/vizier-pem-znglq                           RUNNING  0                          2022-04-08T13:17:27-07:00
```

Cloud components can be checked by running `kubectl get pods -n plc`.

### How do I get the Pixie debug logs?

Install Pixie’s [CLI tool](/installing-pixie/install-schemes/cli) and run `px collect-logs.` This command will output a zipped file named `pixie_logs_<datestamp>.zip` in the working directory. The selected kube-context determines the Kubernetes cluster that outputs the logs, so make sure that you are pointing to the correct cluster.

### My deployment is stuck / fails

We recommend running through the following troubleshooting flow to determine where the deployment has failed.

<svg title='Troubleshooting the Deployment of Pixie' src='troubleshoot-flow.svg' />

*Deploy with CLI gets stuck at “Wait for PEMs/Kelvin”*

This step of the deployment waits for the newly deployed Pixie PEM pods to become ready and available. This step can take several minutes.

If some `vizier-pem` pods are not ready, use kubectl to check the individual pod’s events or check Pixie’s debug logs (which also include pod events).

If pods are still stuck in pending, but there are no Pixie specific errors, check that there is no resource pressure (memory, CPU) on the cluster.

*Deploy with CLI fails to pass health checks.*

This step of the deployment checks that the `vizier-cloud-connector` pod can successfully run a query on the `kelvin` pod. These queries are brokered by the `vizer-query-broker` pod. To debug a failing health check, check the Pixie debug logs for those pods for specific errors.

*Deploy with CLI fails waiting for the Cloud Connector to come online.*

This step of the deployment checks that the Cloud Connector can successfully communicate with Pixie Cloud. To debug this step, check the Pixie debug logs for the `vizier-cloud-connector` pod, check the firewall, etc.

### Why does my cluster show as unavailable / unhealthy in the Live UI?

Confirm that all of the `pl` and `px-operator` namespace pods are ready and available using `px debug pods`. Deploying Pixie usually takes anywhere between 5-7 minutes. Once Pixie is deployed, it can take a few minutes for the UI to show that the cluster is healthy.

To debug, follow the steps in the “Deploy with CLI fails to pass health checks” section in the [above question](/about-pixie/faq/#my-deployment-is-stuck-fails). As long as the Kelvin pod, plus at least one PEM pod is up and running, then your cluster should not show as unavailable.

## Troubleshooting Operation

### Why does my cluster show as disconnected in the Live UI?

`Cluster '<CLUSTER_NAME>' is disconnected. Pixie instrumentation on 'CLUSTER_NAME' is disconnected. Please redeploy Pixie to the cluster or choose another cluster.`

This error indicates that the `vizier-cloud-connector` pod is not able to connect to the cloud properly. To debug, check the events / logs for the `vizier-cloud-connector` pod. Note that after deploying Pixie, it can take a few minutes for the UI to show the cluster as available.

### Why can’t I see data?

*Live UI shows an error.*

Error `Table 'http_events' not found` is usually an issue with deploying Pixie onto nodes with unsupported kernel versions. Check that your kernel version is supported [here](/installing-pixie/requirements/).

Error `Invalid Vis Spec: Missing value for required arg service.` occurs when a script has a required argument that is missing a value. Required script arguments are denoted with an asterisk after the argument name. For example, px/service has a required variable for service name. Select the required argument drop-down box in the top left and enter a value.

Error `Unexpected error rpc error: code = Unknown desc = rpc error: code = Canceled desc = context canceled` is associated with a query timing out. Try reducing the `start_time` window.

*Live UI does not show an error, but data is missing.*

It is possible that you need to adjust the `start_time` window. The `start_time` window expects a negative relative time (e.g. `-5m`) or an absolute time in the format `2020-07-13 18:02:5.00 +0000`.

If specific services / requests are missing, it is possible that Pixie doesn't support the encryption library used by that service. You can see the list of encryption libraries supported by Pixie [here](/about-pixie/data-sources/#encryption-libraries).

If specific services / requests are missing, it is possible that your application was not built with [debug information](/reference/admin/debug-info). See the [Data Sources](/about-pixie/data-sources) page to see which protocols and/or encryption libraries require a build with debug information.

### Why can’t I see application profiles / flamegraphs for my pod / node?

Continuous profiling currently only supports Go/C++/Rust. The [Roadmap](/about-pixie/roadmap) contains plans to expand this support to Java, Ruby, Python, etc.

### Why is the vizier-pem pod’s memory increasing?

This is expected behavior. Pixie stores the data it collects in-memory on the nodes in your cluster; data is not sent to any centralized backend cloud outside of the cluster. So what you are observing is simply the data that it is collecting.

Pixie has a minimum 1GiB memory requirement per node. The default deployment is 2GiB of memory. This limit can be configured with the `--pem_memory_limit` flag when deploying Pixie. Using a value less than 1GiB is not currently recommended.

### Troubleshooting Pixie tracepoint scripts

*I’m not seeing any data for my distributed bpftrace script.*

Rather than query data already collected by the Pixie Platform, Distributed bpftrace Deployment scripts extend the Pixie platform to collect new data sources by deploying tracepoints when the script is run. The first time this type of script is run, it will deploy the probe and query the data (but there won't be much data at this point). Re-running the script after the probe has had more time to gather data will produce more results.

*I'm getting error that tracepoints failed to deploy.*

Run the `px/tracepoint_status` script. It should show a longer error message in the "Tracepoint Status" table.

*How do I remove a tracepoint table?*

It is not currently possible to remove a table. Instead, we recommend renaming the table name (e.g. table_name_0) while debugging the script.