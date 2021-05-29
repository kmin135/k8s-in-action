# 2장 도커와 쿠버네티스 첫걸음

* p101 의 `kubectl expose rc kubia --type=LoadBalancer --name kubia-http` 는 동작하지 않는다.
```
kwon@DESKTOP-0U32D8C:~/k8s-in-action/chap02$ k expose rc kubia --type=LoadBalancer --name kubia-http
Error from server (NotFound): replicationcontrollers "kubia" not found
```
* 이는 p96 에서 구동한 `kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1`  가 deprecated 되었는데 이에 따라 ReplicationController 없이 standalone pod를 띄우는 걸로 동작이 변경됐기 때문이다. 즉. rc가 없으니 expose도 당연히 못 하는 것.
* 대신 `kubectl expose pod kubia --type=LoadBalancer --name kubia-http` 와 같이 pod 를 expose 하는걸로 예제는 해볼 수 있다.

* 참고 : https://stackoverflow.com/questions/64306744/error-when-trying-to-expose-a-docker-container-in-minikube


## 명령어

```bash
# 클러스터 정보 표시
kubectl cluster-info

# 클러스터 노드 조회
kubectl get nodes
# 특정 노드 상세 조회
kubectl describe node kubia-0rrx

# 실행중인 파드를 조회하되 상세정보까지 표시 (Pod IP, Pod가 실행중인 노드)
kubectl get pods -o wide
# 특정 파드 상세정보 확인
kubectl describe pod kubia
```

## 별칭 설정 및 자동완성

```bash
echo "alias k=kubectl" >> ~/.bashrc
source ~/.bashrc

# 자동완성은 bash-completion 패키지를 설치후 아래 스크립트 수행
echo "source <(kubectl completion bash | sed s/kubectl/k/g)" >> ~/.bashrc
source ~/.bashrc
```

## docker-desktop 에서 k8s dashboard 사용하기

* k8s dashboard는 클러스터의 현황을 GUI로 제공해주는 도구임
* 필요하면 아래 링크 참고해서 해보기
* https://andrewlock.net/running-kubernetes-and-the-dashboard-with-docker-desktop/