# 5장 서비스: 클라이언트가 파드를 검색하고 통신할 수 있게 함

* VM 기반의 시스템에서는 접근하고자 하는 서비스의 정확한 서비스 IP나 호스트명으로 구성하는 경우가 많음
* 하지만 k8s 에서 파드는 일시적이라 언제든지 늘어나거나 제거될 수 있으며 IP 주소도 생성시점에 결정되므로 미리 IP를 알 수도 없음

# 5.1 서비스 소개

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
  * 하지만 서비스 생성전에 만들어진 파드에서는 조회되 않음

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

