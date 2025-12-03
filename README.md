Deploy Redis in EKS (Manual Kubernetes Setup)

We’ll deploy a single Redis instance first.
Later you can scale it or make it HA (clustered Redis).

Step 1: Create namespace
kubectl create namespace redis

Step 2: Create Redis ConfigMap

This holds Redis configuration.

# redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: redis
data:
  redis.conf: |
    maxmemory 256mb
    maxmemory-policy allkeys-lru

kubectl apply -f redis-config.yaml

Step 3: Create Redis Deployment
# redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7.0
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: redis-config
              mountPath: /usr/local/etc/redis/redis.conf
              subPath: redis.conf
          args: ["redis-server", "/usr/local/etc/redis/redis.conf"]
      volumes:
        - name: redis-config
          configMap:
            name: redis-config

kubectl apply -f redis-deployment.yaml

Step 4: Expose Redis Service (internal)
# redis-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: redis
spec:
  type: ClusterIP
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    app: redis

kubectl apply -f redis-service.yaml


✅ This makes Redis available internally in your cluster at:
redis.redis.svc.cluster.local:6379

Step 5: Verify Redis is running
kubectl get pods -n redis
kubectl get svc -n redis


You should see:

NAME     READY   STATUS    RESTARTS   AGE
redis-...  1/1     Running   0          1m

NAME     TYPE        CLUSTER-IP     PORT(S)    AGE
redis    ClusterIP   10.100.x.x     6379/TCP   1m

Step 6: Test Redis (from another pod)

You can launch a temporary pod and connect:

kubectl run redis-client --rm -it --image=redis:7.0 -- bash


Inside the pod:

redis-cli -h redis.redis.svc.cluster.local
> set mykey "Hello Redis"
OK
> get mykey
"Hello Redis"

🧩 5️⃣ How to Use Redis in Your Application

Your app connects to Redis using the internal DNS name:

redis.redis.svc.cluster.local:6379


For example:

Python Django:

CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://redis.redis.svc.cluster.local:6379/1",
    }
}


Node.js (Express):

import { createClient } from "redis";
const client = createClient({ url: "redis://redis.redis.svc.cluster.local:6379" });
await client.connect();

⚙️ 6️⃣ (Optional) Deploy Redis Using Helm (production-friendly)

If you prefer a faster setup with persistence, passwords, etc.:

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install redis bitnami/redis -n redis \
  --set auth.enabled=true \
  --set auth.password=MySecureRedisPass \
  --set architecture=standalone


To connect:

redis.redis.svc.cluster.local:6379
password: MySecureRedisPass
