## Control Flow
- 이 강의가 사실상 computing 101 기초과정의 최종 보스이다. 지금까지 배운 레지스터/메모리/스택이 전부 여기로 수렴된다

## 왜 배워야 하는가
- 지금까지는 "데이터를 계산" 하는 법만 배웠었는데, 프로그램이 **"조건에 따라 다른 행동을 하게"** 만들려면 흐름 자체를 바꿀 수 있어야 한다.
그리고 **버퍼 오버플로우 익스플로잇의 최종 목표가 정확히 이것이다** : `rip`(**다음 실행할 명령어 주소**)를 **공격자가 원하는 곳으로 강제로 바꾸는 것**. 이 강의에서 배우는 `jmp`,`call`,`ret`이 전부 결국 "rip를 어떻게 조작하는가"에 대한 명령어들이다.

## 핵심개념

### 1. 코드도 결국 메모리 안의 데이터일 뿐이다
```
메모리 0x400800: pop rax
메모리 0x400801: pop rbx
메모리 0x400802~804: add rax, rbx
메모리 0x400805: push rax
```
CPU는 `rip`가 가리키는 주소의 바이트를 명령어로 해석해서 실행하고, 끝나면 자동으로 다음 명령어 주소로 `rip`를 이동시킨다.
**x86은 가변 길이 명령어 집합**이라, 명령어마다 차지하는 바이트 수가 다르다.

### 2. `jmp` - 무조건 점프
```assembly
mov cx, 1337
jmp stay_lead    ; 이 라벨로 무조건 점프 (다음 명령어를 건너뜀)
mov cx, 0        ; ← 점프 때문에 실행 안 됨!
stay_lead:
push rcx
```
**라벨(`stay_lead:`)은 실제 명령어가 아니라, 어셈블러가 "몇 바이트를 건너뛸지" 계산하게 도와주는 도구**이다.
`jmp`는 내부적으로 "rip에 부호 있는 숫자를 더하는"것뿐이고 (`jmp eb04`처럼), 이 숫자가 음수면 뒤로 점프한다.

`eb fe`(자기 자신으로 점프)는 무한 루프를 만드는 유명한 패턴이다.

### 3. `rflags`(플래그 레지스터)와 조건 비교
```assembly
cmp rax, rbx  ; rax - rbx를 계산하되, 결과는 버리고 플래그만 갱신
test rax, rax ; rax AND rax, 0인지만 체크할 때 씀
```
`cmp`/`test`는 계산 결과 자체보다 **플래그(zero flag, sign flag, carry flag, overflow flag)를 갱신하는데 의미**가 있다. 이 플래그들을 나중에 조건 점프가 검사한다.

### 4. 조건부 점프(conditional jump)
```assembly
cmp rax, rbx
je stay_lead  ; jump if equal (플래그 확인 후 점프할지 결정)
```
`je`(같으면), `jne`(다르면), `jg`(크면, signed), `ja`(크면, unsigned) 등 다양한 조건 점프가 있다.
**중요 포인트** : 부호있는 비교(`jg`)와 부호 없는 비교(`ja`)는 서로 다른 플래그 조합을 검사한다. 
예를 들어 `0xFFFF`는 signed로 보면 음수지만, unsigned로 보면 아주 큰 양수이다. 그래서 같은 비교 명령어(`cmp`)를 쓰고도, 어떤 점프 명령어를 쓰느냐에 따라 부호 해석이 달라진다.

### 5. 루프(loop)
```assembly
mov rcx, 0
loop_header:
  inc rcx
  cmp rcx, 10
  jb loop_header  ; below(unsignedless than)면 루프 처음으로 되돌아감
; 루프 끝나면 rcx = 10
```
조건 점프로 "뒤로" 되돌아가는 걸 반복하면 그게 곧 루프이다. 별도의 "loop문법" 이 있는게 아니라, `jmp`/조건 점프의 조합이다.

### 6. 함수 : `call`과 `ret`
```assembly
check_lead:
  test rdi, rdi  ; 첫 번째 인자(rdi)가 0인지 체크
  jnz not_lead
  mov rax, 0
  ret
not_lead:
  mov raxm 1337
  ret

mov rdi, 0
call check_lead  ; check_lead로 점프하면서, "돌아올 주소"를 스택에 push
; call이 리턴하면 여기로 돌아옴
```
이게 "함수 호출/리턴"의 정체이다. `call`/`ret` 자체가 스택을 이용한 점프 명령어 쌍일 뿐이다.

### 7. Calling Convention (함수 호출 규약)
```text
1번째 인자 = rdi
2번째 인자 = rsi
3번째 인자 = rdx
4번째 인자 = rcx
5번째 인자 = r8
6번째 인자 = r9
리턴값 = rax
```
이미 syscall에서 배운 규칙(rdi, rsi, rdx ... )과 거의 같다. 단, 4번째 인자가 syscall은 `r10`, 일반 함수 호출은 `rcx`라는 차이만 있다.

### 8. Caller-saved vs Callee-saved 레지스터
```
함수가 값을 보존해줘야 하는 레지스터(callee-saved): rbx, rbp, r12~r15
   → 이 레지스터들을 함수가 건드리려면, 먼저 push해서 저장해두고, 
     리턴 전에 pop해서 원상복구해야 함
그 외 레지스터: 함수가 마음대로 바꿔도 됨 (caller-saved)
   → 호출하는 쪽이 필요하면 알아서 미리 저장해둬야 함
rsp: 항상 스택 관리를 위해 유지되어야 함
```

>이 강의 전체가 시스템해킹의 뼈대이다.
> - BOF(버퍼 오버플로우) : 스택에 저장된 "리턴 주소"(call이 push해둔 그 주소)를 덮어써서, `ret`이 공격자가 원하는 곳으로 점프하게 만드는게 BOF 익스플로잇의 본질
> - ROP(Return-Oriented Programming): `ret`으로 여러 코드 조각을 연쇄적으로 이어붙이는 기법이 바로 이 `call`/`ret` 메커니즘을 악용하는 것
> - calling convention : syscall 인자 전달 규칙과 거의 동일해서, 이미 배운 지식이 그대로 재사용됨
> - signed vs unsigned 비교: 정수 관련 취약점(부호 혼동)이 이 조건 점프 선택에서 비롯됨
