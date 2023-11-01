수행 내용

- (Gitlab에 yaml 저장) ArgoCD 통해 EKS에 Pod 배포
- Pod에 들어갈 값 : Vault KV 저장 ⇒ Argocd의 plugin으로 가져오는 방식.

## Vault 구성

- Pod에서 사용할 값 : Vault KV 저장

```bash
# 1. KV secrets engine
vault secrets enable -ns=uplus -path=kvv2 -version=2 kv

# 1-1. KV secrets engine 확인
vault secrets list -ns=uplus

Path             Type            Accessor                 Description
----             ----            --------                 -----------
...
kvv2/            kv              kv_01d15230              n/a

# 2. KV secrets 저장
vault kv put -ns=uplus kvv2/myapp/config \
username="appuser" \
password="suP3rsec(et"

## 2-1. kv-v2/myapp/config 값 확인
vault kv get -ns=uplus kvv2/myapp/config
```

```bash
# 3. Policy 생성
vault policy write -ns=uplus policy-kv2-myapp-config -<<EOF
path "kvv2/data/myapp/config" {
 capabilities=["read"]
}
EOF

# 3.1 생성 확인
# >> VAULT UI 확인 : KV & Policy
```

```bash
# 4. Entity 생성
vault write -ns=uplus -format=json identity/entity \
name=argocd-user \
policies=policy-kv2-myapp-config \
| jq -r ".data.id" > entity_id.txt

## 4-1. Entity 생성 확인
vault read -ns=uplus identity/entity/name/argod-user

# 5. Entity Alias 생성
## 5-1. Token Auth 조회
vault auth list -ns=uplus -format=json | jq -r '.["token/"].accessor' > auth_token_accessor.txt

## 5-2. Entity-Alias 생성 
vault write -ns=uplus identity/entity-alias \
name=argocd-user \
canonical_id=$(cat entity_id.txt) \
mount_accessor=$(cat auth_token_accessor.txt)

# 5-3. 생성 확인
# >> VAULT UI 확인 : Entity & Alias
```

```bash
# 6. Token Role 구성
vault write -ns=uplus auth/token/roles/role-argocd-user \
allowed_entity_aliases=argocd-user \
orphan=true \
token_period=10d \
token_bound_cidrs="43.202.165.54,13.125.181.221"

## 6-1. Token Role 확인 
vault read -ns=uplus auth/token/roles/role-argocd-user

# 7. Token 발급
vault token create -ns=uplus -role=role-argocd-user -entity-alias=argocd-user 

## 7.1 확인
vault token lookup <발급받은 Token 입력>
```

⇒ pod에 들어갈 값  kv & kv 읽을 수 있는 Token 생성 완료

---

## GitLab 구성

- **Vault의 Token 인증에 필요한 값들** → Gitlab에 저장 → Arcocd Application 이용해 관리

```bash
## Projects: Administrator/vault_argocd
## Folder: vault-creds
# 1. Crenedtial 저장
argocd-vault-plugin-credentials.yaml

apiVersion: v1
**kind: Secret
metadata:
  name: argocd-vault-plugin-credentials**
  **namespace: argocd**
stringData:
  **VAULT_ADDR**: "https://uplus.aws-vault.com"
  **VAULT_NAMESPACE**: "uplus"
  **VAULT_TOKEN**: "hvs.CAESIMJVPLCKDZPw7iQp3FU1S6on7PZMUCzfiS7S7bX7glmHGicKImh2cy5EeTBQYXV4N0wycEtNdkJUaUVoaVI3cGsuejNQbmkQ3x4"
  AVP_TYPE: "vault"
  AVP_AUTH_TYPE: "token"
type: Opaque
```

---

## Argocd  구성

```bash
# 1. K8s Namespace
kubectl create namespace argocd
kubectl get namespace

# 2. Argocd 설치 (Stable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## Argocd UI 접속 정보
# https://43.202.165.54:31010/login?return_url=https%3A%2F%2F43.202.165.54%3A31010%2Fapplications

# 3. admin Password 조회
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
# admin
M9XcepSo5YP8ZFQL
```

```bash
# 4. Argocd 설치 확인
k9s
:namespace
# argocd : Deployments 확인 가능
****┌─────────────────────────────────────────── Deployments(argocd)[6] ───────────────────────────────────────────┐
│ NAME↑                                             READY          UP-TO-DATE          AVAILABLE AGE           │
│ argocd-applicationset-controller                    1/1                   1                  1 6d            │
│ argocd-dex-server                                   1/1                   1                  1 6d            │
│ argocd-notifications-controller                     1/1                   1                  1 6d            │
│ argocd-redis                                        1/1                   1                  1 6d            │
│ argocd-repo-server                                  1/1                   1                  1 6d            │
│ argocd-server                                       1/1                   1                  1 6d            │
│                                                                                                              │
```

### [Settings] > [Repository] > [CONNECT REPO]
![Untitled](https://github.com/jslim1995/insideinfo-test/assets/100335118/3bc0b4cc-776f-402c-8860-7744924077af)
![Untitled 1](https://github.com/jslim1995/insideinfo-test/assets/100335118/002f3304-abef-40f2-aaaf-96dc69bcfa6a)

### Application : vault-creds 생성
![Untitled 2](https://github.com/jslim1995/insideinfo-test/assets/100335118/0bdfc2a8-da22-494e-8776-c54d7370637f)
![Untitled 3](https://github.com/jslim1995/insideinfo-test/assets/100335118/dc882a96-886e-4401-8d6d-7aee8ef8102d)
![Untitled 4](https://github.com/jslim1995/insideinfo-test/assets/100335118/b48a17db-f7da-4509-a3ed-415ee366cf27)

- (argocd) Application
![Untitled 5](https://github.com/jslim1995/insideinfo-test/assets/100335118/24182c5d-f2a7-403a-8f1f-11177a4ef4f7)


- **EKS : Secret ⇒** (argocd) argocd-vault-plugin-credentials
![Untitled 6](https://github.com/jslim1995/insideinfo-test/assets/100335118/0cac4f8d-a431-41a3-910f-914030404d9b)


```bash
## [참고] yaml 사용 방법
# 1. GitLab : vault-creds.yaml
#    **-→ [Argocd Application] vault-creds** **생성**
cat >> argocd-vault-creds.yaml -<<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault-creds
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: 'https://kubernetes.default.svc'
  source:
    path: ./vault-creds
    repoURL: 'http://52.78.99.206:8100/root/vault_argocd.git'
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
EOF

## 1-1. [Argocd Application] ****vault-creds 배포
kubectl apply -f argocd-vault-creds.yaml

## 1-2. Argocd Application 배포 확인
kubectl get applications.argoproj.io -n argocd

NAME          SYNC STATUS   HEALTH STATUS
vault-creds   Synced        Healthy

```

```bash
# 2. 배포된 EKS Secrets 확인 : argocd-vault-plugin-credentials 
kubectl get secrets -n argocd

NAME                              TYPE     DATA   AGE
argocd-vault-plugin-credentials   Opaque   5      3m21s

# 2-1. 값 확인
kubectl describe -n argocd secret argocd-vault-plugin-credentials
kubectl get secret argocd-vault-plugin-credentials -o jsonpath="{.data.VAULT_NAMESPACE}" -n argocd | base64 -d; echo
kubectl get secret argocd-vault-plugin-credentials -o jsonpath="{.data.VAULT_TOKEN}" -n argocd | base64 -d; echo
```

### SA 권한 추가 - credentials get

```bash
cd argocd/
vi argocd-repo-server-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
**kind: Role**
metadata:
  **name: argocd-repo-server**
  namespace: argocd
rules:
- apiGroups:
  - ""
  **resourceNames:
  - argocd-vault-plugin-credentials
  resources:
  - secrets
  verbs:
  - get
  - watch**
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocd-repo-server
  namespace: argocd
**subjects:**
- kind: ServiceAccount
  **name: argocd-repo-server**
  namespace: argocd
**roleRef:**
  kind: Role
  **name: argocd-repo-server**
  apiGroup: rbac.authorization.k8s.io
```

### Configmap : Arogcd-Vault-Plugin 생성

```bash
# 3. Plugin 작성 (sidecar 방식)
vi cmp-plugin.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: **cmp-plugin**
  namespace: argocd
data:
  **argocd-vault-plugin.yaml:** |
    apiVersion: argoproj.io/v1alpha1
    kind: ConfigManagementPlugin
    metadata:
      name: argocd-vault-plugin
    spec:
      allowConcurrency: true
      discover:
        find:
          command:
            - sh
            - "-c"
            - "find . -name '*.yaml' | xargs -I {} grep \"<path\\|avp\\.kubernetes\\.io\" {} | grep ."
      generate:
        command:
          - argocd-vault-plugin
          - generate
          - "-s"
          - **"argocd-vault-plugin-credentials"**
          - "."
      lockRepo: false

## 3-1. Plugin 생성
kubectl apply -f cmp-plugin.yaml

## 3-2. 확인
kubectl get configmap -n argocd
kubectl describe configmap -n argocd cmp-plugin
```
![Untitled 7](https://github.com/jslim1995/insideinfo-test/assets/100335118/b344241d-7c41-451b-83d4-becdff677c6d)


### (Sidecar) Argicd-Repo-Server : Arogcd-Vault-Plugin

```bash
## k9s 확인
┌─────────────────────────────────────────────── Containers(argocd/argocd-repo-server-5457fff8c9-knr2s)[4] ────────────────────────────────────────────────┐
│ NAME↑                 PF  IMAGE                             READY  STATE       INIT     RESTARTS PROBES(L:R)    CPU/R:L   MEM/R:L PORTS        AGE       │
│ argocd-repo-server    ●   quay.io/argoproj/argocd:v2.8.4    true   Running     false           0 on:on              0:0       0:0 8081,8084    4d23h     │
**│ argocd-vault-plugin   ●   alpine                            true   Running     false           0 off:off            0:0       0:0              4d23h     │**
│ copyutil              ●   quay.io/argoproj/argocd:v2.8.4    true   Completed   true            0 off:off            0:0       0:0              4d23h     │
│ download-tools        ●   registry.access.redhat.com/ubi8   true   Completed   true            0 off:off            0:0       0:0              4d23h     │
```

```bash
# 4. Argocd repo server에 Plugin (sidecar) 적용
kubectl edit -n argocd **deployment argocd-repo-server**

apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
		**# (1) automountServiceAccountToken : false → true**
      **automountServiceAccountToken**: true

		**# (2) Volumes 추가 : cmp-plugins, custom-tools**
      **volumes**:
      - **configMap**:
          name: cmp-plugin
        name: cmp-plugin
      - name: custom-tools
        **emptyDir: {}**

		**# (3) initContainers 수정 : argocd-vault-plugin 설치**
      **initContainers**:
      - name: download-tools
        **image: registry.access.redhat.com/ubi8**
        env:
          - name: AVP_VERSION
            value: 1.14.0
        command: [sh, -c]
        args:
          - >-
            curl -L https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v$(AVP_VERSION)/argocd-vault-plugin_$(AVP_VERSION)_linux_amd64 -o argocd-vault-plugin &&
            chmod +x argocd-vault-plugin &&
            mv argocd-vault-plugin /custom-tools/
        volumeMounts:
          - mountPath: /custom-tools
            name: custom-tools

		**# (4) containers : sidecar container 추가**
			**containers**:
			 **- name: argocd-vault-plugin**
			        command: [/var/run/argocd/argocd-cmp-server]
			        **image: alpine**
			        securityContext:
			          runAsNonRoot: true
			          runAsUser: 999
			        volumeMounts:
			          - mountPath: /var/run/argocd
			            name: var-files
			          - mountPath: /home/argocd/cmp-server/plugins
			            name: plugins
			          - mountPath: /tmp
			            name: tmp
			
			          # Register plugins into sidecar
			          **- mountPath: /home/argocd/cmp-server/config/plugin.yaml
			            subPath: argocd-vault-plugin.yaml
			            name: cmp-plugin**
			
			          # Important: Mount tools into $PATH
			          - name: custom-tools
			            subPath: argocd-vault-plugin
			            mountPath: /usr/local/bin/argocd-vault-plugin
```

```bash
# 적용된 container 확인
containers:
      - command:
        - /var/run/argocd/argocd-cmp-server
        **image: alpine**
        imagePullPolicy: Always
        **name: argocd-vault-plugin**
        resources: {}
        securityContext:
          runAsNonRoot: true
          runAsUser: 999
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
        - mountPath: /tmp
          name: tmp
        **- mountPath: /home/argocd/cmp-server/config/plugin.yaml
          name: cmp-plugin
          subPath: argocd-vault-plugin.yaml**
        - mountPath: /usr/local/bin/argocd-vault-plugin
          name: custom-tools
          subPath: argocd-vault-plugin
```

```jsx
## (참고) Container 내, plugin 파일 확인
kubectl exec -n argocd -it argocd-repo-server-5457fff8c9-knr2s -c argocd-vault-plugin -- cat /home/argocd/cmp-server/config/plugin.yaml
```

## GitLab 구성

```bash
## Projects: Administrator/vault_argocd
## Folder: pod
# 2. Pod.YAML 생성
vault-kv2-pod.yaml
****
apiVersion: v1
**kind: Pod
metadata:
  name: vault-kv2-pod
  namespace: default**
spec:
  serviceAccountName: default
  containers:
    - name: vault-kv2
      image: alpine
      command: ["sleep","infinity"]
      **env:
        - name: Username
          value: <path:kvv2/data/myapp/config#username>
        - name: Password
          value: <path:kvv2/data/myapp/config#password>**
```

### Application : vault-myapp 생성

```bash
# 5. GitLab : vault-kv2-pod.yaml
#    **-→ [Argocd Application]** vault-kv2-pod **생성**
vi argocd-vault-kv2-pod.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: vault-myapp
  namespace: argocd
spec:
  destination:
    namespace: default
    server: 'https://kubernetes.default.svc’  
  source:
    path: ./MyApp
    repoURL: 'https://<gitlab repo>.git'
    targetRevision: HEAD
    **plugin:
      name: argocd-vault-plugin**
  project: default
  syncPolicy:
    automated:
      selfHeal: true
      prune: true

## 5-1. [Argocd Application] ****
kubectl apply -f argocd-myapp.yaml

kubectl get applications.argoproj.io -n argocd
```

```bash
# 6. 배포된 Pod 확인
kubectl get pod

# 7. Pod에서 Vault 값 확인
**kubectl exec -it vault-kv2-pod -- env**
```
![Untitled 8](https://github.com/jslim1995/insideinfo-test/assets/100335118/e1520faa-9e1d-453c-a6ad-b4d2e6f379f7)

