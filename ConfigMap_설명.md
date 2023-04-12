# ConfigMap

### ConfigMap

---

- 설정 데이터를 저장하는 쿠버네티스 오브젝트
- 키-값 쌍
- 맵의 내용은 컨테이너의 **환경변수** 또는 **컨피그맵 볼륨 파일**로 전달되어 파드는 이 둘을 통해 컨피그맵을 사용할 수 있다.
- 동일한 이름으로 컨피그맵에 관한 여러 매니페스트를 유지할 수 있다. → 모든 환경에서 동일한 파드 정의를 사용해 각 환경에서 서로 다른 설정을 사용할 수 있다.
- Pod와 ConfigMap은 같은 namespace에 있어야 한다.

<br>

### Using ConfigMap as Environment variables and files

---

/configmap/game-demo.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	# ConfigMap's name
  name: game-demo
# data block for setting information
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys; value of file form (file name, file contents)
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

<br>

/pod/configmap-demo-pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
          # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      # Mount the volume below
      volumeMounts:
        - name: config
          mountPath: "/config"
          readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
          - key: "game.properties"
            path: "game.properties"
          - key: "user-interface.properties"
            path: "user-interface.properties"
```

→ 환경변수를 정의하는 `env` 블록에서는 `PLAYTER_INITIAL_LIVES` 와 `UI_PROPERTIES_FILE_NAME` 이라는 두개의 환경변수를 정의했고, `game-demo` ConfigMap에서 값을 가져옴을 나타내는 `configMapKeyRef` 필드를 사용함을 확인할 수 있다.

→ `items` 필드를 사용하여 ConfigMap에서 가져온 파일 형태의 설정 정보를 볼륨 안에 파일로 생성하며, `volumeMounts` 블록을 사용하여 컨테이너 안에서 해당 볼륨을 마운트 한다.

→ ConfigMap에 4개의 키가 있음에도 불구하고 /config 아래에 /config/game.properties와 config/user-interface.properties 두 파일이 생성된다. 만약 items 필드를 생략했을 모든 키가 키와 동일한 이름을 가진 파일들이 생성되어 이 예제 파드의 경우 4개의 파일을 생성하게 된다.

ConfigMap을 사용하는 가장 흔한 방법은 같은 namespace에 있는 pod안에 실행중인 컨테이너들을 위한 세팅을 구성하는 것이다. → ConfigMap은 분배되어 사용될 수 있다.

⇒ 여러 컨테이너들은 각 자신의 `volumeMounts` block 뿐만 아니라 `.spec.volumes`또한 다 필요하다.

<br>

### Mounted CongifMaps are updated automatically

---

- kubelet은 주기적으로 동기화 할때마다 mounted ConfigMap이 Fresh한지 체크한다.
- **→ 마운트된 컨피그맵은 자동으로 업데이트 된다.**
- ConfigMap이 업데이트되는 순간부터 새 키가 파드에 투영되는 순간까지의 총 지연 = kubelet 동기화 기간 + 캐시 전파 지연(kubelet은 로컬캐시를 사용)

⚠️ 환경변수로 사용되는 ConfigMap은 자동으로 업데이트되지 않으며, 파드를 재시작 해야한다.

<br>

### Immutable ConfigMap

---

- 데이터 변경을 방지할때의 이점
  - 애플리케이션 중단을 일으킬 수 있는 우발적(원치않는) 업데이트로부터 보호
  - ConfigMap 모니터링을 중지하여 kube-api server 부하를 크게 줄여 클러스터 성능 향상

```yaml
apiVersion: v1
kind: ConfigMap
metadata: ...
data: ...
immutable: true
```

→ 변경사항을 되돌리거나 data 또는 binaryData 필드 내용을 변경할 수 없다.

📌 변경하기 위해서는 컨피그맵을 삭제하고 다시 작성하는 수밖엔 없는데, Pod는 마운트 포인트를 유지하기 때문해 해당 파드도 삭제한 뒤 다시 만드는 것이 좋다.

<br>

### ConfigMap 생성 방법

---

1. `—-from-literal`

   ```bash
   # sleep-interval=25라는 단일 항목을 가진 fortune-config configmap을 생성
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml create configmap fortune-config --from-literal=sleep-interval=25
   # OUTPUT
   configmap/fortune-config created

   # YAML 정의 출력
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml get configmap fortune-config -o yaml
   # OUTPUT
   apiVersion: v1
   data:
     sleep-interval: "25"
   kind: ConfigMap
   metadata:
     creationTimestamp: "2023-04-12T05:11:30Z"
     name: fortune-config
     namespace: default
     resourceVersion: "8352726"
     uid: 214b72c9-8d6e-4e66-9d54-52405426212f
   ```

2. `—-from-file`

   - sample configuration setting / download
     ```bash
     mkdir -p configure-pod-container/configmap/

     # Download the sample files into `configure-pod-container/configmap/` directory
     wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
     wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties
     ```

   2-1. 디렉터리에 있는 configuration 파일들로부터 configmap 생성

   ```bash
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml create configmap game-config --from-file=configure-pod-container/configmap
   ```

   ```bash
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml get configmap game-config -o yaml
   # OUTPUT
   apiVersion: v1
   data:
     game.properties: |-
       enemies=aliens
       lives=3
       enemies.cheat=true
       enemies.cheat.level=noGoodRotten
       secret.code.passphrase=UUDDLRLRBABAS
       secret.code.allowed=true
       secret.code.lives=30
     ui.properties: |
       color.good=purple
       color.bad=yellow
       allow.textmode=true
       how.nice.to.look=fairlyNice
   kind: ConfigMap
   metadata:
     creationTimestamp: "2023-04-12T05:24:41Z"
     name: game-config
     namespace: default
     resourceVersion: "8355865"
     uid: 352a3373-b991-4d49-80e9-801ea9eb3544
   ```

   2-2. 특정 파일로부터 configmap 생성

   ```bash
   kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties
   ```

   ```bash
   kubectl --kubeconfig kubeconfig-lena-cluster.yaml get configmap game-config-2 -o yaml
   # OUTPUT
   apiVersion: v1
   data:
     game.properties: |-
       enemies=aliens
       lives=3
       enemies.cheat=true
       enemies.cheat.level=noGoodRotten
       secret.code.passphrase=UUDDLRLRBABAS
       secret.code.allowed=true
       secret.code.lives=30
   kind: ConfigMap
   metadata:
     creationTimestamp: "2023-04-12T05:30:08Z"
     name: game-config-2
     namespace: default
     resourceVersion: "8357166"
     uid: dd9b415a-1cc0-419d-8ddf-d4b0fe2dd2aa
   ```
