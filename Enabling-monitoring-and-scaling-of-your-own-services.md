You can use OpenShift Monitoring for your own services in addition to monitoring the cluster. This way, you do not need to use an additional monitoring solution. This helps keeping monitoring centralized. Additionally, you can extend the access to the metrics of your services beyond cluster administrators. This enables developers and arbitrary users to access these metrics.

This is based on OpenShift 4.3. At this time it is only a Technical Preview. See https://docs.openshift.com/container-platform/4.3/monitoring/monitoring-your-own-services.html

###### Enabling monitoring of your own services

Make sure you are logged in as cluster-admin.

```bash
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    techPreviewUserWorkload:
      enabled: true
EOF
```

and check that the prometheus-user-workload pods were created

```bash
$ oc get pod -n openshift-user-workload-monitoring 
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-operator-7bcc9cc899-p8cbr   1/1     Running   1          10h
prometheus-user-workload-0             5/5     Running   6          10h
prometheus-user-workload-1             5/5     Running   6          10h
```

###### Create metrics colletion role

Create a new role for setting up metrics collection.

```bash
$ cat <<EOF | oc apply -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: monitor-crd-edit
rules:

- apiGroups: ["monitoring.coreos.com"]
  resources: ["prometheusrules", "servicemonitors", "podmonitors"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF
```

###### New project

Create a new project (monitor-demo) and give a normal user (developer) admin rights onto the project. Add the new created role to the user.

```bash
$ oc new-project monitor-demo
You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app django-psql-example

to build a new example application in Python. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
$ oc policy add-role-to-user admin developer -n monitor-demo 
clusterrole.rbac.authorization.k8s.io/admin added: "developer"
$ oc policy add-role-to-user monitor-crd-edit developer -n monitor-demo 
clusterrole.rbac.authorization.k8s.io/monitor-crd-edit added: "developer"

```

###### Login as the normal user.

```bash
$ oc login -u developer
Authentication required for https://api.rbaumgar.demo.net:6443 (openshift)
Username: developer
Password: 
Login successful.

You have one project on this server: "monitor-demo"

Using project "monitor-demo".
```

###### Deploying a sample service, monitor-demo-app end expose a route.

Based on an example you will find at [GitHub - rbaumgar/monitor-demo-app: Quarkus demo app to show Application Performance Monitoring(APM)](https://github.com/rbaumgar/monitor-demo-app) 

```bash
$ cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: monitor-demo-app
  name: monitor-demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: monitor-demo-app
  template:
    metadata:
      labels:
        app: monitor-demo-app
    spec:
      containers:
      - image: quay.io/rbaumgar/monitor-demo-app-jvm
        imagePullPolicy: IfNotPresent
        name: monitor-demo-app
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: monitor-demo-app
  name: monitor-demo-app
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
    name: web
  selector:
    app: monitor-demo-app
  type: ClusterIP
EOF
deployment.apps/monitor-demo-app created
service/monitor-demo-app created
$ oc expose svc monitor-demo-app 
route.route.openshift.io/monitor-demo-app exposed
```

###### Check new monitoring app

Check the router url with */hello* and see the hello message and the pod name. Do this multiple times.

```bash
$ export URL=$(oc get route monitor-demo-app -o jsonpath='{.spec.host}')
$ curl $URL/hello
hello from monitor-demo-app monitor-demo-app-78fc685c94-mtm28
```

###### Check the availalable metrics

See all available metrics */metrics* and only application specific metrics */metrics/application*.

```bash
$ curl $URL/metrics/application
# HELP application_org_example_rbaumgar_GreetingResource_greetings_total How many greetings we've given.
# TYPE application_org_example_rbaumgar_GreetingResource_greetings_total counter
application_org_example_rbaumgar_GreetingResource_greetings_total 1.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_rate_per_second gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_rate_per_second 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_one_min_rate_per_second gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_one_min_rate_per_second 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_five_min_rate_per_second gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_five_min_rate_per_second 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_fifteen_min_rate_per_second gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_fifteen_min_rate_per_second 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_min_seconds gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_min_seconds 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_max_seconds gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_max_seconds 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_mean_seconds gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_mean_seconds 0.0
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_stddev_seconds gauge
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_stddev_seconds 0.0
# HELP application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds A measure of how long it takes to perform the primality test.
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds summary
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds_count 0.0
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds{quantile="0.5"} 0.0
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds{quantile="0.75"} 0.0
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds{quantile="0.95"} 0.0
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds{quantile="0.98"} 0.0
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds{quantile="0.99"} 0.0
application_org_example_rbaumgar_PrimeNumberChecker_checksTimer_seconds{quantile="0.999"} 0.0
# HELP application_org_example_rbaumgar_PrimeNumberChecker_performedChecks_total How many primality checks have been performed.
# TYPE application_org_example_rbaumgar_PrimeNumberChecker_performedChecks_total counter
application_org_example_rbaumgar_PrimeNumberChecker_performedChecks_total 0.0

```

 With *greetings_total* you will see how often you have called the */hello* url.

###### Setting up metrics collection

To use the metrics exposed by your service, you need to configure OpenShift Monitoring to scrape metrics from the /metrics endpoint. You can do this using a ServiceMonitor, a custom resource definition (CRD) that specifies how a service should be monitored, or a PodMonitor, a CRD that specifies how a pod should be monitored. The former requires a Service object, while the latter does not, allowing Prometheus to directly scrape metrics from the metrics endpoint exposed by a Pod.

```bash
$ cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: monitor-demo-monitor
  name: monitor-demo-monitor
spec:
  endpoints:
  - interval: 30s
    port: web
    scheme: http
  selector:
    matchLabels:
      app: monitor-demo-app
EOF
servicemonitor.monitoring.coreos.com/monitor-demo-monitor created
$ oc get servicemonitor
NAME                   AGE
monitor-demo-monitor   42s
```









 7405  2020-03-30 16:01:47 oc get routes.route.openshift.io 
 7406  2020-03-30 16:01:57 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net
 7407  2020-03-30 16:02:02 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net/metric
 7408  2020-03-30 16:02:05 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net/metrics
 7409  2020-03-30 16:04:03 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net
 7410  2020-03-30 16:04:05 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net/metrics
 7411  2020-03-30 16:07:40 oc policy add-role-to-user admin demo -n ns1
 7412  2020-03-30 16:08:49 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net/metrics
 7413  2020-03-30 16:08:52 curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net
 7414  2020-03-30 16:11:00 for i in {1...1000}; curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net;done
 7415  2020-03-30 16:11:38 for i in {1...1000}; do curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net; sleep 10; done
 7416  2020-03-30 16:12:17 for i in {1..1000}; do curl prometheus-example-app-ns1.apps.rbaumgar.rhmw-integrations.net; sleep 10; done
 7417  2020-03-30 17:31:27 oc idle all -n ns1
 7418  2020-03-30 17:31:35 oc get pod
 7419  2020-03-30 17:31:44 oc idle --all -n ns1
 7420  2020-03-30 17:31:47 oc get pod
 7421  2020-03-30 17:44:44 curl -sH 'Accept: application/json'  https://api.openshift.com/api/upgrades_info/v1/graph?channel=stable-4.3 | jq
 7422  2020-03-30 18:16:22 oc whoami
 7423  2020-03-30 18:16:38 oc whoami --show-server
 7424  2020-03-30 18:16:38 export LE_API=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
 7425  2020-03-30 18:16:39 export LE_WILDCARD=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}')
 7426  2020-03-30 18:16:40 ~/acme.sh/acme.sh --issue -d ${LE_API} -d *.${LE_WILDCARD} --dns dns_aws
 7427  2020-03-30 18:16:41 export CERTDIR=$HOME/certificates
 7428  2020-03-30 18:16:41 mkdir -p ${CERTDIR}
 7429  2020-03-30 18:16:41 ~/acme.sh/acme.sh --install-cert -d ${LE_API} -d *.${LE_WILDCARD} --cert-file ${CERTDIR}/cert.pem --key-file ${CERTDIR}/key.pem --fullchain-file ${CERTDIR}/fullchain.pem --ca-file ${CERTDIR}/ca.cer
 7430  2020-03-30 18:16:41 oc create secret tls router-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/key.pem -n openshift-ingress
 7431  2020-03-30 18:16:41 oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec": { "defaultCertificate": { "name": "router-certs" }}}'
 7432  2020-03-30 18:16:42 oc get apiserver cluster -o yaml
 7433  2020-03-30 18:17:36 oc delete  secrets router-certs -n openshift-ingress
 7434  2020-03-30 18:17:40 oc whoami --show-server
 7435  2020-03-30 18:17:40 export LE_API=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
 7436  2020-03-30 18:17:40 export LE_WILDCARD=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}')
 7437  2020-03-30 18:17:40 ~/acme.sh/acme.sh --issue -d ${LE_API} -d *.${LE_WILDCARD} --dns dns_aws
 7438  2020-03-30 18:17:42 export CERTDIR=$HOME/certificates
 7439  2020-03-30 18:17:42 mkdir -p ${CERTDIR}
 7440  2020-03-30 18:17:42 ~/acme.sh/acme.sh --install-cert -d ${LE_API} -d *.${LE_WILDCARD} --cert-file ${CERTDIR}/cert.pem --key-file ${CERTDIR}/key.pem --fullchain-file ${CERTDIR}/fullchain.pem --ca-file ${CERTDIR}/ca.cer
 7441  2020-03-30 18:17:42 oc create secret tls router-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/key.pem -n openshift-ingress
 7442  2020-03-30 18:17:42 oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec": { "defaultCertificate": { "name": "router-certs" }}}'
 7443  2020-03-30 18:17:43 oc get apiserver cluster -o yaml
 7444  2020-03-30 18:19:20 oc project openshift-ingress
 7445  2020-03-30 18:19:23 oc get pod
 7446  2020-03-31 09:49:08 cd ~/demo/openshift/openshift-cd-demo/
 7447  2020-03-31 09:49:19 cd ../startstop/
 7448  2020-03-31 09:49:24 ./status.sh 
 7449  2020-03-31 09:49:34 ls
 7450  2020-03-31 09:49:51 mv kubelet-bootstrap-cred-manager-ds.yaml.yaml kubelet-bootstrap-cred-manager-ds.yaml
 7451  2020-03-31 09:49:58 vi kubelet-bootstrap-cred-manager-ds.yaml 
 7452  2020-03-31 13:37:31 oc get machineconfig | grep entitlement
 7453  2020-03-31 13:37:39 oc get machineconfig 
 7454  2020-03-31 14:46:46 ./stop.sh 
 7455  2020-03-31 14:49:00 ./start.sh 
 7456  2020-03-31 14:54:20 oc get node
 7457  2020-04-01 08:29:25 ./stop.sh 
 7458  2020-04-01 13:41:06 clear
 7459  2020-04-01 13:41:13 ./start.sh 
 7460  2020-04-01 16:09:25 ./stop.sh 
 7461  2020-04-01 16:13:18 ./status.sh 
 7462  2020-04-01 16:13:24 ./start.sh 
 7463  2020-04-01 16:15:15 oc login -u admin
 7464  2020-04-01 16:37:57 ./stop.sh 
 7465  2020-04-01 18:47:06 exit
 7466  2020-04-03 12:50:21 curl localhost:8080
 7467  2020-04-03 12:51:30 ls -a
 7468  2020-04-03 12:51:54 ls .mvn
 7469  2020-04-03 12:51:58 ls .mvn -R
 7470  2020-04-03 12:54:46 curl localhost:8080
 7471  2020-04-03 12:55:07 ls -R
 7472  2020-04-03 12:55:44 curl localhost:8080
 7473  2020-04-03 12:55:49 curl localhost:8080/hello
 7474  2020-04-03 12:55:53 curl localhost:8080/metrics
 7475  2020-04-03 13:00:26 curl localhost:8080/metrics/applications
 7476  2020-04-03 13:00:36 curl localhost:8080/metrics/application
 7477  2020-04-03 13:00:57 curl localhost:8080/metrics
 7478  2020-04-03 13:01:06 curl localhost:8080/metrics|grep -i greet
 7479  2020-04-03 13:01:19 ls
 7480  2020-04-03 13:01:27 curl localhost:8080/metrics
 7481  2020-04-03 13:01:52 curl http://localhost:8080/metrics/application
 7482  2020-04-03 13:02:24 curl localhost:8080/metrics/application
 7483  2020-04-03 13:02:35 curl localhost:8080/hello
 7484  2020-04-03 13:02:39 curl localhost:8080/metrics/application
 7485  2020-04-03 13:02:46 curl http://localhost:8080/metrics/application
 7486  2020-04-03 13:03:03 curl localhost:8080/hello
 7487  2020-04-03 13:03:29 curl http://localhost:8080/metrics/application
 7488  2020-04-03 13:04:23 curl http://localhost:8080/metrics/base
 7489  2020-04-03 13:04:26 curl http://localhost:8080/metrics/application
 7490  2020-04-03 13:04:43 curl localhost:8080/hello
 7491  2020-04-03 13:04:47 curl http://localhost:8080/metrics/application
 7492  2020-04-03 13:07:15 curl localhost:8080/prime/1
 7493  2020-04-03 13:07:17 curl localhost:8080/prime/2
 7494  2020-04-03 13:07:20 curl localhost:8080/prime/93
 7495  2020-04-03 13:07:28 curl localhost:8080/prime/95
 7496  2020-04-03 13:07:30 curl localhost:8080/prime/97
 7497  2020-04-03 13:07:37 curl http://localhost:8080/metrics/application
 7498  2020-04-03 13:08:00 clear
 7499  2020-04-03 13:08:02 curl http://localhost:8080/metrics/application
 7500  2020-04-03 13:08:39 curl http://localhost:8080/hello
 7501  2020-04-03 13:08:45 curl http://localhost:8080/metrics/application
 7502  2020-04-03 13:09:14 curl http://localhost:8080/prime/629521085409773
 7503  2020-04-03 13:09:18 curl http://localhost:8080/metrics/application
 7504  2020-04-03 13:10:17 mvn clean package -DskipTests
 7505  2020-04-03 13:10:24 ls target/
 7506  2020-04-03 13:23:55 curl http://localhost:8080/metrics/application
 7507  2020-04-03 13:44:02 pwd
 7508  2020-04-03 13:45:02 while [ true ] ; do         BITS=$(( ( RANDOM % 60 )  + 1 ));         NUM=$(openssl prime -generate -bits $BITS); echo $NUM; sleep 2; done
 7509  2020-04-03 13:46:20 while [ true ] ; do         BITS=$(( ( RANDOM % 60 )  + 1 ));         NUM=$(openssl prime -generate -bits $BITS); echo $NUM $BITS; sleep 2; done
 7510  2020-04-03 13:47:23 skopeo 
 7511  2020-04-03 13:47:50 ls
 7512  2020-04-03 13:51:06 curl http://localhost:8080/metrics/application
 7513  2020-04-03 13:52:40 while [ true ] ; do         BITS=$(( ( RANDOM % 60 )  + 1 ));         NUM=$(openssl prime -generate -bits $BITS);         curl http://localhost:8080/prime/${NUM};         sleep 2; done
 7514  2020-04-03 13:55:34 while [ true ] ; do         BITS=$(( ( RANDOM % 60 )  + 1 ));         NUM=$(openssl prime -generate -bits $BITS);echo $NUM;         curl http://localhost:8080/prime/${NUM};         sleep 2; echo " ";done
 7515  2020-04-07 09:07:21 code .
 7516  2020-04-07 09:09:49 mvn clean package -DskipTests
 7517  2020-04-07 09:10:03 curl http://localhost:8080/hello