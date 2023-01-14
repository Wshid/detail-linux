## [01_14] (실습) OverlayFS로 Union Mount 해보기

### 개념 정리
- Control groups
  - 자원의 사용량 결정(CPU, memory, network, storage)
- Namespacs
  - 자원 격리(네트워크, 프로세스 정보, 사용자 정보)
- Union mount fileSystem
  - 이미지 효율적 관리(이미지 레이어, CoW)