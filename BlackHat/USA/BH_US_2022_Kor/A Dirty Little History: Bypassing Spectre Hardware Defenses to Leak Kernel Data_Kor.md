# A Dirty Little History: Bypassing Spectre Hardware Defenses to Leak Kernel Data

1. 소개 (Introduction)

1.1 개요
이 논문은 Spectre 취약점이 대부분의 현대 CPU에 영향을 미치며, 이를 통해 권한 수준을 넘어서 데이터가 유출될 수 있음을 설명한다. CPU 제조업체들은 이 취약점을 막기 위해 하드웨어 방어 기법을 도입했으며, 본 논문은 이러한 방어 기법이 실제로 효과가 있는지 탐구했다.

2. Spectre-101

2.1 기본 원리
Spectre 취약점은 분기 예측을 악용하여 권한이 없는 데이터 접근을 허용한다. 예를 들어, 조건문을 통해 배열의 인덱스를 검사하는 경우, 분기 예측 유닛(BPU)이 잘못된 예측을 할 수 있으며, 이를 통해 메모리 경계를 벗어난 읽기가 발생한다.

```
if (x < array.size) // size = 128
    y = array[x];

```

2.2 Flush+Reload 공격
이러한 예측 실패는 Flush+Reload 공격 기법과 결합되어 캐시를 통해 데이터를 유출할 수 있다.

```
if (x < array.size) // size = 128
    y = array[x];
    z = reload_buff[y];
```

3. Spectre Hardware Defenses

3.1 소프트웨어 방어 기법
Retpoline: 분기 타겟을 컨트롤하는 방법으로, Intel과 AMD에서 도입.
Arm: Branch Target Indicator라는 방어 메커니즘을 사용한다. BTI는 분기 타겟이 올바른지 검증하여 잘못된 타겟으로의 분기를 방지한다.

3.2 하드웨어 방어 기법
Intel eIBRS: 분기 타겟 버퍼(BTB) 엔트리를 보안 도메인별로 태그하여 분리한다. 이를 통해 사용자 모드와 커널 모드 간의 분기 타겟 혼란을 방지한다.
Arm CSV2: 유사한 방법으로 구현되었으며, CPU가 다른 보안 도메인에서 온 분기 타겟을 무시하도록 하는 메커니즘이다.

4. Spectre 하드웨어 방어 우회 (Bypassing Spectre Hardware Defenses)

4.1 개요
Spectre 하드웨어 방어를 우회하기 위한 방법을 탐구한다. 사용자 히스토리가 커널 예측 정확성을 높이는 데 필요하다는 직관을 바탕으로 한다.

```
printf('Hello');
syscall(write, stdout, 'Hello', 5);
sys_call_table[NR_write](regs);
sys_write;
```

4.2 실험 결과
Intel eIBRS: 완벽한 오예측 발생.
Arm CSV2: 완벽한 오예측 발생.
AMD Retpoline: 오예측 발생하지 않음.

5. Branch History Injection (BHI)
5.1 개요
BHI는 사용자 공간의 분기 히스토리를 사용하여 커널 분기 예측을 제어하는 방법이다. 이를 통해 특정 타겟을 예측하게 만들 수 있다.

```
br_a: jmp rax target_a
target_b;
```

5.2 BHI의 악용
공격자는 분기 히스토리를 제어하여 커널의 간접 분기 타겟을 임의로 예측하게 만들 수 있다.

```
TAG PRIV TARGET
kernel TAG_A kern_func_a;
user TAG_B user_func;
kernel TAG_C kern_func_b;
```

6. Exploitation (익스플로잇)

6.1 계획 (The Plan)
커널 공간에서의 분기 예측을 제어하여 민감한 데이터를 유출하는 방법을 설명한다. 예를 들어, 시스템 콜 핸들러를 희생양으로 삼아 쉽게 트리거할 수 있다.

```
syscall(getpid);
sys_call_table[NR_getpid](regs);
sys_getpid;
```

6.2 Victim Branch (희생양 분기)
시스템 콜 핸들러는 임의의 시스템 콜을 통해 쉽게 트리거할 수 있으며, RDI는 사용자 공간의 저장된 레지스터를 가리킨다.

```
syscall(getpid);
```

이 논문은 Spectre 하드웨어 방어 기법을 우회하여 커널 데이터를 유출하는 방법을 구체적이고 자세하게 설명하며, 여러 공격 시나리오와 방어 기법을 포함한다.
