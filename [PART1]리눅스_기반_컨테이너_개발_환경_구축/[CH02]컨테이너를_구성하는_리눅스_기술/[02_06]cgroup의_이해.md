## [02_06] cgroup의 이해

### 컨테이너 구성하는 3가지 주요 리눅스 기술
- Control groups
- Namespaces
- Union mount filesystem

### cgroups(Control Groups)
- 프로세스들이 사용하는 시스템 자원의 사용 정보를 수집 및 제한시키는
  - 리눅스 커널 기능
  - 모든 ps에 대한 리소스 사용정보 수집
- 제한 가능한 자원
  - CPU, Memory, Network, Device, Block I/O
- 활용 사례
  - runc, YARN(Hadoop), Android 등
- Android 예시
  - Forground, Visible
  - Service, Backgroup, Empty
  - 위 두가지 상황에 따라 자원 제어 가능
- groupv1, groupv2의 차이
  - v1: blkio, memory, pids로 구분
    - 대부분 v1을 사용
  - v2: bg, adhoc 구분

### groups 상세
- **cpu**
  - (CFS) 스케줄러를 이용해 해당 cgroup에 속한 **CPU 사용 시간** 제어
    - e.g. 해당 그룹의 cpu 사용량 10%로 제한 -> 10% 초과 사용시?
    - 사용량이 아닌, 시간으로 제어
- **memory**
  - 해당 cgroup에 속한 프로세스의 memory 사용량에 대한 제어
    - 해당 그룹의 메모리 사용량을 128m으로 제한 -> 128m 초과 사용시? (`oom_control`)
      - 항상 oom으로 죽일지 말지 판단 가능
- **freezer**
  - cgroup의 작업을 일시 중지하거나 다시 시작 -> `docker pause`
- blkio
  - cgroup에 **블록 장치**에 대한 IO 제한 설정(저장공간 제한 개념 X)
- net_cls
  - Linux 트래픽 컨트롤러(tc)가 특정 cgroup 작업에서 발생하는 패킷을 식별하게 하는
  - 클래스 식별자(classid)를 사용하여 **네트워크 패킷**에 **태그**를 지정
- cpuset
  - 개별 cpu 및 메모리 노드를 cgroup에 할당
- cpuacct
  - cgroup이 사용한 CPU 자원에 대한 보고서 생성
- devices
  - cgroup의 작업 단위로 장치에 대핸 액세스를 허용하거나 거부
- ns
  - namespace 서브 시스템

### cgroup 사용 사례
- WEB Service
  - Core 워크로드에 대한 영향을 최소화 하기 위해 적용
  - 그룹별 memory, IO 제한을 걸어 사용
- kubernetes pod
  - `resources.limits.cpu, mem`에 따라 지정 제출
  - **CRI**(gRPC) 스펙에 맞게 변환
  - runc의 OCI스펙에 맞게 변환
  - 프로세스 run