# Shell-operator metrics

Shell-operator exports Prometheus metrics to the `/metrics` path. The default port is 9115.

## Metrics

* `shell_operator_hook_run_seconds{hook="", binding="", queue=""}` — a histogram with hook execution times. "hook" label is a name of the hook, "binding" is a binding name from configuration, "queue" is a queue name where hook is queued.
* `shell_operator_hook_run_errors_total{hook="hook-name", binding="", queue=""}` — this is the counter of hooks’ execution errors. It only tracks errors of hooks with the disabled `allowFailure` (i.e. respective key is omitted in the configuration or the `allowFailure: false` parameter is set). This metric has a "hook" label with the name of a failed hook.
* `shell_operator_hook_run_allowed_errors_total{hook="hook-name", binding="", queue=""}` — this is the counter of hooks’ execution errors. It only tracks errors of hooks that are allowed to exit with an error (the parameter `allowFailure: true` is set in the configuration). The metric has a "hook" label with the name of a failed hook.
* `shell_operator_hook_run_success_total{hook="hook-name", binding="", queue=""}` — this is the counter of hooks’ success execution. The metric has a "hook" label with the name of a succeeded hook.
* `shell_operator_hook_enable_kubernetes_bindings_success{hook=""}` — this gauge have two values: 0.0 if Kubernetes informers are not started and 1.0 if Kubernetes informers are successfully started for a hook.   
* `shell_operator_hook_enable_kubernetes_bindings_errors_total{hook=""}` — a counter of failed attempts to start Kubernetes informers for a hook. 
* `shell_operator_hook_enable_kubernetes_bindings_seconds{hook=""}` — a gauge with time of Kubernetes informers start.

* `shell_operator_tasks_queue_length{queue=""}` — a gauge showing the length of the working queue. This metric can be used to warn about stuck hooks. It has the "queue" label with the queue name.

* `shell_operator_task_wait_in_queue_seconds_total{hook="", binding="", queue=""}` — a counter with seconds that the task to run a hook elapsed in the queue.

* `shell_operator_live_ticks` — a counter that increases every 10 seconds. This metric can be used for alerting about an unhealthy Shell-operator. It has no labels.

* `shell_operator_kube_jq_filter_duration_seconds{hook="", binding="", queue=""}` — a histogram with jq filter timings.

* `shell_operator_kube_event_duration_seconds{hook="", binding="", queue=""}` — a histogram with kube event handling timings.

* `shell_operator_kube_snapshot_objects{hook="", binding="", queue=""}` — a gauge with count of cached objects (the snapshot) for particular binding.

* `shell_operator_kube_snapshot_bytes{hook="", binding="", queue=""}` — a gauge with size in bytes of cached objects for particular binding. Each cached object contains a Kubernetes object and/or result of jqFilter depending on the binding configuration. The size is a sum of the length of Kubernetes object in JSON format and the length of jqFilter‘s result in JSON format.

* `shell_operator_kubernetes_client_request_result_total` — a counter of requests made by kubernetes/client-go library. 

* `shell_operator_kubernetes_client_request_latency_seconds` — a histogram with latency of requests made by kubernetes/client-go library. 

* `shell_operator_tasks_queue_action_duration_seconds{queue_name="", queue_action=""}` — a histogram with measurements of low level queue operations. Use QUEUE_ACTIONS_METRICS="no" to disable this metric.

## Custom metrics

Hooks can export metrics by writing a set of operations in JSON format into $METRICS_PATH file.

Operation to register a counter and increase its value:

```json
{"name":"metric_name","add":1,"labels":{"label1":"value1"}}
```

Operation to register a gauge and set its value:

```json
{"name":"metric_name","set":33,"labels":{"label1":"value1"}}
```

Labels are not required, but Shell-operator adds a `hook` label.

Several metrics can be expored at once. For example, this script will create 2 metrics:

```
echo '{"name":"hook_metric_count","add":1,"labels":{"label1":"value1"}}' >> $METRICS_PATH
echo '{"name":"hook_metrics_items","add":1,"labels":{"label1":"value1"}}' >> $METRICS_PATH
```

The metric name is used as-is, so several hooks can export same metric name. It is responsibility of hooks‘ developer to maintain consistent label cardinality.

There is no operation to delete metrics and no ttl mechanism yet. If hook set value for metric, this value is exported until the process ends.

To expire metrics, you can use a "group" field. When Shell-operator receives operations with "group", it expires previous metrics with the same group and apply new values. This grouping works across hooks and label values.

For example:

`hook1.sh` returns these metrics:

```
echo '{"group":"hook1", "name":"hook_metric","add":1,"labels":{"kind":"pod"}}' >> $METRICS_PATH
echo '{"group":"hook1", "name":"hook_metric","add":1,"labels":{"kind":"replicaset"}}' >> $METRICS_PATH
echo '{"group":"hook1", "name":"hook_metric","add":1,"labels":{"kind":"deployment"}}' >> $METRICS_PATH
echo '{"group":"hook1", "name":"hook1_special_metric","set":12,"labels":{"label1":"value1"}}' >> $METRICS_PATH
echo '{"group":"hook1", "name":"common_metric","set":300,"labels":{"source":"source3"}}' >> $METRICS_PATH
echo '{"name":"common_metric","set":100,"labels":{"source":"source1"}}' >> $METRICS_PATH
```

`hook2.sh` returns these metrics:

```
echo '{"group":"hook2", "name":"hook_metric","add":1,"labels":{"kind":"configmap"}}' >> $METRICS_PATH
echo '{"group":"hook2", "name":"hook_metric","add":1,"labels":{"kind":"secret"}}' >> $METRICS_PATH
echo '{"group":"hook2", "name":"hook2_special_metric","set":42}' >> $METRICS_PATH
echo '{"name":"common_metric","set":200,"labels":{"source":"source2"}}' >> $METRICS_PATH
```

Prometheus scrapes these metrics:

```
# HELP hook_metric hook_metric
# TYPE hook_metric counter
hook_metric{hook="hook1.sh", kind="pod"} 1
hook_metric{hook="hook1.sh", kind="replicaset"} 1
hook_metric{hook="hook1.sh", kind="deployment"} 1
hook_metric{hook="hook2.sh", kind="configmap"} 1
hook_metric{hook="hook2.sh", kind="secret"} 1
# HELP hook2_special_metric hook2_special_metric
# TYPE hook1_special_metric gauge
hook1_special_metric{hook="hook1.sh", label1="value1"} 12
# HELP hook2_special_metric hook2_special_metric
# TYPE hook2_special_metric gauge
hook2_special_metric{hook="hook2.sh"} 42
# HELP common_metric common_metric
# TYPE common_metric gauge
common_metric{hook="hook1.sh", source="source1"} 100
common_metric{hook="hook1.sh", source="source3"} 300
common_metric{hook="hook2.sh", source="source2"} 200
```

On next execution of `hook1.sh` values for `hook_metric{kind="replicaset"}`, `hook_metric{kind="deployment"}`, `common_metric{source="source3"}` and `hook1_special_metric` are expired and hook returns only one metric:

```
echo '{"group":"hook1", "name":"hook_metric","add":1,"labels":{"kind":"pod"}}' >> $METRICS_PATH
```

Shell-operator expires previous values for group "hook1" and updates value for `hook_metric{hook="hook1.sh", kind="pod"}`. Values for group `hook2` and `common_metric` without group are left intact. Now Prometheus scrapes these metrics:

```
# HELP hook_metric hook_metric
# TYPE hook_metric counter
hook_metric{hook="hook1.sh", kind="pod"} 2
hook_metric{hook="hook2.sh", kind="configmap"} 1
hook_metric{hook="hook2.sh", kind="secret"} 1
# HELP hook2_special_metric hook2_special_metric
# TYPE hook2_special_metric gauge
hook2_special_metric{hook="hook2.sh"} 42
# HELP common_metric common_metric
# TYPE common_metric gauge
common_metric{hook="hook1.sh", source="source1"} 100
common_metric{hook="hook2.sh", source="source2"} 200
```
