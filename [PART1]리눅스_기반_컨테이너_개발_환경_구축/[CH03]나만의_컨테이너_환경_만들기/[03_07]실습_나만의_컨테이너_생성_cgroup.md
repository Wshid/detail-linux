## [03_07] (실습) 나만의 컨테이너 생성 - cgroup

### 요구사항
- Process 갯수 제한(PID)
- 컨테이너 내 process 갯수 제한
- 실습 방법
  - **fork bomb**
  - fork를 반복해서 수행하는 과정
    ```bash
    # 시스템을 다운시킬 수 있는 스크립트
    :(){ :|:& };:
    ```

### 실습
```go
# 위 내용은 이전 실습과 동일

func child() {
  fmt.Printf("Running child: %v as %d\n", os.Args[2:], os.Getpid())

  // 새로 추가된 함수
  cg()

  // hostname 설정
  syscall.Sethostname([]byte["container"])

  // 루트 파일 시스템 다운로드: https://hub.docker.com/_/ubuntu
  // Dockerhub내에 ubunutu를 검색하여 다운로드
  /*
  mkdir /tmp/ubuntu
  tar zxf ubuntu-focal-oci-amd64-root.tar.gz -C /tmp/ubuntu
  */

  // 루트 파일 시스템 변경
  syscall.Chroot("/tmp/ubuntu") // 다운받은 ubuntu로 fs 변경
  syscall.Chdir("/") // root로 이동
  syscall.Mount("proc", "proc", "proc", 0, "")
  // 함수가 종료될때까지 지연 이후 수행, ps 종료시에 unmount 하기 위함
  defer syscall.Unmount("proc", 0)

  cmd := exec.Command(os.Args[2], os.Args[3:]...)
  cmd.Stderr = os.Stderr
  cmd.Stdin = os.Stdin
  cmd.Stdout = os.Stdout

  must(cmd.Run())
}

func cg() {
  cgroups := "/sys/fs/cgroups/"
  pids := filepath.Join(cgroups, "pids")
  // linux_campus이라는 control group 생성
  os.Mkdir(filepath.Join(pids, "linux_campus"), 0755)
  // control group 설정 구문
  must(ioutil.WriteFile(filepath.Join(pids, "linux_campus/pids.max"), []byte("20"), 0700)) // pid 갯수 최대 20개로 제한
  must(ioutil.WriteFile(filepath.Join(pids, "linux_campus/notify_on_release"), []byte("1"), 0700))
  must(ioutil.WriteFile(filepath.Join(pids, "linux_campus/cgroup.procs"), []byte(strconv.Itoa(os.Getpid())), 0700)) // 자신의 pid를 control group내에 등록
  // 위 구문을 수행해야 정상 적용됨
}
```
- 수행
  ```bash
  go run . run /bin/bash
  :(){ :|:& };: # fork-bomb!
  # cgroup이 정상 적용되었다면 아래와 같은 에러 문구 발생
  # bash: fork: retry: Resource temporarily unavailable

  cd /sys/fs/cgroup/pids/
  ls # linux_campus 디렉터리 존재 여부 확인
  cd linux_campus
  cat pids.max
  cat tasks  # ps 확인
  cat pids.current # 현재 pids 수 확인
  ```
