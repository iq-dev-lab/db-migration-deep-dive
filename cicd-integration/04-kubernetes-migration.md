# Kubernetes 배포와 마이그레이션

---

## 🎯 핵심 질문

- Init Container를 사용하여 마이그레이션을 앱 시작 전에 자동으로 실행하려면?
- 마이그레이션을 완전히 분리한 Job으로 실행하되, Deployment 배포를 연쇄적으로 진행하려면?
- Helm을 사용할 때 `helm.sh/hook: pre-upgrade`로 업그레이드 전에 마이그레이션을 실행하려면?
- 마이그레이션 실패 시 Pod을 시작하지 않도록 보호하려면?
- 여러 환경(dev, staging, prod)에서 환경별 DB 정보를 안전하게 관리하려면?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kubernetes 환경에서의 마이그레이션은 **클라우드 네이티브 배포**의 핵심입니다:

1. **자동화**: 마이그레이션을 배포 파이프라인에 자동 통합 (수동 개입 최소화)
2. **일관성**: 모든 환경(dev/staging/prod)에서 동일한 프로세스
3. **멱등성**: Pod 재시작 시에도 마이그레이션이 안전하게 실행 (이미 적용된 것은 건너뜀)
4. **버전 관리**: Helm `values.yaml`로 환경별 DB 정보 관리
5. **모니터링**: Kubernetes Event와 Pod 로그로 마이그레이션 성공/실패 추적

특히 **Init Container 패턴은 가장 단순**하고, **Job 패턴은 가장 유연**합니다. 팀의 규모와 요구사항에 따라 선택해야 합니다.

---

## 😱 흔한 실수 (Before — ...)

### 실수 1: "Init Container에서 마이그레이션 실패해도 무시하고 앱 시작"

```yaml
# deployment.yaml (잘못된 예)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      initContainers:
      - name: db-migrate
        image: flyway/flyway:9
        command:
          - sh
          - -c
          - |
            flyway -url=$DB_URL migrate || echo "Migration failed, continuing anyway"
            # ^ 위험! 실패해도 계속 진행
      
      containers:
      - name: app
        image: myapp:v1.0
```

**결과**:
- 마이그레이션 실패해도 앱 시작
- 앱이 예상하지 못한 스키마로 시작 → 에러 로그만 쌓임
- 문제의 원인을 찾기 어려움

---

### 실수 2: "Deployment와 Job의 마이그레이션 순서를 보장하지 않음"

```yaml
# deployment.yaml (문제)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  spec:
    replicas: 3

---
# job.yaml (별도 파일)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration

---
# kubectl apply -f . 
# 또는 helm install ...
# 
# 마이그레이션 Job이 먼저 생성되지 않음!
# Deployment가 먼저 시작될 수도 있음
```

**결과**:
- 마이그레이션 전에 앱이 구 스키마로 시작
- Race condition → 일부 Pod은 성공, 일부는 실패

---

### 실시 3: "Pod 재시작으로 Init Container가 여러 번 실행되는데도 멱등성을 고려 안 함"

```
Pod 시작 (첫 번째):
  Init Container 실행
    V1, V2, V3 마이그레이션 적용
    (flyway_schema_history에 기록됨)

Pod 다운 → Pod 재시작 (두 번째):
  Init Container 실행
    V1, V2, V3 마이그레이션... 또 실행할까?
    
  Flyway: flyway_schema_history 확인
    V1, V2, V3이 이미 적용되어 있음 → 스킵
    ✓ 멱등성 보장 (다시 실행되지 않음)
```

**하지만 아래 경우는 위험**:
```yaml
initContainers:
- name: custom-init
  command:
    - sh
    - -c
    - |
      mysql -u $DB_USER -p$DB_PASS << 'EOF'
      INSERT INTO seed_data (...) VALUES (...);  # 멱등성 없음!
      # 재시작되면 중복 INSERT → 오류
      EOF
```

---

### 실수 4: "Secret과 ConfigMap이 없어서 DB 정보를 하드코딩"

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      initContainers:
      - name: db-migrate
        image: flyway/flyway:9
        env:
        - name: FLYWAY_URL
          value: "jdbc:mysql://prod-db.us-east-1.rds.amazonaws.com:3306/mydb"  # 위험!
        - name: FLYWAY_USER
          value: "admin_user"  # 위험!
        - name: FLYWAY_PASSWORD
          value: "MyS3cr3tP@ssw0rd"  # 보안 위험!
```

**결과**:
- YAML 파일이 Git에 커밋되면 보안 정보 노출
- 누구든 프로덕션 DB 접근 가능

---

## ✨ 올바른 접근 (After — ...)

### 올바른 패턴 1: Init Container (단순함)

```yaml
# deployment.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: app-prod

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: app-prod
data:
  DB_URL: jdbc:mysql://mysql.prod.svc.cluster.local:3306/mydb
  MIGRATIONS_DIR: /migrations

---

apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: app-prod
type: Opaque
stringData:
  DB_USER: migration_user
  DB_PASSWORD: "SecureP@ssw0rd123!"

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: app-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      serviceAccountName: app
      
      # Init Container: 마이그레이션 실행
      initContainers:
      - name: db-migrate
        image: flyway/flyway:9
        imagePullPolicy: Always
        
        command:
          - sh
          - -c
          - |
            set -e  # 오류 발생 시 즉시 중단
            
            echo "Starting database migration..."
            
            flyway \
              -url="${DB_URL}" \
              -user="${DB_USER}" \
              -password="${DB_PASSWORD}" \
              -locations=filesystem:/migrations \
              -baselineOnMigrate=true \
              -outOfOrder=false \
              validate
            
            echo "Validation passed, running migration..."
            
            flyway \
              -url="${DB_URL}" \
              -user="${DB_USER}" \
              -password="${DB_PASSWORD}" \
              -locations=filesystem:/migrations \
              -baselineOnMigrate=true \
              migrate
            
            echo "Migration completed successfully"
        
        env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_URL
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        
        volumeMounts:
        - name: migrations
          mountPath: /migrations
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      
      # Application Container
      containers:
      - name: app
        image: myapp:v1.2.0
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 8081
        
        env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_URL
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        - name: SPRING_FLYWAY_ENABLED
          value: "false"  # Init Container에서 이미 실행했으므로 비활성화
        
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1024Mi"
            cpu: "1000m"
        
        volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
      
      # Volume: 마이그레이션 파일
      volumes:
      - name: migrations
        configMap:
          name: db-migrations
          items:
          - key: V1__initial.sql
            path: V1__initial.sql
          - key: V2__add_users.sql
            path: V2__add_users.sql
      
      - name: config
        configMap:
          name: app-config
      
      # Pod 종료 시간
      terminationGracePeriodSeconds: 30

---

# 마이그레이션 파일을 ConfigMap으로 관리
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-migrations
  namespace: app-prod
data:
  V1__initial.sql: |
    CREATE TABLE IF NOT EXISTS users (
      id BIGINT PRIMARY KEY AUTO_INCREMENT,
      email VARCHAR(255) NOT NULL UNIQUE,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    CREATE TABLE IF NOT EXISTS posts (
      id BIGINT PRIMARY KEY AUTO_INCREMENT,
      user_id BIGINT NOT NULL,
      title VARCHAR(255) NOT NULL,
      content LONGTEXT,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (user_id) REFERENCES users(id)
    );
  
  V2__add_users.sql: |
    ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'ACTIVE';
    ALTER TABLE users ADD COLUMN last_login TIMESTAMP NULL;

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
  namespace: app-prod

---

apiVersion: v1
kind: Service
metadata:
  name: app
  namespace: app-prod
spec:
  selector:
    app: app
  ports:
  - name: http
    port: 8080
    targetPort: http
  - name: metrics
    port: 8081
    targetPort: metrics
  type: ClusterIP
```

---

### 올바른 패턴 2: Job으로 완전 분리

```yaml
# db-migration-job.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: app-prod

---

apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: app-prod
data:
  DB_URL: jdbc:mysql://mysql.prod.svc.cluster.local:3306/mydb

---

apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: app-prod
type: Opaque
stringData:
  DB_USER: migration_user
  DB_PASSWORD: "SecureP@ssw0rd123!"

---

apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-{{ .Release.Revision }}  # Helm에서 사용
  namespace: app-prod
  labels:
    app: migration
    version: "{{ .Chart.Version }}"
spec:
  # 완료된 Job은 1개만 유지 (이전 Job은 자동 삭제)
  ttlSecondsAfterFinished: 3600  # 1시간 후 자동 삭제
  
  # 실패 시 재시도
  backoffLimit: 3
  
  # Job 타임아웃
  activeDeadlineSeconds: 300  # 5분
  
  template:
    metadata:
      labels:
        app: migration
    spec:
      serviceAccountName: migration
      
      containers:
      - name: flyway
        image: flyway/flyway:9
        imagePullPolicy: Always
        
        command:
          - sh
          - -c
          - |
            set -e
            
            echo "=== Database Migration Job ==="
            echo "Starting at: $(date)"
            
            # Validate
            echo ""
            echo "Step 1: Validating migrations..."
            flyway \
              -url="${DB_URL}" \
              -user="${DB_USER}" \
              -password="${DB_PASSWORD}" \
              -locations=filesystem:/migrations \
              validate
            
            # Migrate
            echo ""
            echo "Step 2: Running migrations..."
            flyway \
              -url="${DB_URL}" \
              -user="${DB_USER}" \
              -password="${DB_PASSWORD}" \
              -locations=filesystem:/migrations \
              migrate
            
            echo ""
            echo "Completed at: $(date)"
            echo "✓ Migration successful"
        
        env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_URL
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        
        volumeMounts:
        - name: migrations
          mountPath: /migrations
        
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      
      volumes:
      - name: migrations
        configMap:
          name: db-migrations
      
      # Job 실패 시 자동 재시도 안 함
      restartPolicy: Never

---

# Deployment는 Job이 완료된 후 생성 (Helm에서 처리)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: app-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myapp:v1.2.0
        
        env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            configMapKeyRef:
              name: db-config
              key: DB_URL
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_USER
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
        - name: SPRING_FLYWAY_ENABLED
          value: "false"
        
        ports:
        - containerPort: 8080
        
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5

---

# 마이그레이션 Job을 위한 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: migration
  namespace: app-prod

---

# Job이 Pod을 생성할 수 있도록 권한 부여
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: migration
  namespace: app-prod
rules:
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: migration
  namespace: app-prod
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: migration
subjects:
- kind: ServiceAccount
  name: migration
  namespace: app-prod
```

---

### 올바른 패턴 3: Helm Hook으로 Pre-upgrade 마이그레이션

```yaml
# helm/templates/db-migration.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-db-migration-{{ .Release.Revision }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{ include "app.labels" . | nindent 4 }}
  annotations:
    # Helm Hook: upgrade 또는 install 전에 이 Job 실행
    "helm.sh/hook": pre-upgrade,pre-install
    # Job 완료를 기다림
    "helm.sh/hook-weight": "1"
    # 이전 Hook Job은 자동 삭제
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  ttlSecondsAfterFinished: 3600
  backoffLimit: 3
  template:
    spec:
      containers:
      - name: flyway
        image: "{{ .Values.migration.image }}:{{ .Values.migration.tag }}"
        command:
          - sh
          - -c
          - |
            set -e
            echo "Running pre-upgrade migration..."
            flyway \
              -url="${DB_URL}" \
              -user="${DB_USER}" \
              -password="${DB_PASSWORD}" \
              -locations=filesystem:/migrations \
              migrate
        
        env:
        - name: DB_URL
          valueFrom:
            configMapKeyRef:
              name: {{ include "app.fullname" . }}-db-config
              key: url
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: {{ include "app.fullname" . }}-db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ include "app.fullname" . }}-db-secret
              key: password
        
        volumeMounts:
        - name: migrations
          mountPath: /migrations
      
      volumes:
      - name: migrations
        configMap:
          name: {{ include "app.fullname" . }}-migrations
      
      restartPolicy: Never

---

# helm/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
spec:
  template:
    spec:
      # 마이그레이션이 완료될 때까지 대기하는 로직은 없음
      # Helm Hook이 자동으로 순서를 보장함
      
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        # ...

---

# helm/values.yaml
migration:
  image: flyway/flyway
  tag: "9"

database:
  host: mysql.prod.svc.cluster.local
  port: 3306
  name: mydb
  user: migration_user
```

**Helm 배포 프로세스**:
```bash
helm upgrade --install my-app ./helm -f values-prod.yaml

# 실행 순서:
# 1. Pre-upgrade Hook 실행 (db-migration Job)
# 2. Job 완료 대기
# 3. Deployment 업데이트
# 4. 새 Pod 시작 (마이그레이션 이미 완료됨)
```

---

## 🔬 내부 동작 원리

### 1. Init Container의 실행 순서 및 보장

```
Pod 생성 요청
  ↓
1. 모든 Init Container 순차 실행
   Container 1: db-migrate
     - 실행
     - 완료 (exit 0) 또는 실패 (exit 1)
   
   Container 2: (있으면) app-setup
     - Container 1이 성공해야만 실행
   ↓
2. 모든 Init Container 성공
   ↓
3. App Container 시작
   (Main Container)
   ↓
4. Pod 상태: Running
```

**실패 처리**:
```
Init Container 실패 (exit 1)
  ↓
Pod 자동 재시작
  ↓
Init Container 다시 실행
  ↓
(최대 restartPolicy 횟수까지 재시도)

결과: Pod이 CrashLoopBackOff 상태
      (마이그레이션이 완료될 때까지 앱 실행 안 됨)
```

---

### 2. ConfigMap을 통한 마이그레이션 파일 관리

```yaml
# Option 1: 작은 파일 (<1MB)
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-migrations
data:
  V1__initial.sql: |
    CREATE TABLE users (...);
  V2__add_status.sql: |
    ALTER TABLE users ADD COLUMN status VARCHAR(50);

---

# Option 2: 큰 파일 (>1MB) → 별도 저장
# 문제: ConfigMap 최대 크기 1MB
# 해결: Kubernetes Secret 또는 Git 리포지토리에서 직접 다운로드
```

**마이그레이션 파일 로드 방식**:

```bash
# 방법 1: ConfigMap 마운트 (권장)
volumeMounts:
- name: migrations
  mountPath: /migrations

volumes:
- name: migrations
  configMap:
    name: db-migrations

# 결과: /migrations/V1__initial.sql, /migrations/V2__add_status.sql 등

---

# 방법 2: Git에서 직접 클론
initContainers:
- name: clone-migrations
  image: alpine/git
  command:
    - sh
    - -c
    - |
      git clone https://github.com/myorg/myapp.git /src
      cp /src/db/migrations/* /migrations/
  volumeMounts:
  - name: migrations
    mountPath: /migrations
```

---

### 3. Secret을 통한 민감 정보 보호

```yaml
# Secret 생성
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  # Base64 인코딩 (선택적, 자동으로 변환)
  DB_USER: migration_user
  DB_PASSWORD: "SecureP@ssw0rd123!"

---

# Secret 참조 (Pod에서)
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-credentials
      key: DB_USER

# Pod 내에서:
# echo $DB_USER → "migration_user"
# (Secret의 실제 값)
```

**보안 고려사항**:
```
Secret 저장소:
- etcd (기본): 암호화 권장
  kubectl apply -f secret.yaml  # Base64만 (암호화 아님)

- Sealed Secret (권장): 
  - Secret을 암호화하여 Git 저장 가능
  - install.sh로 자동 복호화

- HashiCorp Vault (더 강력):
  - 중앙 집중식 시크릿 관리
  - 감사 로그 기록
```

---

### 4. Job의 멱등성 보장

```
Job 실행
  ↓
Flyway validate: 마이그레이션 파일 checksum 확인
  ↓
Flyway migrate: 아직 적용되지 않은 마이그레이션만 실행
  (flyway_schema_history에 기록)
  ↓
Job 성공 (Pod 종료)

---

Job 다시 실행 (Pod 재시작)
  ↓
Flyway validate: 이미 적용된 마이그레이션은 건너뜀
  ↓
Flyway migrate: 신규 마이그레이션만 실행
  (중복 실행 없음)
  ↓
Job 성공 (Pod 종료)
```

**이를 보장하는 것**: `flyway_schema_history` 테이블
```sql
SELECT * FROM flyway_schema_history;

-- 예:
-- version | description | installed_by | installed_on
-- V1      | initial     | migration    | 2024-01-15 10:00:00
-- V2      | add_users   | migration    | 2024-01-15 10:00:05
```

Flyway는 이 테이블을 보고 **어떤 마이그레이션이 이미 적용됐는지** 판단합니다.

---

## 💻 실전 실험

### 실험 1: Init Container 배포

```bash
#!/bin/bash
# deploy-with-init-container.sh

set -e

# Step 1: Minikube 또는 실 클러스터에 연결
kubectl cluster-info

# Step 2: 네임스페이스 생성
kubectl create namespace app-prod --dry-run=client -o yaml | kubectl apply -f -

# Step 3: Secret 생성
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=migration_user \
  --from-literal=DB_PASSWORD='SecureP@ssw0rd123!' \
  -n app-prod \
  --dry-run=client -o yaml | kubectl apply -f -

# Step 4: ConfigMap 생성 (마이그레이션 파일)
cat > /tmp/V1__initial.sql << 'EOF'
CREATE TABLE IF NOT EXISTS users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(255) NOT NULL UNIQUE
);
EOF

kubectl create configmap db-migrations \
  --from-file=V1__initial.sql=/tmp/V1__initial.sql \
  -n app-prod \
  --dry-run=client -o yaml | kubectl apply -f -

# Step 5: Deployment 배포
kubectl apply -f deployment-init-container.yaml -n app-prod

# Step 6: Pod 상태 확인
echo "Waiting for Pod to start..."
kubectl wait --for=condition=ready pod \
  -l app=app \
  -n app-prod \
  --timeout=300s

# Step 7: Init Container 로그 확인
kubectl logs -l app=app -n app-prod --container db-migrate --all

# Step 8: App Container 로그 확인
kubectl logs -l app=app -n app-prod --container app

echo "✓ Deployment successful"
```

---

### 실험 2: Job 배포 및 순서 보장

```bash
#!/bin/bash
# deploy-with-job.sh

set -e

# Step 1: Job 배포
kubectl apply -f db-migration-job.yaml -n app-prod

# Step 2: Job 완료 대기
echo "Waiting for migration job..."
kubectl wait --for=condition=complete job/db-migration \
  -n app-prod \
  --timeout=300s

# Job 결과 확인
JOB_STATUS=$(kubectl get job db-migration -n app-prod -o jsonpath='{.status.succeeded}')
if [ "$JOB_STATUS" != "1" ]; then
  echo "❌ Migration job failed"
  kubectl logs -l app=migration -n app-prod
  exit 1
fi

echo "✓ Migration job completed"

# Step 3: 이제 안전하게 Deployment 배포
kubectl apply -f deployment.yaml -n app-prod

# Step 4: Deployment 롤아웃 대기
kubectl rollout status deployment/app -n app-prod --timeout=300s

echo "✓ Deployment successful"
```

---

### 실험 3: Helm Hook을 통한 자동 마이그레이션

```bash
#!/bin/bash
# deploy-with-helm.sh

set -e

# Step 1: Helm 차트 디렉토리 구조
# my-app-chart/
# ├── Chart.yaml
# ├── values.yaml
# ├── templates/
# │   ├── db-migration.yaml     (Hook: pre-upgrade)
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   └── configmap-migrations.yaml

# Step 2: Helm 레포지토리 업데이트 (선택사항)
helm repo update

# Step 3: 첫 설치
echo "Installing helm chart..."
helm install my-app ./my-app-chart \
  -n app-prod \
  -f values-prod.yaml \
  --create-namespace

# 결과:
# 1. Pre-install Hook (db-migration Job) 실행
# 2. Job 완료 대기
# 3. Deployment, Service 등 생성
# 4. Pod 시작

# Step 4: 상태 확인
kubectl get all -n app-prod

# Step 5: 새 버전으로 업그레이드
echo ""
echo "Upgrading to new version..."
helm upgrade my-app ./my-app-chart \
  -n app-prod \
  -f values-prod.yaml \
  --set image.tag=v1.2.1

# 결과:
# 1. Pre-upgrade Hook (새 db-migration Job) 실행
# 2. 새 마이그레이션 적용
# 3. Deployment 롤링 업데이트

# Step 6: 이전 Hook Job 자동 정리
echo ""
echo "Checking completed jobs..."
kubectl get jobs -n app-prod

# 이전 Job은 helm.sh/hook-delete-policy에 의해 자동 삭제됨

echo "✓ Helm deployment successful"
```

---

## 📊 성능/비용 비교

| 패턴 | 설정 복잡도 | 마이그레이션 시간 | 실패 처리 | 확장성 | 추천 상황 |
|------|-----------|------------------|---------|--------|----------|
| Init Container | 낮음 | ~5분 (Pod 시작 포함) | Pod 자동 재시작 | 낮음 | 소규모팀, 단순 구조 |
| Job 분리 | 중간 | ~2분 (병렬) | Job 재시도 | 중간 | 중간 규모 |
| Helm Hook | 낮음~중간 | ~2분 (자동) | Hook 실패 시 배포 차단 | 높음 | 엔터프라이즈 |

---

## ⚖️ 트레이드오프

### Init Container
**장점**: 단순함, Deployment 하나로 통합
**단점**: Pod 시작 느려짐, 확장 어려움

### Job 분리
**장점**: 독립적 실행, 실패 격리, 수동 제어 가능
**단점**: 순서 보장 필요 (별도 스크립트), 파이프라인 복잡

### Helm Hook
**장점**: 자동 순서 보장, Helm과 통합
**단점**: Helm 사용 필수, 학습곡선

---

## 📌 핵심 정리

1. **Init Container가 가장 단순** (시작 전 마이그레이션)
   - 마이그레이션 실패 시 Pod 시작 차단
   - ConfigMap으로 파일 관리
   - Secret으로 DB 정보 보호

2. **Job 분리가 가장 유연** (독립적 실행)
   - kubectl 또는 Helm으로 수동 제어 가능
   - 마이그레이션과 앱 배포 완전 분리
   - 재시도 정책 세밀 조절

3. **Helm Hook이 가장 자동화** (Pre-upgrade 자동)
   - `helm.sh/hook: pre-upgrade` 로 순서 자동 보장
   - Helm 버전과 함께 관리
   - 롤백 시 이전 버전의 마이그레이션 유지

4. **멱등성 자동 보장** (Flyway)
   - `flyway_schema_history` 로 중복 실행 방지
   - Pod 재시작해도 안전
   - 여러 인스턴스 동시 시작해도 Lock으로 순서 보장

5. **보안 필수** (Secret + ConfigMap)
   - DB 정보는 Secret
   - 마이그레이션 파일은 ConfigMap
   - Git에는 YAML만 커밋 (민감 정보 제외)

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Init Container에서 마이그레이션이 10분 걸린다면 Pod 시작이 10분 지연되지 않을까?</strong></summary>

**문제**:
```
Pod 생성 → Init Container 실행 (10분) → App Container 시작
↓
Kubernetes Startup Probe 타임아웃!

apiVersion: v1
kind: Pod
spec:
  startupProbe:
    httpGet:
      path: /health
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
    # 타임아웃: 30 * 10 = 300초 (5분) < 10분 (Init Container)
```

**결과**: Pod이 시작되기 전에 프로브 타임아웃 → CrashLoopBackOff

**해결 방법 1: Startup Probe 증가**
```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 120  # 120 * 10 = 1200초 (20분)
  periodSeconds: 10
```

**해결 방법 2: 마이그레이션을 Job으로 분리**
```
Job (5분) → Job 완료 → Deployment 시작 (30초)
(병렬: Job 중에 다른 Deployment 배포 가능)
```

**해결 방법 3: 대용량 마이그레이션은 사전 배포**
```
마이그레이션 예정 24시간 전 별도 Job으로 먼저 실행
(DB 스키마 미리 준비)
→ 실제 배포는 빠르게 진행
```

**교훈**: 10분 이상의 마이그레이션은 **Job 분리 또는 Helm Hook** 추천.
</details>

<details>
<summary><strong>Q2: ConfigMap에 마이그레이션 파일을 저장할 수 없다면 (최대 1MB)?</strong></summary>

**예**: 대용량 마이그레이션 파일 (10MB)

**해결 방법 1: 파일 분할**
```bash
# 원본 파일 (10MB)
V10__load_bulk_data.sql (10MB)

# 분할
split -b 512KB V10__load_bulk_data.sql V10__load_bulk_data.sql.

# 결과
V10__load_bulk_data.sql.aa (512KB) → ConfigMap 1
V10__load_bulk_data.sql.ab (512KB) → ConfigMap 2
...
V10__load_bulk_data.sql.t (512KB) → ConfigMap 20
```

**해결 방법 2: Git 리포지토리에서 직접 다운로드**
```yaml
initContainers:
- name: clone-migrations
  image: alpine/git:latest
  command:
    - sh
    - -c
    - |
      git clone --depth 1 --single-branch \
        --branch main \
        https://github.com/myorg/myapp.git /src
      cp /src/db/migrations/* /migrations/
  volumeMounts:
  - name: migrations
    mountPath: /migrations
```

**해결 방법 3: 외부 저장소 (S3, GCS)**
```yaml
initContainers:
- name: download-migrations
  image: amazon/aws-cli:latest
  command:
    - sh
    - -c
    - |
      aws s3 cp s3://my-bucket/migrations/ /migrations/ --recursive
      aws s3 cp s3://my-bucket/V10__load_bulk_data.sql /migrations/
  env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: access-key
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: secret-key
  volumeMounts:
  - name: migrations
    mountPath: /migrations
```

**교훈**: ConfigMap 초과 파일은 **Git 또는 S3에서 직접 로드**.
</details>

<details>
<summary><strong>Q3: 롤링 업데이트 중에 구 Pod과 신 Pod이 동시에 마이그레이션을 실행한다면?</strong></summary>

**상황**:
```
Deployment 롤링 업데이트 중 (maxSurge: 1)

Old Pod (v1.0 + 구 스키마): 마이그레이션 실행 안 함
  ↓
New Pod (v1.1 + Init Container에서 마이그레이션): 마이그레이션 실행
  ↓
New Pod의 Init Container가 V5, V6, V7 마이그레이션 실행
  ↓
구 Pod (v1.0): 새 스키마로 쿼리 시도
  ↓
V5 마이그레이션이 추가한 새 컬럼 → 구 앱이 이해 못함 → 에러!
```

**해결 방법: 스키마 하위 호환성**

스키마 변경은 **구 앱도 이해**해야 함:
```sql
-- V5__add_new_column.sql (안전)
ALTER TABLE users ADD COLUMN new_field VARCHAR(255) DEFAULT 'DEFAULT_VALUE';
-- 구 앱이 이 컬럼을 무시해도 상관없음

-- V5__rename_column.sql (위험)
ALTER TABLE users RENAME COLUMN old_field TO new_field;
-- 구 앱이 old_field를 SELECT → 에러!
```

**더 나은 방법: 앱 버전 먼저 배포**
```
Step 1: 앱 v1.1 배포 (신 스키마 대응)
        하지만 구 컬럼도 여전히 사용 가능하도록
        
Step 2: 모든 Pod이 v1.1로 업데이트됨

Step 3: 마이그레이션 실행 (스키마 변경)
        이제 모든 앱이 신 스키마 이해 가능
        
Step 4: 앱 v1.2 배포 (신 컬럼 활용)
```

**교훈**: 롤링 업데이트 중에는 **스키마 하위 호환성이 필수**, 또는 **Init Container를 비활성화하고 미리 Job으로 마이그레이션** 실행.
</details>

---

<div align="center">

**[⬅️ 이전: 마이그레이션 검증 자동화](./03-migration-validation.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 감사(Audit)와 컴플라이언스 ➡️](./05-migration-audit.md)**

</div>
