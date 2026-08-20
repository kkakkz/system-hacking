## 어셈블리로 syscall 다루기 (read/write/open)
- 이 강의가 "왜 종료하는 법을 배웠나"에 대한 답이자, 지금까지 배운 syscall 지식(rax=번호, rdi/rsi/rdx=인자)을
`read`,`write`,`open`이라는 실전 syscall 세 개로 확장하는 내용이다.

## 왜 배워야 하는가
>"syscall은 사실 `call`명령어랑 비슷한데, 프로그램 내부가 아니라 **운영체제로 점프**하는 것뿐이다."
지금까지 `exit`하나만 다뤄서 "syscall=그냥 종료하는 것"처럼 느껴질 수도 있는데, 사실 **네트워크, 파일, 화면출력 등 "순수 연산이 나닌 모든 것"이 다 syscall을 통해 이루어진다**
`read`/`write`/`open`을 배우면 비로소 프로그램이 "입력을 받고, 파일을 열고, 결과를 출력하는" 진짜 프로그램다운 동작을 어셈블리로 짤 수 있게 된다.

## 핵심개념

### `read` syscall
```assembly
mov rdi, 0      ; 1번째 인자: 파일 디스크립터 (0 = 표준 입력, stdin)
mov rsi, rsp    ; 2번째 인자: 읽은 데이터를 저장할 버퍼 주소 (여기선 스택)
mov rdx, 100    ; 3번째 인자: 몇 바이트를 읽을지
mov rax, 0      ; syscall 번호: read = 0
syscall
; 리턴값: rax에 "실제로 읽은 바이트 수"가 담김
```
**syscall 인자 순서(rdi->rsi->rdx->r10 ...)**가 여기서 실전으로 처음 3개 다 쓰이는 사례이다.

### `write` syscall
```assembly
mov rdi, 1      ; 1번째 인자: 파일 디스크립터 (1 = 표준 출력, stdout)
mov rsi, rsp    ; 2번째 인자: 출력할 데이터가 있는 메모리 주소
mov rdx, rax    ; 3번째 인자: 몇 바이트를 쓸지 (직전 read의 리턴값을 재활용!)
mov rax, 1      ; syscall 번호: write = 1
syscall
```
`read`의 리턴값(`rax`에 담긴 "실제로 읽은 바이트 수")를 그대로 `write`의 3번쨰 인자 (`rdx`)로 재사용한다 - 읽은 만큼만 정확히 다시 출력하는 자연스러운 패턴.

### 파일 디스크립터(fd)개념
```
0 = stdin  (표준 입력)
1 = stdout (표준 출력)
```
`read`/`write` 의 1번째 인자는 **"어디서 읽을지/어디로 쓸지"를 나타내는 정수(fd)**이다. 이 숫자 자체가 왜 0,1 인지는 나중에 더 다룰텐데, 일단 지금은 "입력은 0, 출력은1"로 외워두면 된다.

### 스택 위에 직접 문자열 만들기
```assembly
mov byte ptr [rsp+0], '/'
mov byte ptr [rsp+1], 'f'
mov byte ptr [rsp+2], 'l'
mov byte ptr [rsp+3], 'a'
mov byte ptr [rsp+4], 'g'
mov byte ptr [rsp+5], 0     ; null terminator
```
문자열 `"/flag"`를 스택 메모리에 한 바이트씩 직접 써서 만드는 과정. 결과적으로 스택엔 `2f 66 6c 61 67 00`(ASCII `/`,`f`,`l`,`a`,`g`,`\0`)이 저장된다.

**핵심 규칙**: 문자열은 **ASCII 바이트들의 연속 + 마지막에 0바이트(null terminator)** 로 끝난다는걸 여기서 명시적으로 배운다. 이것이 나중에 "C 스타일 문자열"의 정체이다.

### `open` syscall
```assembly
mov rdi, <"/flag" 문자열의 스택 주소>   ; 1번째 인자: 열 파일 경로 (포인터!)
mov rsi, 0                            ; 2번째 인자: flags (0 = 읽기 전용)
mov rax, 2                            ; syscall 번호: open = 2
syscall
; 리턴값: rax에 새로 열린 파일의 fd(파일 디스크립터) 번호가 담김
```
`read`/`write`와 다르게, `open`의 1번째 인자는 **숫자가 아니라 문자열의 메모리 주소(포인터)** 이다. 지금까지 배운 "포이터로 데이터 가리키기" 개념이 여기서 실전으로 쓰인다.

### syscall 번호/플래그 상수는 어떻게 알아내나
상수들(`0_RDONLY=0` 등)은 그냥 외우거나 찾아봐야 한다. man page를 참고하거나, C프로그램으로 매크로 값을 출력해서 확인하는 방법이 있다.
