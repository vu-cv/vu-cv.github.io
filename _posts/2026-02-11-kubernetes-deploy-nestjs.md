---
layout: article
title: Kubernetes – Deploy NestJS lên K8s cơ bản
tags: [kubernetes, k8s, nestjs, docker, devops, deploy, cloud]
---
Kubernetes (K8s) orchestrates containers — tự động scale, restart khi crash, rolling deploy không downtime. Bài này hướng dẫn deploy NestJS lên K8s từ Dockerfile đến production-ready deployment.

## 1. Kiến trúc cơ bản

```
Internet → LoadBalancer → Ingress → Service → Pods
                                              ├── Pod 1 (NestJS)
                                              ├── Pod 2 (NestJS)
                                              └── Pod 3 (NestJS)
```

- **Pod**: Unit nhỏ nhất — 1 hoặc nhiều container
- **Deployment**: Quản lý replica, rolling update
- **Service**: Stable endpoint cho Pods (IP thay đổi khi restart)
- **Ingress**: Route HTTP traffic vào Service
- **ConfigMap/Secret**: Config và credentials

## 2. Dockerfile production

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production=false
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app

# Non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nestjs -u 1001

COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/package.json ./

USER nestjs

ENV NODE_ENV=production PORT=3000
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s \
  CMD wget -qO- http://localhost:3000/health/live || exit 1

CMD ["node", "dist/main.js"]
```

## 3. Kubernetes Manifests

### ConfigMap và Secret

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: shopxyz-config
  namespace: production
data:
  NODE_ENV: production
  PORT: "3000"
  DB_PORT: "5432"
  REDIS_PORT: "6379"

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: shopxyz-secrets
  namespace: production
type: Opaque
stringData:
  DB_HOST: "shopxyz-db.xxx.rds.amazonaws.com"
  DB_PASSWORD: "StrongPassword123!"
  JWT_SECRET: "your-very-long-jwt-secret-key"
  REDIS_URL: "redis://shopxyz-redis:6379"
  OPENAI_API_KEY: "sk-xxx"
```

### Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopxyz-api
  namespace: production
  labels:
    app: shopxyz-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopxyz-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Tối đa 1 pod thêm khi update
      maxUnavailable: 0  # Không down pod nào khi update → zero-downtime
  template:
    metadata:
      labels:
        app: shopxyz-api
    spec:
      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/shopxyz-api:latest
          ports:
            - containerPort: 3000

          envFrom:
            - configMapRef:
                name: shopxyz-config
            - secretRef:
                name: shopxyz-secrets

          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3

      # Graceful shutdown
      terminationGracePeriodSeconds: 30

      # Phân tán pods ra nhiều nodes
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: shopxyz-api
```

### Service

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: shopxyz-api-svc
  namespace: production
spec:
  selector:
    app: shopxyz-api
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP  # Internal — Ingress sẽ expose ra ngoài
```

### Ingress (với Nginx Ingress Controller)

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shopxyz-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"  # Max upload size
spec:
  tls:
    - hosts:
        - api.shopxyz.com
      secretName: shopxyz-tls
  rules:
    - host: api.shopxyz.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shopxyz-api-svc
                port:
                  number: 80
```

### HorizontalPodAutoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shopxyz-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shopxyz-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale khi CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## 4. Deploy Commands

```bash
# Apply tất cả manifests
kubectl apply -f k8s/ -n production

# Xem trạng thái
kubectl get pods -n production
kubectl get deployments -n production
kubectl describe pod <pod-name> -n production

# Xem logs
kubectl logs -f deployment/shopxyz-api -n production

# Rolling update (deploy image mới)
kubectl set image deployment/shopxyz-api \
  api=123456789.dkr.ecr.ap-southeast-1.amazonaws.com/shopxyz-api:v2.0.0 \
  -n production

# Rollback khi có lỗi
kubectl rollout undo deployment/shopxyz-api -n production

# Scale manual
kubectl scale deployment shopxyz-api --replicas=5 -n production

# Exec vào pod để debug
kubectl exec -it <pod-name> -n production -- sh
```

## 5. GitHub Actions — CI/CD

```yaml
# .github/workflows/deploy.yml
- name: Build and push Docker image
  run: |
    aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY
    docker build -t $ECR_REGISTRY/shopxyz-api:${{ github.sha }} .
    docker push $ECR_REGISTRY/shopxyz-api:${{ github.sha }}

- name: Deploy to Kubernetes
  run: |
    aws eks update-kubeconfig --name shopxyz-cluster --region ap-southeast-1
    kubectl set image deployment/shopxyz-api \
      api=$ECR_REGISTRY/shopxyz-api:${{ github.sha }} \
      -n production
    kubectl rollout status deployment/shopxyz-api -n production --timeout=120s
```

## 6. Graceful Shutdown trong NestJS

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // K8s gửi SIGTERM khi terminate pod
  app.enableShutdownHooks();

  await app.listen(3000);
}

// Trong AppModule hoặc service
@Injectable()
export class AppService implements BeforeApplicationShutdown {
  beforeApplicationShutdown(signal?: string) {
    console.log(`Gracefully shutting down (${signal})`);
    // Close DB connections, flush queues...
  }
}
```

## 7. Kết luận

- **Deployment + HPA**: Tự scale theo CPU/memory — không cần can thiệp thủ công
- **Rolling Update + maxUnavailable: 0**: Zero-downtime deployment
- **Health probes**: K8s tự restart pod lỗi, không route traffic đến pod chưa ready
- **Resource limits**: Bắt buộc — tránh một pod "ăn" toàn bộ node resources
- **Graceful shutdown**: NestJS cần handle SIGTERM — hoàn thành request đang xử lý trước khi tắt

K8s có learning curve cao nhưng là chuẩn industry — master Deployment, Service, Ingress, HPA là đủ cho hầu hết use cases.
