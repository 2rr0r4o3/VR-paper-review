# Lost Control: Breaking Hardware-Assisted Kernel Control-Flow Integrity with Page-Oriented Programming

1. 서론 (Introduction)
서론에서는 Control-Flow Integrity(CFI) 개념과 CFI의 중요성에 대해 설명한다. CFI는 프로그램 실행 흐름의 무결성을 보장하기 위해 사용되며, 실행 흐름을 사전에 정의된 Control-Flow Graph(CFG)에 따라 제한한다. 하지만, 현재의 CFI 구현들은 주로 간접 분기(Indirect Branch)에 집중하고 있으며, 코드가 쓰기 불가 상태임을 전제로 한다.

2. Control-Flow Integrity (CFI)
CFI는 프로그램의 합법적인 실행 흐름을 정의하는 Control-Flow Graph(CFG)를 생성하여, 런타임 동안 실행 흐름을 모니터링하고 CFG에서 벗어나는 흐름을 방지한다. CFG는 정적 분석과 동적 분석을 통해 생성되며, 전방 에지(Indirect Calls and Jumps)와 후방 에지(Returns)로 구성된다. 이상적인 CFI는 모든 실행 흐름 가로채기를 방지할 수 있다.

3. Hardware-Assisted Kernel CFI
하드웨어 보조 커널 CFI는 주로 Microsoft Control-Flow Guard(CFG)와 Clang/LLVM CFI를 포함한다. Microsoft CFG는 비트맵 기반 전방 에지 검증 정책을 사용하며, Intel Control-flow Enforcement Technology(CET)를 활용한다. Clang/LLVM CFI는 함수 타입 기반 전방 에지 검증 정책을 사용하며, Intel CET를 활용할 수 있다. FineIBT는 Clang/LLVM CFI와 Intel CET를 기반으로 하지만 호출자 측 검증 정책을 사용한다. 이러한 기술들은 주로 간접 분기에 대한 제한을 강화한다.

4. Intel Control-flow Enforcement Technology (CET)
Intel CET는 Indirect Branch Tracking(IBT)와 Shadow Stack(SS)을 포함한다. IBT는 ENDBR32와 ENDBR64 명령어를 사용하여 간접 호출 및 점프의 유효 타겟 위치를 표시한다. Shadow Stack은 함수 호출 시 보호된 영역에 반환 주소를 저장하고, 함수 복귀 시 스택과 보호된 영역에서 반환 주소를 팝하여 비교한다.

5. 하드웨어 보조 커널 CFI의 문제점
하드웨어 보조 커널 CFI는 소프트웨어 기반 CFI가 간접 분기를 엄격하게 제한하지 못하는 문제를 보완하기 위해 사용된다. 하드웨어 기반 CFI(CET)는 간접 호출이나 점프의 타겟이 ENDBRANCH 명령어로 시작하도록 강제하며, 반환 주소가 정확한 호출 위치와 일치해야 한다. 하지만, 이러한 메커니즘도 완벽하지 않으며 특정 취약점이 존재한다.

6. Page-Oriented Programming (POP)
POP은 새로운 형태의 페이지 수준 코드 재사용 공격으로, 커널 CFI의 약점을 이용한다. POP은 합법적인 코드 페이지와 직접 분기를 활용하여 새로운 제어 흐름을 만든다. 커널 메모리 읽기 및 쓰기 취약점을 사용하여 커널 내 페이지 테이블을 프로그래밍하고, 페이지 수준 가젯을 식별하고 연결하여 강력한 CFI 적용을 우회한다.

7. POP의 단계별 설명
7.1. 페이지 조각화 단계 (Page Carving Stage)
POP 공격의 첫 번째 단계는 가젯과 시스템 호출 후보를 식별하는 것이다. 여기서 가젯은 함수의 일부이며, 시스템 호출 후보는 특정 기능을 수행하는 함수이다. Call 가젯은 시스템 호출 후보를 commit_creds()와 연결하고, NOP 가젯은 불필요한 함수를 분리한다.

7.2. 페이지 스티칭 단계 (Page Stitching Stage)
가젯을 데이터와 연결하여 새로운 제어 흐름을 만드는 단계이다. 가젯의 물리적 페이지를 직접 분기 타겟의 논리 주소에 매핑하고, commit_creds()에 전달되는 인자를 재매핑한다. 이 과정에서 커널 페이지 테이블을 사용하여 모든 프로세스와 커널 스레드가 공유하는 페이지 테이블을 조작한다.

7.3. 페이지 플러싱 단계 (Page Flushing Stage)
페이지 플러싱 단계에서는 TLB(Translation Lookaside Buffer)의 오래된 매핑을 삭제하고 새로운 매핑을 적용한다. 이를 위해 글로벌 비트를 제거하고 일정 시간 동안 대기하여 TLB가 모든 커널 매핑 데이터를 저장하지 못하도록 한다.

7.4. 공격 단계 (Exploitation Stage)
공격 단계에서는 임의의 인자를 사용하여 타겟 시스템 호출을 실행한다. 새로운 제어 흐름이 검증 없이 commit_creds()를 호출하며, 페이지 플러싱 단계가 수행된 동일한 코어에서 실행되어야 한다.

8. 결론 (Conclusion)
결론에서는 커널 CFI의 효과와 약점을 요약하고, POP이 커널 CFI를 우회할 수 있는 새로운 코드 재사용 공격임을 강조한다. POP의 완화는 여전히 개방된 문제이며, Intel HLAT(하이퍼바이저 관리 논리 주소 변환)를 사용한 완화책이 필요하다.
