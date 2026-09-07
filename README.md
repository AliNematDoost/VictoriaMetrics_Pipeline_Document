# Task Report 

## Implementing Metrics in Django application 

At first all metrics added to code based on their functionality. Metrics are added to `./clusterproject/metrics.py` and they are used where they should to change the metrics while the code is running.
Added metrics are exposed on /metrics.

Django project repo : `https://github.com/AliNematDoost/Django-Project-Hamamouz`

## Deploying application on k3s cluster 

Then deployed the application on k3s cluster and exposed endpoints on host `nematdoust.osdl.ir` using ingress. now /api/metrics is accessible using http://nematdoust.osdl.ir/api/metrics.

## Deploying VictoriaMetrics Pipeline


### Concepts I learned here
Before starting anything, I preferred to read more about concepts in this part, so I got these:

1. We are going to first install VictoriaMetrics operator. but what exactly iis operator here?

   Operator is a Controller placed in k8s that manages and creates resources for CRDs we define. Kubenetes will know that we have new types of kind and accepts them but does not know what to do with them.
   at this time operator comes in and manages new objects with new kinds defined earlier and create kubenetes resources for them as they need. ( Deployment also has a controller that creates replicaset for
   deployment objects )

2. Got deeper about components of VictoriaMetrics and what they gonna do. like:
   - VMServiceScrape: This is going to identify which targets we are going to collect their metrics
   - VMAgent: This is going to collect metrics from targets and remote write them
   - VMSingle: This is going to store collected metrics is storage

Now for deploying the Pipeline I followed this structure: 

django /metrics --> django service <-- VNServiceScrape matches and targets this <--VMAgent collects metrics and writes <-- VMSingle stores metrics on storage


### VMSingle

`VMSingle` is the VictoriaMetrics component responsible for **storing the collected metrics**.

Metrics are retained for 4 days. Data older than this is automatically removed.

```yaml
storage:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

This requests a **1 GiB persistent volume** for storing the metrics. `ReadWriteOnce` means the volume can be mounted for read/write by one node.

---

### VMAgent

`VMAgent` is responsible for **discovering scrape targets, collecting metrics, and sending them to VictoriaMetrics**.

```yaml
serviceScrapeNamespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: monitoring-system
```

This tells VMAgent to look for `VMServiceScrape` resources in the `monitoring-system` namespace which we have made VmServiceScrape in it.

```yaml
serviceScrapeSelector:
  matchLabels: {}
```

An empty selector means that VMAgent accepts all matching `VMServiceScrape` objects in the selected namespace.

```yaml
replicaCount: 1
scrapeInterval: 15s
```

There is one VMAgent replica, and it collects metrics every 15 seconds.

```yaml
remoteWrite:
  - url: "http://vmsingle-hamamooz-vmsingle.monitoring-system.svc:8429/api/v1/write"
```

After collecting metrics, VMAgent sends them to the VMSingle instance through its Kubernetes Service.
Found the name and port of VMSingle service as below:
```
k get svc -n monitoring-system
NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
vm-operator-victoria-metrics-operator   ClusterIP   10.43.142.87    <none>        8080/TCP,9443/TCP   17h
vmagent-hamamooz-vmagent                ClusterIP   10.43.175.31    <none>        8429/TCP            17h
vmsingle-hamamooz-vmsingle              ClusterIP   10.43.130.242   <none>        8429/TCP,8428/TCP   17h
```

**no matter to use which port for connecting to VMSingle service because both of them route the traffic to the same port of VMSingle pod:**
```
k describe svc vmsingle-hamamooz-vmsingle -n monitoring-system
Name:                     vmsingle-hamamooz-vmsingle
Namespace:                monitoring-system
Labels:                   app.kubernetes.io/component=monitoring
                          app.kubernetes.io/instance=hamamooz-vmsingle
                          app.kubernetes.io/name=vmsingle
                          managed-by=vm-operator
Annotations:              <none>
Selector:                 app.kubernetes.io/component=monitoring,app.kubernetes.io/instance=hamamooz-vmsingle,app.kubernetes.io/name=vmsingle,managed-by=vm-operator
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.130.242
IPs:                      10.43.130.242
Port:                     http  8429/TCP
TargetPort:               8429/TCP
Endpoints:                10.42.1.67:8429
Port:                     http-alias  8428/TCP
TargetPort:               8429/TCP
Endpoints:                10.42.1.67:8429
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

Therefore, the flow is:

**VMServiceScrape --> VMAgent --> VMSingle**

---

### VMServiceScrape

`VMServiceScrape` tells VMAgent **which Kubernetes Service to scrape and where the metrics endpoint is located**.

```yaml
namespaceSelector:
  matchNames:
    - application
```

This tells VMAgent to look for the target Service in the `application` namespace.

```yaml
selector:
  matchLabels:
    service: django
```

It selects the Django Service using its label:

```yaml
service: django
```
For that reason I modified the django-service manifest and added a label to its metadata section.

```yaml
endpoints:
  - port: metrics
    path: /api/metrics
```
For that reason I modified the django-service manifest and added a name to port section.

This tells VMAgent to scrape the selected Service on the port named `metrics` and request:

```text
/api/metrics
```

So this resource creates the connection:

```text
Django Service --> VMServiceScrape --> VMAgent
```

Together with the other components, the complete pipeline is:

```text
Django /api/metrics --> Django Service --> VMServiceScrape --> VMAgent --> VMSingle
```

**Note**
The final version of django-service after updates:
```
apiVersion: v1
kind: Service
metadata:
  name: django-service
  namespace: application
  labels:
    service: django
spec:
  selector:
    app: django
  ports:
    - name: metrics
      port: 8000
      targetPort: 8000
```



## Problems and Challenges Solved 

### Ingress creation and VMUI api call

First of all I created a new Ingress Rule for exposing VMUI on host I already have. so I created this :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vm-ingress
  namespace: monitoring-system
spec:
  ingressClassName: traefik
  rules:
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /vmui
            pathType: Prefix
            backend:
              service:
                name: vmsingle-hamamooz-vmsingle
                port:
                  number: 8429
```

**Also in this step I learned that Kubernetes interprets that the Backend service specified in ingress.yaml is placed in the same namespace as ingress is created in it. So here Kunernetes searches for 
service called `vmsingle-hamamooz-vmsingle` in monitoring-system namespace. As a result of this, it would be better to create the ingress rule in the same namespace as service**

After creating this ingress VVMUI was available on `http://nematdoust.osdl.ir/vmui` but searching query did not work. I checked the network tab of browser inspect and found out that the queries api call is on a different path : `http://nematdoust.osdl.ir/prometheus/api/v1/query_range`

and we configured ingress to only accept paths with prefix /vmui. 

So I configured ingress rule as below:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vm-ingress
  namespace: monitoring-system
spec:
  ingressClassName: traefik
  rules:
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /vmui
            pathType: Prefix
            backend:
              service:
                name: vmsingle-hamamooz-vmsingle
                port:
                  number: 8429
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /prometheus
            pathType: Prefix
            backend:
              service:
                name: vmsingle-hamamooz-vmsingle
                port:
                  number: 8429
```

And created a new rule to also accept /prometheus which is used for query API calles in vmui.

### Getting 400 for scraping /api/metrics

After port forwarding VMAgent and getting the list of targets using this command:
```
curl localhost:8429/targets
```

I found out that the VMAgent trying to scrape is getting 400 error from django pod. that was because of because of `DJANGO_ALLOWED_HOSTS` in Django application that only allows specific hosts. The value of DJANGO_ALLOWED_HOSTS used to be `nematdoust.osdl.ir` in order to send request to Backend pod only through ingress. But when VMAgent tries to connect to Django pod to scrape and collect metrics it uses http://<pod_IP>:8000/api/metrics and host <pod_IP>:8000 is not allowed in Django settings. 

So changed the value of DJANGO_ALLOWED_HOSTS to '*' and restarted Django pod to use the new config and after that VMAgent was able to collect metrics successfully:
```
curl localhost:8429/targets
job=serviceScrape/monitoring-system/django-metrics/0 (1/1 up)
	state=up, endpoint=http://10.42.0.66:8000/api/metrics, labels={container="django",endpoint="metrics",instance="10.42.0.66:8000",job="django-service",namespace="application",pod="django-756c467667-gj8kx",service="django-service"}, scrapes_total=1250, scrapes_failed=0, last_scrape=1.718s ago, scrape_duration=44ms, scrape_response_size=55KiB, samples_scraped=489, error=
job=serviceScrape/monitoring-system/vmagent-hamamooz-vmagent/0 (1/1 up)
	state=up, endpoint=http://10.42.1.68:8429/metrics, labels={container="vmagent",endpoint="http",instance="10.42.1.68:8429",job="vmagent-hamamooz-vmagent",namespace="monitoring-system",pod="vmagent-hamamooz-vmagent-6cfd6d867d-svt8p",service="vmagent-hamamooz-vmagent",victoriametrics_app="true"}, scrapes_total=4694, scrapes_failed=0, last_scrape=6.201s ago, scrape_duration=4ms, scrape_response_size=73KiB, samples_scraped=1077, error=
job=serviceScrape/monitoring-system/vmagent-hamamooz-vmagent/1 (1/1 up)
	state=up, endpoint=http://10.42.1.68:8435/metrics, labels={container="config-reloader",endpoint="8435",instance="10.42.1.68:8435",job="vmagent-hamamooz-vmagent-reloader-http",namespace="monitoring-system",pod="vmagent-hamamooz-vmagent-6cfd6d867d-svt8p",service="vmagent-hamamooz-vmagent"}, scrapes_total=4694, scrapes_failed=0, last_scrape=1.721s ago, scrape_duration=2ms, scrape_response_size=17KiB, samples_scraped=290, error=
job=serviceScrape/monitoring-system/vmsingle-hamamooz-vmsingle/0 (1/1 up)
	state=up, endpoint=http://10.42.1.67:8429/metrics, labels={container="vmsingle",endpoint="http",instance="10.42.1.67:8429",job="vmsingle-hamamooz-vmsingle",namespace="monitoring-system",pod="vmsingle-hamamooz-vmsingle-5bf65bdbcd-sj2p7",service="vmsingle-hamamooz-vmsingle",victoriametrics_app="true"}, scrapes_total=4693, scrapes_failed=0, last_scrape=10.218s ago, scrape_duration=6ms, scrape_response_size=67KiB, samples_scraped=1101, error=
```

## Iteration 2: Security

Now every user can query VMSimgle without any authentication or authorization. In order to limit access to writing ot or reading from VMSingle I decided to use VMUser+VMAuth. 

### VMUser
For that reason I created two VMUser CRs, One for writing credentials and one for reading:
```
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMUser
metadata:
  name: vmui-reader
  namespace: monitoring-system
spec:
  username: reader
  password: PASSWORD
  targetRefs:
    - static:
        url: "http://vmsingle-hamamooz-vmsingle.monitoring-system.svc:8429"
      paths:
        - "/vmui"
        - "/vmui/.*"
        - "/prometheus/.*"
---
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMUser
metadata:
  name: vmagent-writer
  namespace: monitoring-system
spec:
  username: writer
  password: PASSWORD
  targetRefs:
    - static:
        url: "http://vmsingle-hamamooz-vmsingle.monitoring-system.svc:8429"
      paths:
        - "/api/v1/write"
```
I have declared a username and password for each user which they will be authenticated with them in VMAuth. Also declared a permitted paths for each user, which VMAuth will use them to check if user is authorized to access paths or not. And also declared a targetRefs for each user that is the prefix-url that VMAuth sends new request to url + input_path.

After applying new VMUsers, operator will create a secret containing username and password for each VMUser:
```
k describe secret vmuser-vmagent-writer -n monitoring-system
Name:         vmuser-vmagent-writer
Namespace:    monitoring-system
Labels:       app.kubernetes.io/component=monitoring
              app.kubernetes.io/instance=vmagent-writer
              app.kubernetes.io/name=vmuser
              managed-by=vm-operator
Annotations:  <none>

Type:  Opaque

Data
====
password:  9 bytes
username:  6 bytes
```

### VMAuth

VMAuth is a reverse proxy that sits front of VMSingle and every request that used to go directly to VMSingle should now be routed to VMAuth and checked first ( to see if the user is authorized or not )

```
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAuth
metadata:
  name: vmauth
  namespace: monitoring-system
spec:
  selectAllByDefault: true
  replicaCount: 1
```

It selects all users defined in namespace monitoring-system using VMUser. 

Now every request that used to go directly to VMSingle should now be routed to VMAuth, for that reason I have configured ingress rules to route traffic to VMAuth instead of VMSingle:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vm-ingress
  namespace: monitoring-system
spec:
  ingressClassName: traefik
  rules:
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /vmui
            pathType: Prefix
            backend:
              service:
                name: vmauth-vmauth
                port:
                  number: 8427
    - host: nematdoust.osdl.ir
      http:
        paths:
          - path: /prometheus
            pathType: Prefix
            backend:
              service:
                name: vmauth-vmauth
                port:
                  number: 8427
```

After VMAuth receives the request, it performs two checks in sequence:

1. **Authentication**: it extracts the credentials from the request's Basic Auth header and matches them against the set of `VMUser` objects it loaded (via `selectAllByDefault: true`). If no `VMUser` matches, the request is rejected with `401 Unauthorized`.
2. **Authorization**: once a matching user is found, VMAuth checks whether the requested path matches one of that user's `paths` entries. If the path isn't in the allowed list, the request is rejected. the user is a valid VMAuth user, but not permitted to access that particular endpoint.

If both checks pass, VMAuth builds a new outgoing request by concatenating that `VMUser`'s `targetRefs.static.url` with the original request's path and query string, then forwards it to VMSingle. VMSingle receives this exactly as if the client had called it directly. it has no awareness that authentication happened upstream.

The write path follows the same two-step logic, just with a different `VMUser`: 

VMAgent attaches Basic Auth credentials (pulled from the `vmuser-vmagent-writer` secret) to every `remoteWrite` request it sends to VMAuth. VMAuth authenticates those credentials against the `vmagent-writer` VMUser, confirms `/api/v1/write` is in its permitted paths, and only then forwards the write to VMSingle. The mechanism is identical to the read flow. the only difference is which `VMUser` is involved and which path is being checked :

```
  remoteWrite:
    - url: "http://vmauth-hamamooz-vmauth.monitoring-system.svc:8427/api/v1/write"
      basicAuth:
        username:
          name: vmuser-vmagent-writer
          key: username
        password:
          name: umuser-vmagent-writer
          key: password
```

credentials are extracted from secrets that were created by operator after applying VMUsers.

## Testing VictoriaMetrics

<img width="1919" height="813" alt="image" src="https://github.com/user-attachments/assets/b7a69f2f-9906-4edb-b925-232a58187534" />

For metric `hamamooz_backup_jobs_total` we have the state above shown in VMUI. Now creating a new backup and getting the list of backups of an app will results in change in metrics:

<img width="1919" height="813" alt="image" src="https://github.com/user-attachments/assets/375ee891-23c8-4600-a39d-a67fd99d863d" />

So with this test, we can understand that VMUser and VMAuth are performing as expected too. I am reading and running queries in VMUI ( indirectly on VMSingle ) with valid credentials and VMAgent is collecting and remote writing new metrics on VMSingle with valid credentials. So based on that VMAuth is also working alright and authenticates users and accesses. 

- accessing VMUI with wrong credentials results in getting 401 unathorized:

<img width="569" height="283" alt="image" src="https://github.com/user-attachments/assets/5043c86a-cb2c-4b76-81c8-ddcc1bf54513" />

## Pre-built Dashboards for VMSingle and VMAgent in Grafana

I have created components for deploying Grafana which are placed in `grafana/` ( deployment + service + ingress rule + pvc to persistent dashboards and datas we add to Grafana )

Now Grafana is accessible in `http://nematdoust.osdl.ir/` with defauld credentials.

I have added two pre-built dashboards ( 10229 and 12683 ) for VMSingle and VMAgent and also created a datasource to VMSingle and tried to expose metrics that are not, in order to avoid `No Data` in dashboard panels. 


## Alerting Pipeline

Now I am going to deploy an alerting pipeline. For that reason I targeted one of my own metrics as condition of alert. 

For creating alerting pipeline I have created these VictoriaMetrics components:
1. VMRule: Define the rule ( condition ) that should be checked and fired if condition is true

```
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: backup-rule
  namespace: monitoring-system
  labels:
    app: vmrule
spec:
  groups:
    - name: test-alert
      rules:
        - alert: BackupJobsCountTooHigh
          expr: sum(hamamooz_backup_jobs_total{operation="create"}) > 10
          for: 1s
```

Defining label for VMRole component is important because of using it in VMAlert to select this VMRule and use its condition in evaluation.

We must define a group for alert that is used later in VMAlertManager. Then a rule is defined with an alert name and expression that is going to be queried on data source. Also we specify that if condition is true for 1 second, state of alert will be changed from `pending` to `firing` 
   
2. VMAlert: Get rules from VMRule and query them to VMSingle, if true, notify VMAlertManager and remote write to VMSingle ( optional but nice to have )

```
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAlert
metadata:
  name: vmalert
  namespace: monitoring-system
spec:
  replicaCount: 1
  datasource:
    url: "http://vmsingle-vmsingle.monitoring-system.svc:8429"
  notifiers:
    - url: "http://vmalertmanager-vmalertmanager.monitoring-system.svc:9093"
  remoteWrite:
    url: "http://vmsingle-vmsingle.monitoring-system.svc:8429"
  evaluationInterval: "5s"
  ruleSelector:
    matchLabels:
      app: vmrule
```

VMAlert is going to query expression of VMRule on VMSingle ( using this api call : `http://vmsingle-vmsingle.monitoring-system.svc:8429/api/v1/query?query={exp}` ) and based on result, it fires alert to VMAlertmanager. It also remote writes the alert into VMSingle ( using this api call : `http://vmsingle-vmsingle.monitoring-system.svc:8429/api/v1/write` ). VMAlert evaluates the condition every 5 seconds.  

  
3. VMAlertManager: Check fired alerts and send them periodically to a service to expose alerts to user.

```
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAlertmanager
metadata:
  name: vmalertmanager
  namespace: monitoring-system
spec:
  replicaCount: 1
  configRawYaml: |
    route:
      receiver: 'webhook'
      group_wait: 10s
      group_interval: 30s
      repeat_interval: 1m
    receivers:
      - name: 'webhook'
        webhook_configs:
          - url: 'http://webhook-service.monitoring-system.svc.cluster.local:8080/webhookapp/alerts'
            send_resolved: true
```

I have defined a receiver here to send alerts. The receiver of alert is chosen and also specified in `receivers` section. Also some other configs are needed as below:
**group_wait**: after getting a fired alert, VMAlertManager first waits for 10 seconds in order to get some other alerts with the same group name and fire them all together with only one notification instead of one notif for each of them. Also if the state of alert changes from firing to pending, we will receive a notification for that because of `send_resolved: true`.

**group_interval**: VMAlertManager searches each group of alerts and check if new alert is fired or if `repeat_interval` is reached, if one of them is happened then a notif of alert will be sent. 

**repeat_interval**: If an alert stays in firing state, it will be sent every 1 minute as a reminder.
  
4. Alert exposing service created using Flask: Get alerts from VMAlertmanager and show them to user.

I have developed a notification REST-API service using Flask. It gets alerts using POST request on `/webhookapp/alerts`  and returns the list of alerts using GET request on the same path. 

Service is exposed using Ingress on `nematdoust.osdl.ir/webhookapp/alerts` and returns the list of alerts already fired. 

You can find more about webhook app in this repo: https://github.com/AliNematDoost/REST_API_Webhook


## End to End Test for Alert pipeline

First of all I should mention that I have chosen a metric and a condition that makes the expression to be always True. ( number of create operations of hamamooz_backup_jobs_total metric is 14 which is more than 10 )

So we expect alert to be fired continuously and notifications to be shown in `nematdoust.osdl.ir/webhookapp/alerts` like this :
<img width="1172" height="947" alt="image" src="https://github.com/user-attachments/assets/fb858c43-77cc-4950-b5d0-0f9f85b4ad9b" />

Notification webhook is correctly showing fired alerts of the same group and name we already created in VMRule. 

but let's check alert status after each hop of pipeline :

1. First I want to check if VMAlert has successfully taken the expression from VMRule:

For that purpose, First I port-forwarded service of VMAlert on port 8080 of localhost:

- get the list of services:
```
k get svc -n monitoring-system
NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
grafana-service                         ClusterIP   10.43.161.153   <none>        3000/TCP                     2d5h
vm-operator-victoria-metrics-operator   ClusterIP   10.43.142.87    <none>        8080/TCP,9443/TCP            3d22h
vmagent-vmagent                         ClusterIP   10.43.19.250    <none>        8429/TCP                     2d
vmalert-vmalert                         ClusterIP   10.43.209.169   <none>        8080/TCP                     46h
vmalertmanager-vmalertmanager           ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   45h
vmauth-vmauth                           ClusterIP   10.43.156.155   <none>        8427/TCP                     3d
vmsingle-vmsingle                       ClusterIP   10.43.110.115   <none>        8429/TCP,8428/TCP            3d
webhook-service                         ClusterIP   10.43.97.99     <none>        8080/TCP                     18h
```

- port-forward the service of VMAlert:

```
k port-forward svc/vmalert-vmalert -n monitoring-system 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

- Now I got the list of alerts using this API call:

```
curl -s http://localhost:8080/api/v1/alerts | jq
{
  "status": "success",
  "data": {
    "alerts": [
      {
        "state": "firing",
        "name": "BackupJobsCountTooHigh",
        "value": "14",
        "labels": {
          "alertgroup": "test-alert",
          "alertname": "BackupJobsCountTooHigh"
        },
        "annotations": {},
        "activeAt": "2026-09-07T06:12:05Z",
        "id": "13676737222207199333",
        "rule_id": "7093531774342171864",
        "group_id": "14948295105770102364",
        "expression": "sum(hamamooz_backup_jobs_total{operation=\"create\"}) > 10",
        "source": "http://vmalert-vmalert-7d7d5fbb97-s8ljr:8080/vmalert/alert?group_id=14948295105770102364&alert_id=13676737222207199333",
        "restored": false,
        "stabilizing": false
      }
    ]
  }
}
```

As we expected, our alert is present in list and its state is `firing`. Also the value of query ( 14 ) is given in output, so that proves VMAlert has queried the expression on VMSingle successfully and got the value of metric.

So this pipeline is working as expected: 
**Expression of Rule defined in VMRule --> VMAlert --> Query executed on VMSingle by VMAlert**

2. Next hop is notifier which is VMAlertManager that should send firing alert to webhook:

VMAlert is actually sending alerts to VMAlertmanager:
```
k port-forward svc/vmalert-vmalert -n monitoring-system 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080


curl -s http://localhost:8080/api/v1/notifiers | jq
{
  "status": "success",
  "data": {
    "notifiers": [
      {
        "kind": "static",
        "targets": [
          {
            "address": "http://vmalertmanager-vmalertmanager.monitoring-system.svc:9093/api/v2/alerts",
            "labels": {},
            "lastError": ""
          }
        ]
      }
    ]
  }
}
```

And VMAlertManager is also getting alert from VMAlert successfully:
```
k -n monitoring-system port-forward statefulset/vmalertmanager-vmalertmanager 9093:9093
Forwarding from 127.0.0.1:9093 -> 9093
Forwarding from [::1]:9093 -> 9093


curl -s http://localhost:9093/api/v2/alerts | jq
[
  {
    "annotations": {},
    "endsAt": "2026-09-07T15:05:39.061Z",
    "fingerprint": "7d4ae5877a4dd249",
    "receivers": [
      {
        "name": "webhook"
      }
    ],
    "startsAt": "2026-09-05T18:05:30.000Z",
    "status": {
      "inhibitedBy": [],
      "mutedBy": [],
      "silencedBy": [],
      "state": "active"
    },
    "updatedAt": "2026-09-07T15:05:19.057Z",
    "generatorURL": "http://vmalert-vmalert-7d7d5fbb97-s8ljr:8080/vmalert/alert?group_id=14948295105770102364&alert_id=13676737222207199333",
    "labels": {
      "alertgroup": "test-alert",
      "alertname": "BackupJobsCountTooHigh"
    }
  }
]
```

3. Getting the list of alerts in `http://nematdoust.osdl.ir/webhookapp/alerts` proves that POST requests from VMAlertManager reach webhook. ( since only service sending request to webhook is VMAlertManager )
