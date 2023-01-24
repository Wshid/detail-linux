## [03_04] out_of_memory 로그 확인
- Out of Memory 로그 확인
  - `OOM Killer`가 특정 프로세스를 종료 시켜서, 메모리 공간을 확보

### 어떻게 메모리가 부족한 상황을 재현하는가
- `swap`이 비활성화된 리눅스 머신(e.g. EC2 default)
  - `swap` 여부 확인
    ```bash
    # 따로 row가 없음
    cat /proc/swaps
    
    # SwapTotal이 0으로 노출
    cat /proc/meminfo
    ```
- 회수가 불가능한 메모리를 지속적으로 증가
  - `file backed memory X`, 
  - `anonymous memory` 공간 사용을 늘림
    - e.g. `heap` 공간을 지속적으로 할당
    - 프로그램이 동적으로 만들어내기 때문에, 파일 원본이 X
- 부하 발생
  ```bash
  # 아래 명령 수행 후, 일정 기간 유지 및 SIGINT(Ctrl+C)
  stress-ng --bigheap 10
  ```
- `Out of Memory` 킬러 동작과 관련된 커널 메세지(`/var/log/syslog`)
  ```log
  stress-ng: invoked with 'stress-n' by user 1000
  
  kernel: stress-ng-highe invoked oom-killter: gfp_mast=...

  Out of memory: Killed process ...

  oom_reaper: reaped process ...
  ```
  - 위 정보 외
    - stack trace
    - register
    - 프로세스 정보
    - 메모리 회수 정보

### OOM 킬러의 프로세스 종료 기준(v5.19)
- 가장 높은 `oom_score`를 가진 프로세스가 종료
  ```bash
  /proc/PID/oom_score
  ```
- score의 범위: `0 ~ 1000`
  ```bash
  cat /proc/1/oom_score
  ```
- score를 높이는 경우
  - 메모리를 많이 사용하는 경우
    - e.g. rss, pagetable 크기가 큰 경우
  - `oom_score_adj`가 높은 경우
- `oom_killer`관련 코드
  - https://github.com/torvalds/linux/blob/v5.19/mm/oom_kill.c
#### 메모리를 많이 사용하더라도 종료될 확률을 낮추려면
- `oom_score_adj`를 낮게 설정

#### 특정 프로세스가 OOM 킬러에 의해 종료되지 않게 하려면
- HINT: `OOM_SOCRE_ADJ_MIN=-1000`
  ```bash
  sudo su
  echo -1000 > /proc/465/oom_score_adj
  
  # 해당 값이 0으로 변경됨
  cat /proc/465/oom_score
  ```
