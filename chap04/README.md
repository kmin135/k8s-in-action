# 4장 레플리케이션과 그 밖의 컨트롤러: 관리되는 파드 배포

* 일반적으로 파드를 직접 생성할일은 거의 없음
* 대신 레플리케이션컨트롤러 또는 디플로이먼트 같은 리소스를 생성하여 파드를 생성하고 관리함

## 파드를 안정적으로 유지하기

* 기본적으로 파드가 노드에 스케줄링되면 해당 노드의 kubelet은 파드의 컨테이너를 실행하고 계속 실행되도록 유지함
* 컨테이너의 주 프로세스에 crash가 발생하면 kubelet이 컨테이너를 다시 시작함
* 그러나 OOM 처럼 치명적인 문제이긴 하지만 프로세스 자체는 살아있다면 kubelet이 이를 감지하지 못 함
* 또는 애플리케이션 내부의 무한 루프, 교착 상태 등이 발생하여 응답을 하지 않는 상황도 생각할 수 있음
* 이런 경우까지 상정하여 애플리케이션이 다시 시작되도록 하려면 애플리케이션 내부의 기능에 의존하지 말고 외부에서 애플리케이션 상태를 체크해야함

### 라이브니스 프로브

* 라이브니스 프로브 (liveness probe) 를 통해 컨테이너가 살아있는지 확인할 수 있음
* 파드의 `spec` 에 지정할 수 있으며 쿠버네티스는 주기적으로 프로브를 실행하여 프로브가 실패할 경우 컨테이너를 다시 시작함
* 쿠버네티스가 제공하는 라이브니스 프로브 메커니즘 3가지
  1. HTTP GET Probe : 지정한 url에 http get 요청을 보내고 응답 코드가 정상이면(2xx, 3xx) 이면 성공했다고 보고, 오류 응답이거나 응답이 없으면 실패로 간주함
  2. TCP Socket Probe : 지정된 포트에 TCP 연결을 시도하여 성공/실패를 확인
  3. Exec Probe : 컨테이너내에서 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인함. 상태 코드가 0이면 성공 그 외의 코드는 실패로 간주
* 라이브니스 프로브는 각 노드의 kubelet이 수행함. 마스터에서 실행 중인 쿠버네티스 컨트롤 플레인은 이 프로세스에 관여하지 않음.
  * 대신 노드 자체에 크래시가 발생하면 중단된 모든 파드의 대체 파드를 생성하는 것은 컨트롤 플레인의 역할임
  * 참고로 직접 생성한 파드는 각 노드의 kubelet에서만 관리는데 kubelet은 노드에서 실행되므로 노드 자체가 고장 나면 이런 파드들은 자동으로 복구되지 않음.

---

* HTTP 기반 라이브니스 프로브
* 예제로 사용한 `luksa/kubia-unhealthy` 는 5회부터의 요청에 500 리턴하는 예제용 이미지

```bash
k create -f kubia-liveness-probe.yaml

# 모니터링해보면 실행 후 약 1분30초 주기로 RESTARTS 값이 늘어난다
k get po kubia-liveness

# describe 해보면 컨테이너가 다시 시작된 이유를 볼 수 있음
k describe po kubia-liveness
```
```
# ...
Last State:     Terminated
      Reason:       Error
      Exit Code:    137
# ...
Restart Count:  1
    Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
# ...
Events:
  Warning  Unhealthy  3s (x6 over 2m13s)   kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    3s (x2 over 113s)    kubelet            Container kubia failed liveness probe, will be restarted
```

* 종료코드가 137인데 이는 프로세스가 외부 신호에 의해 종료되었음을 나타냄
* 137은 `128+x` 로 나온 값인데 x는 프로세스에 전송된 시그널 번호이고 이 시그널로 인해 컨테이너가 종료된 것임. 137이 나왔다는 것은 x가 SIGKILL(9) 이라는 거고 프로세스가 강제 종료되었음을 알 수 있음
  * 마찬가지로 143이면 SIGTERM(15) 을 받아 종료된 것임
* 컨테이너가 종료되면 완전히 새로운 컨테이너가 생성됨. 동일한 컨테이너가 다시 시작되는것이 아님.
* 다음은 컨테이너의 로그 관련 팁임
```bash
# logs는 기본적으로 현재 컨테이너의 로그를 표시함
k logs kubia-liveness
# --previous 옵션을 통해 이전 컨테이너의 로그를 볼 수 있음. 이전 컨테이너가 종료된 이유를 파악하는 경우 등의 상황에 유용함.
k logs kubia-liveness --previous
```

* 라이브니스의 추가 속성 설정

```bash
Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
```

* delay : 컨테이너가 시작된 후 몇 초후 프로브가 시작되는가
* timeout : 각 라이브니스 요청에 몇 초 안에 응답해야하는가. 여기서 지정한 시간 내에 응답이 안 오면 fail로 카운트
* period : 몇 초마다 프로브를 수행하는가
* failure : 이 값으로 지정한 횟수 만큼 연속 실패하면 컨테이너가 다시 시작됨

```yaml
# ...
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 15
```

* 이 중에서도 애플리케이션마다 컨테이너 구동 후 실제로 서비스 가능하기까지 걸리는 시간이 있으므로 delay는 적절히 설정해야함. 안 그러면 컨테이너가 제대로 구동되지도 못 하고 계속 재시작만 되는 상황이 생길 수 있음.

---

* 효과적인 라이브니스 구성팁
  * 대상 컨테이너 자체의 문제만 확인하게 구성. 예를들어 프론트서버 라이브니스가 DB연결까지 체크하는건 적절하지 않음. DB 자체의 문제라면 프론트서버를 재시작해봐야 의미가 없기 때문.
  * 라이브니스 체크 전용 경로를 구성하고 (ex. `/health`) 해당 경로는 인증이 필요하지 않도록 설정
  * 1초 이내에 완료되도록 가볍게 구성

## 레플리케이션컨트롤러(rc) 소개

* rc 는 파드가 항상 실행되도록 보장함
* 클러스터에서 노드가 사라지거나 노드에서 파드가 제거되면 rc가 이를 감지하고 교체 파드를 생성함
* rc는 파드의 여러 복제본(replica)을 작성하고 관리하기 위한 것

---

* rc의 동작
* 레이블 셀렉터를 활용해 매치되는 파드를 찾고 실제 파드 수가 의도하는 수와 일치하는지 항상 확인함
* 너무 적으면 파드 템플릿으로 새 복제본을 만들고 너무 많으면 실행 중인 초과 복제본을 제거함
* 파드 인스턴스는 다른 노드로 재배치되는 것이 아니고 완전히 새로운 파드 인스턴스를 새로 만드는 것임
* rc의 3가지 핵심요소
  * 레이블 셀렉터 : rc의 범위에 있는 파드를 결정
  * 레플리카 수 : 실행할 파드의 의도하는(desired) 수
  * 파드 템플릿 : 새로운 파드 레플리카를 만들 때 사용
* rc의 레이블 셀렉터, 파드 템플릿을 도중에 변경해도 기존 파드에는 영향이 없음
  * 레이블 셀렉터를 변경하면 기존 파드들이 rc의 관리 범위를 벗어나므로 영향이 없고
  * 파드 템플릿을 변경해도 변경 후의 새로 생성되는 파드에만 영향을 미치고 이미 만들어진 파드는 유지되기 때문임
* rc의 이점
  * 기존 파드가 사라지면 새 파드를 시작해 파드가 항상 실행되도록 함
  * 클러스터 노드에 장애가 발생하면 장애가 발생한 노드에서 실행중이던 모든 파드에 관한 교체 복제본이 생성됨
  * 수동 또는 자동으로 파드를 스케일링 하기 용이함

---

* [kubia-rc.yaml](kubia-rc.yaml)
* 템플릿의 파드 레이블은 rc의 레이블 셀렉터와 완전히 일치해야함
  * 아니면 새로운 파드를 가동시켜도 실제 복제본수가 의도하는 복제본 수와 일치하지 않으므로 새 파드를 무한정 만들 수 있음
  * 셀렉터를 지정하지 않으면 템플릿의 레이블을 자동으로 사용함. 오타도 방지하고 yaml도 짧아지므로 이 방법을 권장함.

```bash
# 3개의 파드가 생성됨을 볼 수 있음
k create -f kubia-rc.yaml

# 임의로 파드를 삭제하고 조회해보면 새로운 파드가 생성됨을 볼 수 있음
k delete po kubia-jc5x8

# rc 정보 확인
kubectl get rc

# rc 상세정보 확인
kubectl describe rc kubia
```

---

* 위 예제에서 파드가 삭제되니 새로운 파드가 생성되었는데 실제 동작방식은 파드의 삭제 자체에 대한 대응이 아닌 결과적인 상태(파드수가 부족함)에 대응한 결과임
* rc는 삭제되는 파드에 대해 즉시 통지를 받는데 이 통지 자체가 파드를 생성하게 하는 것은 아님. 이 통지는 rc가 실제 파드 수를 확인하고, 적절한 조치를 취하도록 하는 트리거 역할을 수행함.
  * 즉, 현재 파드수가 목표 파드수보다 모자라므로 새로운 파드를 생성할 뿐임
* 특정 노드에 장애가 발생했고 해당 노드에 rc가 관리하는 파드가 있었다면 결과적으로 rc는 현재 파드수가 부족함을 파악하고 새로운 파드를 다른 노드에 만들게됨. 이처럼 사람의 개입없이 자동으로 복구 처리가 됨.
```bash
# 특정 노드의 네트워크가 down 된 상황을 가정
# 대상 노드의 STATUS가 NotReady 로 표시됨 
kubectl get node

# 노드가 몇 분간 계속 NotReady 이면 해당 노드에 있던 파드들의 상태가 Unknown으로 변경되고 rc는 즉시 새 파드를 생성함
kubectl get pods

# 대상 노드가 다시 정상화되면 Unknown 상태로 변경되었던 파드는 삭제됨
```

### rc 범위 안팎으로 파드 이동하기

* rc가 관리하는 파드는 rc와 직접적으로 묶이지 않음
* rc는 단지 레이블 셀렉터와 일치하는 파드만을 관리할 뿐임
  * 단 파드는 `metadata.ownerReferences` 필드로 rc를 참조함. 특정 파드가 속한 rc를 찾는데 유용함.

```bash
# rc가 관리하는 레이블이 아니라면 추가해도 무관함
k label po kubia-tgcbq type=special

# rc가 관리하는 레이블을 변경하면 즉시 새로운 파드가 생성됨
k label po kubia-tgcbq app=kubia2 --overwrite
```
* 레이블이 변경되어 rc의 관리에서 벗어난 파드는 수동으로 생성한 파드와 동일함. 삭제도 직접 해줘야함.
* 운영중에 특정 파드에서 문제가 발생한 경우 해당 파드만 의도적으로 레이블을 변경하여 rc의 범위 밖으로 빼내고 문제 확인을 하는 용도로 활용할 수 있음
* 반대로 rc의 레이블 셀렉터를 변경하면 실제 파드수가 0개가 되므로 다시 목표 파드수만큼 파드를 생성할 것이고 기존에 만들어졌던 파드들은 수동 생성된 파드와 동일해짐
* rc는 레이블 셀렉터 변경이 가능하지만 후반의 다른 파드를 관리하는 리소스들은 그렇지 않음.

### 파드 템플릿 변경

* rc의 파드 템플릿을 변경하면 기존 파드에는 영향이 없고 변경 후에 새로 만들어지는 파드만 해당 템플릿으로 생성됨

```bash
# 실행중인 rc 편집하기
kubectl edit rc kubia
```

### 수평 파드 스케일링

```bash
# 명령어 사용 방식
kubectl scale rc kubia --replicas=5

# rc 정의를 편집하는 선언적인 방식
# 에디터에서 spec.replicas 항목을 변경
kubectl edit rc kubia
```
* 어느 방식이든 rc를 이용한 수평 파드 스케일링은 "x개의 인스턴스가 실행되게 하고 싶다"는 의도를 언급하는 것임.
* 구체적으로 무엇을 어떻게 하라고 말하는 게 아니라 의도하는 상태를 지정할 뿐임.

### rc 삭제

```bash
# rc를 삭제하면 관리받던 파드도 삭제됨
kubectl delete rc kubia

# 옵션을 통해 파드는 삭제되지 않도록 할 수 있음
# 물론 해당 파드들은 관리에서 벗어나게됨
kubectl delete rc kubia --cascade=false
```

* `--cascade=false` 옵션으로 rc만 삭제하고 레이블 셀렉터가 일치하는 새로운 rc 를 만들면 대상 파드들은 다시 관리받게 됨
  * 활용예제 : rc대신 레플리카셋으로 관리방식을 바꿀 때 무중단으로 변경이 가능함

## 레플리케이션컨트롤러(rc) 대신 레플리카셋(rs)으로 대체

* rs가 rc를 완전히 대체함
* 단, rs도 일반적으로는 직접 생성하기보다는 상위 수준의 디플로이먼트 리소스를 생성할 때 자동으로 생성됨.

### rc와 rs의 비교

* 기본 동작은 동일하나 rs가 더 풍부한 셀렉터 표현식을 제공함
* [kubia-replicaset.yaml](kubia-replicaset.yaml)
* [kubia-rc](kubia-rc.yaml)와는 셀럭터 정의 부분이 다름. rc는 selector 바로 아래에 오고 rs는 selector.matchLabels 에 지정함

### rs 사용법

* 만약 셀렉터와 일치하는 파드개수가 이미 만들어져 있었다면 새롭게 파드를 생성하지 않음.

```bash
# rs 상태 확인
kubectl get rs
kubectl describe rs
```
---

* API 버전 속성
* core API 그룹에 속하는 리소스는 apiVerison 필드가 필요없음. (ex. 파드)
* 반면 ReplicaSet 은 `apps/v1` 이라고 지정했는데 `apps`는 API 그룹, `v1` 은 API 버전을 의미함.

---

* 다양한 표현식을 지원하는 `matchExpressions` 를 지원함
```yaml
  selector:
    matchExpressions:
      - key: app
        operator: In
        values:
         - kubia
```
* In, NotIn, Exists, DoesNotExists 의 4가지 연산자를 제공
* 여러 표현식을 지정하면 모든 표현식이 true 여야 매칭됨
* matchLabels와 함께 사용할경우에도 모든 레이블, 표현식이 true 여야 매칭됨

```bash
# rs 삭제
kubectl delete rs kubia
```

## 4.4 데몬셋

* rc, rs 모두 클러스터 내 어딘가에 지정된 수만큼의 파드를 실행하는데 사용함
* 데몬셋은 클러스터 내 모든 노드에 노드당 1개씩만 파드가 실행되길 원할 때 사용함
* 시스템 수준의 작업을 수행하는 인프라 관련 파드에 유용함
  * 로그 수집기, 리소스 모니터링 등
* 데몬셋의 특성상 원하는 (DESIRED) 복제본 수라는 개념이 없음. 1개씩이니까.
* 노드가 다운되도 다른 곳에 파드를 생성하지 않음.
* 반면 새로운 노드가 추가되거나 누군가 실수로 데몬셋이 관리하는 파드를 삭제하면 즉각 데몬셋이 파드를 생성함
* 기본적으로 모든 클러스터 노드에 파드를 배포하지만 `node-Selector` 속성을 지정하여 파드를 배포할 노드의 그룹을 지정할 수 있음

---

* 예제 : ssd를 가지는 노드 그룹에 ssd-monitor 라는 데몬을 실행하고자 하는  경우 등
* [ssd-monitor-daemonset.yaml](ssd-monitor-daemonset.yaml)

```bash
k create -f ssd-monitor-daemonset.yaml

# disk=ssd 인 노드가 없을때는 파드가 생성되지 않음
k get ds

# disk=ssd 라벨을 지정해주면 대상 노드에 즉각 ssd-monitor 파드가 생성됨
k label node docker-desktop disk=ssd

# 라벨을 변경하면 해당 노드에서 제거됨
k label node docker-desktop disk=hdd --overwrite
```

## 4.5 잡 리소스

* rc, rs, ds 는 모두 계속 실행되야하는 파드를(웹 애플리케이션 등) 실행하는데 쓰임
* 반면 잡 리소스는 명시적으로 완료 가능한 프로세스를 한 번만 실행시키는데 사용됨
  * batch job 등

---

* 예제1 : [batch-job.yaml](batch-job.yaml)
* 2분간 sleep 후 종료되는 단순한 프로세스
* restartPolicy 는 OnFailure 또는 Never 로 명시적으로 설정해야함
  * OnFailure인 경우 실행중에 노드에 장애가 발생할 경우 다른 노드에서 실행됨.

```bash
k get jobs
# 책에는 완료된 pod는 -a 옵션을 줘야 보인다는데 해당 옵션은 없고 그냥 completed 상태로 잘 조회됨
k get po
# 완료된 pod 가 남아있으므로 로그를 조회하기 용이함
k logs batch-job-8vss
# job을 삭제하면 대상 pod도 제거됨
k delete job batch-job
```

-- 

* 예제2 : [multi batch job](multi-completion-batch-job.yaml)
* 예제1과 동일하되 completions에 지정한 만큼 순차적으로 실행함
* parallelism 에 지정한만큼 병렬로 실행할 수도 있음 (생략하면 1이라고 볼 수 있음)

```bash
# 책에서는 kubectl scale job 명령으로 job 실행 도중에 parallelism을 바꾸는데 이는 1.15부터 불가능한 방법임
# 대신 아래와 같이 edit 명령으로 parallelism 값을 수정하고 저장하면 실행 도중에 동시 수행 pod 개수를 변경할 수 있음
k edit job multi-completion-batch-job
```

---

* job pod의 최대 실행시간 (timeout) 설정이 필요할 경우 대상 파드 스펙에 `activeDeadlineSeconds` 속성을 설정할 수 있음
  * 파드가 설정값 보다 오래 실행되면 쿠버네티스가 종료시키고 job이 실패한 것으로 표시됨
* 추가로 job의 설정에 `spec.backoffLimit` 를 지정해 job을 실패한 것으로 표시하기 전에 재시도할 수 있는 횟수를 설정할 수 있음. 기본값 6.

## 4.6 주기적으로 Job 실행 (CronJob)

* 이름대로 Job에 cron을 적용한 방식
* cron 설정 외에는 Job과 유사
* `startingDeadlineSeconds` 옵션을 지정하면 지정한 초 이내에 파드가 실행되지 않으면 실패로 간주함. 파드가 예정보다 너무 늦게 시작되서는 안 될 경우에 유용함.

---

* 예제 : [cronjob](cronjob.yaml)
```bash
k get cronjob

NAME                              SCHEDULE             SUSPEND   ACTIVE   LAST SCHEDULE   AGE
batch-job-every-fifteen-minutes   0,15,30,45 * * * *   False     0        <none>          24s
```