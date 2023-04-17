# RBAC로 클러스터 보안

### RBAC

---

- RBAC은 인가 플러그인으로 많은 클러스터에서 기본적으로 활성화돼 있다.
- **API 서버 내에서 실행되는 RBAC와 같은 인가 플러그인은 클라이언트가 요청한 자원에서 요청한 동사를 수행할 수 있는지를 판별**한다.
- 사용자가 액션을 수행할 수 있는지 여부를 결정하는 핵심 요소로 **User Role**을 사용한다.
  - ex) user role에 secret 정보를 업데이트하는 권한이 없으면 API 서버는 사용자가 시크릿 정보에 대해 PUT 또는 PATCH 요청을 수행하지 못하게 한다.
- RBAC은 권한이 없는 사용자가 클러스터 상태를 보거나 수정하지 못하게 한다.
- default serviceAccount는 추가 권한을 부여하지 않는 한 클러스터 상태를 볼 수 없으며 어떤 식으로든 수정할 수 없다.
- 쿠버네티스 API 서버와 통신하는 애플리케이션을 작성하려면 RBAC 관련 리소스로 권한을 관리하는 방법을 이해해야 한다.

<br>

### RBAC 리소스

---

<aside>
💁‍♀️ RBAC 인가 규칙은 네 개의 리소스로 구성되며 두 개의 그룹으로 분류할 수 있다.
- **Role, ClusterRole** : 리소스에 수행할 수 있는 **동사**를 지정
- **RoleBinding, ClusterRoleBinding** : 위의 롤을 특정 **사용자, 그룹 또는 serviceAccount**에 바인딩
- 롤은 수행할 수 있는 작업을 정의하고, 바인딩은 누가 이를 수행할 수 있는지를 정의한다.

</aside>

<br>

**Role, RoleBinding** VS **ClusterRole, ClusterRoleBinding**

- Role과 RoleBinding은 네임스페이스가 지정된 리소스
- ClusterRole과 ClusterRoleBinding은 네임스페이스가 지정되지 않은 클러스터 수준의 리소스

<br>

### Role과 RoleBinding 사용하기

---

**1️⃣ Role 생성**

/role/pod-reader.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

Role 생성

```bash
kubectl create -f pod-reader.yaml

role.rbac.authorization.k8s.io/pod-reader created
```

<br>

2️⃣ **RoleBinding 생성 (serviceAccount에 Role Binding)**

주체(user, group, serviceAccount)에 Role을 Binding하는 것은 RoleBinding 리소스를 만들어 수행한다.

`kubectl create serviceaccount for-role` 로 for-role이라는 sa(주체)를 만든다.

“default” 네임스페이스의 “for-role” serviceAccount에 “pod-reader” Role이 부여하기

```bash
kubectl create rolebinding test --role=pod-reader --serviceaccount=default:for-role
```

```bash
kubectl get rolebinding test -n default -o yaml

# OUTPUT
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2023-04-17T05:31:28Z"
  name: test
  namespace: default
  resourceVersion: "10078241"
  uid: 64dcd1cd-45ea-4c8a-a9f3-1939263c1497
# pod-reader 라는 롤을 참조하는 test 롤바인딩
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
# default 네임스페이스에 있는 for-role 서비스어카운트에 바인드
subjects:
- kind: ServiceAccount
  name: for-role
  namespace: default
```

→ **.subjects 목록에 다른 네임스페이스에 있는 serviceAccount를 추가하면, 비록 다른 네임스페이스에 있는 serviceAccount이지만 default 네임스페이스의 롤을 수행할 수 있게 할 수 있다.**

<br>

`auth can-i`를 사용하여 `for-role`서비스 계정이 `default`네임스페이스에서 `pods`리소스를 `get`할 수 있는지 확인하기

```bash
kubectl auth can-i get pods --namespace=default --as=system:serviceaccount:default:for-role
```

→ 결과는 yes or no로 확인할 수 있다.

<br>

### ClusterRole과 ClusterRoleBinding 사용하기

<aside>
❓ **ClusterRole과 ClusterRoleBinding이 왜 필요한가?**
- 일반 Role은 Role이 위치하고 있는 동일한 네임스페이스의 리소스에만 액세스할 수 있다. 다른 네임스페이스의 리소스에서 누군가가 액세스할 수 있게하려면 해당 네임스페이스마다 롤과 롤바인딩을 만들어야 한다. 
- 이때, ClusterRole과 ClusterRoleBinding을 사용함으로써 네임스페이스가 지정되지 않은 리소스나 리소스가 아닌 URL에 액세스를 허용하는 클러스터 수준의 리소스로 각각의 네임스페이스에 동일 Role을 재정의할 필요 없이 개별 네임스페이스에 Bind해서 공통적인 Role로 사용할 수 있다.

</aside>

<br>

1️⃣ **ClusterRole 생성**

/role/secret-reader.yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
  - apiGroups: [""]
    #
    # at the HTTP level, the name of the resource for accessing Secret
    # objects is "secrets"
    resources: ["secrets"]
    verbs: ["get", "watch", "list"]
```

ClusterRole 생성

```bash
kubectl create -f secret-reader.yaml

clusterrole.rbac.authorization.k8s.io/secret-reader created

```

<br>

**2️⃣ ClusterRoleBinding**

```bash
kubectl create clusterrolebinding test --clusterrole=secret-reader

clusterrolebinding.rbac.authorization.k8s.io/test created
```

```bash
kubectl get clusterrolebinding test -n default -o yaml

# OUTPUT
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2023-04-17T05:56:43Z"
  name: test
  resourceVersion: "10084286"
  uid: 2a4463f8-37ea-48d5-be1e-f8d163b1ade3
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: secret-reader
```
