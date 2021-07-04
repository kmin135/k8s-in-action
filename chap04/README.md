# 4장 레플리케이션과 그 밖의 컨트롤러: 관리되는 파드 배포

* 일반적으로 파드를 직접 생성할일은 거의 없음
* 대신 레플리케이션컨트롤러 또는 디플로이먼트 같은 리소스를 생성하여 파드를 생성하고 관리함

## 파드를 안정적으로 유지하기

* 기본적으로 파드가 노드에 스케줄링되면 해당 노드의 kubelet은 파드의 컨테이너를 실행하고 계속 실행되도록 유지함
* 컨테이너의 주 프로세스에 crash가 발생하면 kubelet이 컨테이너를 다시 시작함
* 그러나 OOM 처럼 치명적인 문제이긴 하지만 프로세스 자체는 살아있다면 kubelet이 이를 감지하지 못 함
* 또는 애플리케이션 내부의 무한 루프, 교착 상태 등이 발생하여 응답을 하지 않는 상황도 생각할 수 있음
* 이런 경우까지 상정하여 애플리케이션이 다시 시작되도록 