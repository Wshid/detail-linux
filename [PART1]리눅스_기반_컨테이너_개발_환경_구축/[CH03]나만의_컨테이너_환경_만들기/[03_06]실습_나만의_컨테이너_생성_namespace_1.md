## [03_06] (실습) 나만의 컨테이너 생성 - namespace_1

### 요구사항
- 호스트명 변경(UTS), process id 변경(PID), process list 정보(mount, proc) 변경
- 1단계: run 명령어 전달시 run 함수 실행
- 2단계: 새로운 프로세스에서 명령어 실행
- 3단계: 새로운 UTS 설정 추가 -> hostname 변경
- 4단계: 컨테이너 환경 시작시 호스트명을 container로 변경
- 5단계: 컨테이너 환경에서 ps 명령 실행시 제한된 프로세스 정보만 조회
  - root filesystem 변경

### 실습 - step1
```go
package main

import {
  "fmt"
  "os"
}

// docker run image <CMD> <ARG>
// go run main.go run <CMD> <ARG>

// Step1: 명령어 종류에 따른 함수 실행. run 명령어 전달시 run 함수 실행
func main() {
  switch os.Args[1] {
    case "run":
      run()
    default:
      os.Exit(1)
  }
}

func run() {
  // 전달 받은 argument 출력
  fmt.Printf("Running: %v\n", os.Args[2:])
}
```
- 수행
  ```bash
  go run . run ls -l 
  go run . foo ls -l # 비정상 수행
  ```

### 실습 - step2
```go
package main

import {
  "fmt"
  "os"
  "os/exec"
}

// Step2: 새로운 프로세스에서 명령어 실행
func main() {
  switch os.Args[1] {
    case "run":
      run()
    default:
      os.Exit(1)
  }
}

func run() {
  fmt.Printf("Running: %v\n", os.Args[2:])
  // 전달받은 Args를 기준으로 새로운 ps 생성
  cmd := exec.Command(os.Args[2], os.Args[3:]...)
  cmd.Stderr = os.Stderr
  cmd.Stdin = os.Stdin
  cmd.Stdout = os.Stdout

  cmd.Run()
}
```
- 수행
  ```bash
  go run . run pwd
  ```

### 실습 - step3
```go
package main
import {
  "fmt"
  "os"
  "os/exec"
  "syscall"
}

// Step3: 새로운 UTS 설정 추가. Clone flag 추가 NEW UTS namespace. hostname 수동 변경 실습
func main() {
  switch os.Args[1] {
    case "run":
      run()
    default:
      os.Exit(1)
  }
}

func run() {
  fmt.Printf("Running: %v\n", os.Args[2:])
  cmd := exec.Command(os.Args[2], os.Args[3:]...)
  cmd.Stderr = os.Stderr
  cmd.Stdin = os.Stdin
  cmd.Stdout = os.Stdout

  // 명령어 수행 전 Property 지정
  cmd.SysProcAttr = &syscall.SysProcAttr{
    // ps 생성시 새로운 namespace를 지정
    Cloneflags: syscall.CLONE_NEWUTS,
  }

  must(cmd.Run())
}

// error가 발생했을 때, 에러를 확인하고, ps를 종료
func must(err error) {
  if err != nil {
    panic(err)
  }
}
```
- 수행
  - `/etc/profile`내에 다음 설정 추가
    ```bash
    export PATH=$PATH:/usr/local/go/bin
    ```
  - 명령 수행
    ```bash
    sudo su # 관리자 권한 필요
    go run . run /bin/bash
    hostname
    hostname container
    hostname # 다르게 노출되는지 확인 필요
    ```

### 실습 - step4
```go
package main
import {
  "fmt"
  "os"
  "os/exec"
  "syscall"
}

// Step4: 컨테이너 환경 시작시 호스트명을 container로 변경
func main() {
  switch os.Args[1] {
    case "run":
      run()
    case "child":
      child()
    default:
      os.Exit(1)
  }
}

func run() {
  fmt.Printf("Running: %v\n", os.Args[2:])
  // /proc/self/exe: 자기 자신의 프로그램 실행
  // argument로 child와 전달받은 argument 전달
  cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
  cmd.Stderr = os.Stderr
  cmd.Stdin = os.Stdin
  cmd.Stdout = os.Stdout

  cmd.SysProcAttr = &syscall.SysProcAttr{
    Cloneflags: syscall.CLONE_NEWUTS,
  }

  must(cmd.Run())
}

func child() {
  fmt.Printf("Running: %v\n", os.Args[2:])
  
  // hostname 설정
  syscall.Sethostname([]byte("container"))

  cmd := exec.Command(os.Args[2], os.Args[3:]...)
  cmd.Stderr = os.Stderr
  cmd.Stdin = os.Stdin
  cmd.Stdout = os.Stdout

  must(cmd.Run())
}

func must(err error) {
  if err != nil {
    panic(err)
  }
}
```
- 명령 수행
  ```bash
  go run . run /bin/bash # root@container로 수행 확인
  ```

### 실습 - step5
```go

// Step5: 컨테이너 환경에서 ps 명령 실행 시 제한된 프로세스 정보만 조회. 루트 파일 시스템 변경.
//  실습으로 ps, cat /os-release 명령 실행

func main() {
  switch os.Args[1] {
    case "run":
      run()
    case "child":
      child()
    default:
      os.Exit(1)
  }
}

func run() {
  fmt.Printf("Running: %v\n", os.Args[2:])
  cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
  cmd.Stderr = os.Stderr
  cmd.Stdin = os.Stdin
  cmd.Stdout = os.Stdout

  cmd.SysProcAttr = &syscall.SysProcAttr{
    // CLONE_NEWPID: 새로운 PID namespace 생성
    // CLONE_NEWNS: mount의 new namespace 지정
    Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS,
  }

  must(new.Run())
}

func child() {
  fmt.Printf("Running child: %v as %d\n", os.Args[2:], os.Getpid())

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
```
- 수행
  ```bash
  go run . run /bin/bash
  echo $$ # pid 확인
  ps aux | head -5 # /sbin/init 확인, ps 목록이 local과 다른지 확인
  ```