# 5장 서비스: 클라이언트가 파드를 검색하고 통신할 수 있게 함

* VM 기반의 시스템에서는 접근하고자 하는 서비스의 정확한 서비스 IP나 호스트명으로 구성하는 경우가 많음
* 하지만 k8s 에서 파드는 일시적이라 언제든지 늘어나거나 제거될 수 있으며 IP 주소도 생성시점에 결정되므로 미리 IP를 알 수도 없음

## 5.1 서비스 소개

* k8s의 서비스는 동일한 서비스를 제공하는 파드 그룹에 지속적인 단일 접점을 제공하는 리소스
  * 서비스는 존재하는 동안 불변인 IP 주소와 포트를 가짐
* 서비스를 이용하면 클라이언트는 실제 서비스를 제공하는 개별파드위치는 알 필요가 없으므로 대상 파드들은 자유롭게 클러스터안에 배치될 수 있음
* P207의 그림5.1은 서비스를 활용한 구조를 보여줌
  * 외부 클라이언트는 프론트엔드 서비스 주소만 알면되고
  * 프론트엔드는 백엔드의 서비스 주소만 알면됨

---

* 서비스 생성 기본 : [kubia-svc.yaml](kubia-svc.yaml)
```bash
# 아직은 클러스터 IP만 존재하므로 클러스터 내부에서만 접근 가능함
k get svc

# 테스트를 위해 클러스터내의 파드를 한 개 고르고
k get po

# docker 처럼 exec 명령을 통해 클러스터내의 파드에서 생성한 서비스의 클러스터IP로 테스트할 수 있음
# 더블대시(--)는 kubectl 명령줄 옵션의 끝을 의미함. 생략하면 curl의 옵션 -s를 kubectl exec의 옵션으로 해석해버림
k exec kubia-nt8rq -- curl -s http://10.98.57.206
```
* 기본 포워드 정책은 RR이라 요청을 할 때마다 랜덤으로 파드가 선택됨
* `spec.sessionAffinity`값을 `ClientIP` 로 설정하면 동일 클라이언트 IP의 모든 요청을 동일 파드로 전달함
  * 기본값은 `None` 임
  * 서비스는 `TCP/UDP` 기반이므로 쿠키 기반 세션 어피니티는 지원하지 않음
* `spec.ports` 아래에 복수의 포트 포워딩 정책을 정의할 수 있음
  * 단, 레이블 셀렉터는 항상 서비스 전체에 적용되며 포트마다 개별적으로 설정할 수 없음.
```yaml
# ...
spec:
  ports:
  - name: http # 서비스의 80은 파드의 8080 으로
    port: 80
    targetPort: 8080
  - name: https # 서비스의 443은 파드의 8443 으로
    port: 443
    targetPort: 8443
# ...
```
* 파드 정의에 포트 이름을 정의한 경우(named port), 서비스에도 해당 포트 이름으로 포워딩 정책을 명시할 수 있음
```yaml
kind: Pod
spec:
  containers:
  - name: kubia
    ports:
    - name: http # 8080 포트를 http라 정의
      containerPort: 8080
# ...
```
```yaml
kind: Service
spec:
  ports:
  - name: http # 서비스의 포트 80은 파드에서 http 라 명명한 포트로 포워드한다.
    port: 80
    targetPort: http
# ...
```
* 이를 활용하면 추후 파드의 포트가 바뀌더라도 서비스의 정의를 변경하지 않고도 적용할 수 있는 장점이 있음

---

* 클라이언트 파드들은 서비스의 IP와 포트를 어떻게 찾아야 하나?
* 서비스 생성 후 만들어지는 파드내 환경변수로 만들어짐
```bash
k exec kubia-5h7q7 -- env

# ...
KUBIA_SERVICE_HOST=10.98.57.206
# ...
```
* env의 변수명 정책은 서비스 이름의 대시(-)는 밑줄(_)로 변경되고 서비스 이름이 환경변수명의 접두어가 되고 모든 문자는 대문자로 표시됨
* 하지만 서비스 생성전에 만들어진 파드에서는 조회되지 않음

---

* 환경변수대신 k8s에 내장된 DNS 서버를 활용할 수 있음
  * 파드가 내부 DNS 서버를 활용할지는 각 파드 스펙의 `dnsPolicy` 로 구성하는 것도 가능
* 클라이언트 파드는 내부 DNS를 활용하면 환경변수대신 FQDN(정규화된 도메인 이름)으로 서비스에 액세스할 수 있음
* 서비스명이 `kubia` 라면 기본 FQDN은 `kubia.default.svc.cluster.local` 이 됨 
  * kubia : 서비스명
  * default : 서비스가 정의된 네임스페이스
  * `svc.cluster.local` : 모든 클러스터의 로컬 서비스 이름에 사용되는 클러스터의 도메인 접미사
* FQDN으로 서비스에 연결할 때 같은 네임스페이스에 있다면 네임스페이스, `svc.cluster.local` 접미사 생략 가능
```bash
# 전체 FQDN
k exec kubia-5h7q7 -- curl -s http://kubia.default.svc.cluster.local
# 접미사 생략
k exec kubia-5h7q7 -- curl -s http://kubia.default
# 동일 네임스페이스이면 네임스페이스 생략
k exec kubia-5h7q7 -- curl -s http://kubia

# 각 파드의 /etc/resolv.conf 를 보면 원리를 알 수 있다.
k exec kubia-5h7q7 -- cat /etc/resolv.conf
```
```bash
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

* 추가로 서비스 IP에는 ping이 안 되는데 이는 서비스의 클러스터IP가 가상 IP이므로 서비스 포트와 결합된 경우에만 의미를 가지기 때문이다.

## 5.2 클러스터 외부에 있는 서비스 연결

* 클러스터 외부의 다른 서버들에 대한 서비스도 만들 수 있음
* 먼저 앞서 살펴본 서비스는 사실 파드에 직접 연결되는 것이 아니고 엔드포인트 리소스가 그 사이에 있음
```bash
# 서비스의 속성값에 Endpoints 가 있고
k describe svc kubia
```
```text
# ...
TargetPort:        8080/TCP
Endpoints:         10.1.3.26:8080,10.1.3.27:8080,10.1.3.28:8080
```
```bash
# 아래처럼 직접 조회도 가능
k get endpoints kubia
```
```text
NAME    ENDPOINTS                                      AGE
kubia   10.1.3.26:8080,10.1.3.27:8080,10.1.3.28:8080   6d20h
```

* 클라이언트가 서비스에 연결하면 서비스 프록시는 endpoints 중 하나의 IP와 포트 쌍을 선택하고 들어온 연결을 대상 파드의 수신 대기 서버로 전달함
* 지금까지는 서비스의 정의에 있던 파드 셀렉터를 통해 엔드포인트가 암묵적으로 동시에 만들어진 것임
* 파드 셀렉터를 생략하고 서비스를 만들면 엔드포인트를 수동으로 구성하고 업데이트할 수 있음
* 바로 이 점을 활용해 외부 서버들에 접근하기 위한 서비스를 정의할 수 있음

```yaml
# 파드셀렉터를 정의하지 않았으므로 엔드포인트는 생성되지 않음
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```
```yaml
# 엔드포인트의 정의 name은 매칭되는 서비스와 일치해야함
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:  # 접속할 외부 서버들을 지정
    - ip: 11.11.11.11 
    - ip: 22.22.22.22
    ports:
    - port: 80
```

* 위와 같이 수동으로 엔드포인트를 구성하면 추후 외부 서버들을 클러스터내로 마이그레이션 하고자할 때 서비스 정의에 셀렉터를 추가하면 다시 엔드포인트를 자동으로 관리할 수 있음
* 반면 서비스에서 셀렉터를 제거하면 k8s는 엔드포인트 자동업데이트를 멈춤

---

* 엔드포인트를 수동으로 만들지 않고 외부 서비스의 별칭역할을 하는 서비스를 만들수도 있음
* `type: ExternalName` 을 사용
```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.somecompany.com # 실제 접속할 외부 서비스의 FQDN
  ports:
  - port: 80
```
* 이를 통해 이를 사용할 파드에서는 `api.somecompany.com`로 사용하는 대신 `external-service.default.svc.cluster.local` 과 같은 클러스터 내부의 FQDN 으로 외부 서비스에 연결할 수 있음
* 이 또한 구체적인 위치를 추상화한 것이므로 추후 환경변화에 유연하게 대응할 수 있음
* ExternalName 은 DNS레벨에서 구현됨 (실제로 CNAME DNS 레코드가 생성됨). 따라서 해당 서비스에 접근할 때는 서비스 프록시를 무시하고 외부 서비스에 직접 연결되고 이 때문에 ExternalName 유형의 서비스는 ClusterIP를 얻지 못 함.

## 5.3 외부 클라이언트에 서비스 노출

  시스템을 외부에 오픈하는 방법. VM 환경에서 맨 앞단에 위치하는 L4 역할이라고 보면 됨.
  
1. NodePort : 클러스터의 각 노드가 노드 자체에서 포트를 열고 해당 포트로 수신된 트래픽을 서비스로 전달
2. NodePort 의 확장인 로드밸런서 : 쿠버네티스가 실행 중인 인프라 환경(클라우드 등)에 프로비저닝된 전용 LB 사용. LB는 트래픽을 모든 노드의 노드포트로 전달
3. Ingress : L7 레벨로 작동. 단일 IP 주소로 여러 서비스를 노출하는 등 L4 대비 많은 기능 제공

### NodePort

* [kubia-svc-nodeport.yaml](kubia-svc-nodeport.yaml)
```bash
k get svc kubia-nodeport
NAME            CLUSTER-IP      EXTERNAL-IP PORT(S)
kubia-nodeport  10.111.254.223  <nodes>     80:30123/TCP
```
* 다음의 IP들로 접속 가능
  * 10.111.254.223:80
  * <첫 번째 노드IP>:30123
  * <두 번째 노드IP>:30123
  * ...
* 외부 인터넷의 사용자는 CLUSTER-IP 로는 못 붙으므로 각 노드의 IP를 알아야함
* 또한 각 노드들은 30123 포트를 오픈해야함 (inbound)
* 특정 노드가 down 되었을 때 클라이언트측은 이를 모르고 계속 요청을 보낼 수 있음

```bash
# Tip : JSONPath를 사용해 모든 노드의 IP 가져오기
k get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# 이외에도 jsonpath를 활용하면 다양한 리소스의 원하는 정보를 쉽게 얻을 수 있음
# https://kubernetes.io/ko/docs/reference/kubectl/jsonpath/
```

### 외부 로드밸런서로 서비스 노출

* 클라우드 공급자 (AWS, Azure 등)이 제공하는 k8s 는 보통 로드밸런서를 자동 프로비저닝하는 기능을 제공함
* 서비스 유형을 노드포트 대신 로드밸런서로 바꾸기만 하면 됨
* 로드밸런서 서비스는 노드포트 서비스의 확장임
* [kubia-svc-loadbalancer.yaml](kubia-svc-loadbalancer.yaml)
```bash
k get svc kubia-loadbalancer
NAME            CLUSTER-IP      EXTERNAL-IP     PORT(S)
kubia-nodeport  10.111.254.223  130.211.52.173  80:32143/TCP
```
* 서비스를 생성하면 사용중인 클라우드 인프라에서 로드밸런서를 생성함
* 그냥 NodePort 때와는 달리 EXTERNAL-IP 가 단일IP로 설정됨
* 사용자는 `http://130.211.52.173` 으로 접속가능함.
* 즉, 해당 공인 IP를 고정사용하도록 구매하고 DNS에 등록하면 시스템을 오픈할 수 있음
* 단순히 각 노드 앞단에 LB 장비가 추가된 형태이므로 그냥 노드포트처럼 각 노드에 방화벽을 오픈하여 노드에 직접 접속하는 것도 가능함

---

* 노드포트나 노드포트+LB 모두 클라이언트 IP가 유지되지 않음
* (내의견) LB 방식의 경우 XFF 적용이 가능하지 싶은데 이건 각 클라우드 벤더별로 확인이 필요한 사항

### Ingress

* https://kubernetes.io/ko/docs/concepts/services-networking/ingress/
* 한 IP 주소로 수십 개의 서비스를 제공할 수 있음
* 요청한 Host 와 경로에 따라 요청을 전달한 서비스를 결정함
* L7 LB이므로 그냥 서비스가 못 하는 쿠키 기반 세션 어피니티 등의 추가 기능을 제공함
* 인그레스를 실행하려면 클러스터에 인그레스 컨트롤러를 실행해야함. 컨트롤러는 클라우드 벤더별로 다를 수 있음.
* 인그레스의 기능은 인그레스 컨트롤러 구현마다 다르므로 세부 내용은 각 구현체의 문서를 확인해야함

---

* [kubia-ingress.yaml](kubia-ingress.yaml)
```shell
k get ingresses
```
* 동작방식은 인그레스 컨트롤러가 클라이언트의 요청을 받으면 `인그레스->서비스->엔드포인트` 를 통해 파드 IP를 조회한다음 인그레스 컨트롤러가 요청을 파드에 전달함
  * 모든 컨트롤러가 이런 것은 아니지만 대부분 이와 같다고 함
* 위 yaml을 보면 rules, paths 라고 되있는대로 한 개의 ingress에 복수의 host와 host별 경로를 설정할 수 있음
  * 실제 처리는 클라이언트가 보내는 HTTP의 `Host` 헤더로 구분함

---

* [kubia-ingress-tls.yaml](kubia-ingress-tls.yaml)
* 인그레스를 그냥 만들면 HTTP만 지원하며 HTTPS 는 안 됨
* tls 지원을 위해 개인키와 인증서를 통해 시크릿을 만들고 이를 yaml에 지정해야함
```shell
openssl genrsa -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.cert -days 365 -subj /CN=kubia.example.com
k create secret tls tls-secret --cert=tls.cert --key=tls.key
```

## 5.5 레디니스 프로브 (readiness probe) : 파드가 연결을 수락한 준비가 됐을 때 신호 보내기

* 라이브니스 프로브는 컨테이너가 살아있는지 주기적으로 점검하고 자동으로 재시작해주는 역할이었음
* 레디니스 프로브는 컨테이너가 준비가 된 것인지를 주기적으로 점검하고 준비된 컨테이너만 서비스에 등록되도록 유지함
* "준비가 됐다" 의 기준은 컨테이너마다 다름.
  * 웹서버라면 특정 GET 요청에 올바른 응답을 하는지가 될 수 있음
  * 각 애플리케이션의 요건에 맞게 개발자가 레디니스 프로브를 정의해야함

---

* 레디니스의 유형은 라이브니스처럼 3가지를 지원함
1. Exec 프로브 : 프로세스를 실행하고 그 종료 상태 코드로 컨테이너 상태를 확인함
2. HTTP GET 프로브 : http 응답의 상태 코드로 확인
3. TCP 소켓 프로브 : 지정된 포트로 소켓이 열리는지로 확인

---

* 레디니스의 동작
* 주기적으로 레디니스 프로브를 실행하고 파드가 준비되지 않았으면 서비스(엔드포인트)에서 제외하고 준비되면 다시 서비스에 추가함.
  * 이를 통해 준비든 파드만 응답하도록 구성할 수 있음
  * 최초의 레디니스 동작까지의 지연시간을 설정할 수 있음 (`initialDelaySeconds`)
* 라이브니스와 달리 프로브가 실패해도 파드를 제거하거나 재시작하지 않음. 단지 서비스에 추가할지 말지를 결정하는 것임.
* 이러한 레디니스를 적절히 구성하면 클라이언트는 정상인 파드와만 통신하게 되므로 일부 문제가 발생한 파드가 있어도 영향을 받지 않게 됨.

---

* [kubia-rc-readinessprobe.yaml](kubia-rc-readinessprobe.yaml)
* 위와 같이 파드템플릿 부분에 readiness 을 정의할 수 있음
* 레디니스는 아래와 같이 적용됨
* 기본적으로 10초 주기로 실행됨
```bash
k describe po kubia-2cc8v
# ...
Readiness:      exec [ls /var/ready] delay=0s timeout=1s period=10s #success=1 #failure=3
```

---

* 모든 파드는 레디니스 프로브를 정의하는 것을 권장
* 정의하지 않으면 생성 즉시 서비스에 등록됨. 이 경우 초기 실행에 시간이 걸리는 파드의 경우 응답이 지연되거나 실패하는 경우가 발생하게됨.
* k8s는 파드를 삭제하자마자 모든 서비스에서 파드를 제거한다.

## 5.6 헤드리스 서비스로 개별 파드 찾기

* 앞에서 살펴본 서비스는 기본적으로 엔드포인트에 등록된 임의의 파드로 요청을 전달함.
* 반면에 특정 클라이언트가 모든 파드에 연결해야하거나, 파드가 다른 특정 파드에 각각 연결해야하는 경우에 지금까지의 서비스는 적합하지 않음.
* 이 경우 헤드리스 서비스를 사용할 수 있음

---

* [kubia-svc-headless.yaml](kubia-svc-headless.yaml)
* `clusterIP: None` 이 서비스를 헤드리스 서비스로 만듬
* 준비가 파드가 2개 이상일 때 DNS 조회를 수행해보면
```bash
# dnsutils 는 nslookup 이 설치된 임의의 파드라 가정
k exec dnsutils -- nslookup kubia-headless
# 현재 준비된 상태의 모든 파드의 DNS A 레코드가 반환됨
# ...
Name:     kubia-headless.default.svc.cluter.local
Address:  10.101.1.4
Name:     kubia-headless.default.svc.cluter.local
Address:  10.101.1.5

# 기존에 만들어둔 평범한 서비스에 요청하면 서비스의 clusterIP 1개 응답이 옴
k exec dnsutils -- nslookup kubia
# ...
Name:     kubia.default.svc.cluter.local
Address:  10.98.57.206
```
* 헤드리스 서비스를 사용해도 클라이언트 입장에서는 크게 다를 게 없음
* 하지만 파드의 IP를 직접 반환하므로 클라이언트는 서비스 프록시가 아닌 파드에 직접 연결함
* 로드밸런싱 측면에서는 기본적인 서비스는 서비스 프록시가 담당하지만 헤드리스 서비스는 DNS 라운드 로빈 메커니즘으로 동작하는 것도 특징

---

* 때로는 서비스 레이블 셀렉터에 매칭되는 모든 파드를 찾고 싶을 수 있음. (준비되지 않은 파드 포함)
* 이 경우 DNS 조회 메커니즘을 사용할 수 있음
  * 서비스 스펙에 `publishNotReadyAddresses=True` 을 지정하면 파드의 준비상태에 관계없이 DNS 레코드로 등록됨