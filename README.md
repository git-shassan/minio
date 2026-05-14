# MinIO Kubernetes Manifests — Manifest B

This document describes the Kubernetes/OpenShift YAML manifests used to deploy a MinIO object storage instance within the `minio` namespace.

---

## Secret — MinIO Credentials

Stores the MinIO root username and password as a Kubernetes Secret, allowing the Deployment to securely reference credentials without hardcoding them in plain text.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-secret
  namespace: minio
stringData:
  minio_root_password: minio123
  minio_root_user: minio
```

---

## Service — MinIO API and UI

Exposes both the MinIO S3-compatible API (port `9000`) and the web-based management UI (port `9090`) through a single ClusterIP Service, using named ports for clean routing integration.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: minio
spec:
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: api
    port: 9000
    protocol: TCP
    targetPort: 9000
  - name: ui
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: minio
  sessionAffinity: None
  type: ClusterIP
```

---

## PersistentVolumeClaim

Requests 50Gi of persistent storage for MinIO's data directory, ensuring that object data survives pod restarts or rescheduling.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pvc
  namespace: minio
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  volumeMode: Filesystem
```

---

## Deployment

Deploys a single MinIO pod with the official image, mounting the persistent volume at `/data`, injecting credentials from the Secret, and configuring resource limits, health probes, and a `Recreate` rollout strategy for safe upgrades.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: minio
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: minio
    spec:
      containers:
      - args:
        - server
        - /data
        - --console-address
        - :9090
        env:
        - name: MINIO_ROOT_USER
          valueFrom:
            secretKeyRef:
              key: minio_root_user
              name: minio-secret
        - name: MINIO_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: minio_root_password
              name: minio-secret
        image: quay.io/minio/minio:latest
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 30
          periodSeconds: 5
          successThreshold: 1
          tcpSocket:
            port: 9000
          timeoutSeconds: 1
        name: minio
        ports:
        - containerPort: 9000
          protocol: TCP
        - containerPort: 9090
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          tcpSocket:
            port: 9000
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 250m
            memory: 1Gi
          requests:
            cpu: 20m
            memory: 100Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /data
          name: data
          subPath: minio
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: minio-pvc
```

---

## Route — MinIO API

Creates an OpenShift Route that exposes the MinIO S3-compatible API externally over HTTPS, with TLS edge termination and automatic HTTP-to-HTTPS redirect.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-api
  namespace: minio
spec:
  port:
    targetPort: api
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: edge
  to:
    kind: Service
    name: minio-service
    weight: 100
  wildcardPolicy: None
```

---

## Route — MinIO UI

Creates an OpenShift Route that exposes the MinIO web management console externally over HTTPS, with TLS edge termination and automatic HTTP-to-HTTPS redirect.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: minio-ui
  namespace: minio
spec:
  port:
    targetPort: ui
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: edge
  to:
    kind: Service
    name: minio-service
    weight: 100
  wildcardPolicy: None
```
