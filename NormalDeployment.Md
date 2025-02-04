At first deploy elasticsearch Operator:

1. Insatll CRDs
 - kubectl create -f https://download.elastic.co/downloads/eck/2.16.1/crds.yaml

2. Install Operator with RBAC rules
 - kubectl apply -f https://download.elastic.co/downloads/eck/2.16.1/operator.yaml

3. Elasticsearch Deployment manifest
```yaml
---
#Secret
apiVersion: v1
kind: Secret
metadata:
 name: efk-elasticsearch-sec
 namespace: my namespace
data:
 password: pass
 username: username
type: Opaque

---

apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: my-elasticsearch
  namespace: my-namespace
spec:
  version: 8.0.0  # Replace with the appropriate version
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  auth:
    fileRealm:
      - secretName: my-elasticsearch-eck-sec
  nodeSets:
    - name: default
      count: 3  # Adjust the number of replicas as needed
      volumeClaimTemplates:
        - metadata:
            name: elasticsearch-data
          spec:
            accessModes:
              - ReadWriteOnce  # Adjust the access mode as needed
            resources:
              requests:
                storage: 10Gi  # Adjust the storage size as needed
            storageClassName: standard  # Adjust the storage class as needed
      podTemplate:
        spec:
          containers:
            - name: elasticsearch
              resources:
                requests:
                  cpu: "500m"
                  memory: "1Gi"
                limits:
                  cpu: "1"
                  memory: "2Gi"
      config:
        node.store.allow_mmap: false
```

4. Configure fuletnd
Deploy fluentd ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: namespace-name
data:
  fluent.conf: |
    <source>
      @type tail
      read_from_head true
      tag kubernetes.*
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      exclude_path ["/var/log/containers/fluent*"]
      exclude_path ["/var/log/containers/ingress-nginx*"]
      <parse>
        @type regexp
        #https://regex101.com/r/ZkOBTI/1
        expression ^(?<time>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.[^Z]*Z)\s(?<stream>[^\s]+)\s(?<character>[^\s])\s(?<message>.*)$
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV.fetch('KUBERNETES_SERVICE_HOST') + ':' + ENV.fetch('KUBERNETES_SERVICE_PORT') + '/api'}"
      verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
      ca_file "#{ENV['KUBERNETES_CA_FILE']}"
      skip_labels "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_LABELS'] || 'false'}"
      skip_container_metadata "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_CONTAINER_METADATA'] || 'false'}"
      skip_master_url "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_MASTER_URL'] || 'false'}"
      skip_namespace_metadata "#{ENV['FLUENT_KUBERNETES_METADATA_SKIP_NAMESPACE_METADATA'] || 'false'}"
      include_namespace true
      include_labels ["app", "deployment"] # Adjust as needed
    </filter>
    <match **>
      @type elasticsearch
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST'] || 'efk-elasticsearch-es-http.core-ns.svc.cluster.local'}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT'] || '9200'}"
      user "#{ENV['FLUENT_ELASTICSEARCH_USER'] || 'aes'}"
      password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD'] || 'aes1234'}"
      scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME'] || 'http'}"
      index_name kube_containers
      type_name fluentd
      time_key time
      include_timestamp true
    </match>    

```

Deploy fluentd daemonset
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: namespace
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
  namespace: namespace
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: efk
    meta.helm.sh/release-namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - namespaces
  verbs:
  - get
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
  namespace: namespace
  labels:
    app.kubernetes.io/managed-by: Helm
  annotations:
    meta.helm.sh/release-name: efk
    meta.helm.sh/release-namespace: default
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: namespace
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: namespace
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.15.3-debian-elasticsearch7-1.1
          env:
            - name:  FLUENT_ELASTICSEARCH_HOST
              value: "efk-elasticsearch-es-http.{{ .Values.namespace }}.svc.cluster.local"
            - name:  FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
            - name: FLUENT_ELASTICSEARCH_USER
              value: elasticsearh_username
            - name: FLUENT_ELASTICSEARCH_PASSWORD
              value: elasticsearch_pass
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config-volume
              mountPath: /fluentd/etc/fluent.conf
              subPath: fluent.conf
      terminationGracePeriodSeconds: 30
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config-volume
          configMap:
            name: fluentd-config # this is our fluentd cm name
```

Kibana Deployment:

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: efk-kibana
  namespace: {{.Values.namespace}}
spec:
  version: 8.14.1
  http:
    tls:
      selfSignedCertificate:
        disabled: true
  count: 1
  elasticsearchRef:
    name: efk-elasticsearch
  config:
    server.basePath: "/application-log"
    server.rewriteBasePath: true
  podTemplate:
    spec:
      containers:
      - name: kibana
        readinessProbe:
          httpGet:
            path: /application-log
            port: 5601
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
          successThreshold: 1
```