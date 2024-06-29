### eBPF ELFs: JMPing Through the Windows

#### 1. 소개 (Introduction)

**발표자 소개**
- **Richard Johnson**
  - Trellix의 선임 수석 보안 연구원
  - 취약점 연구 및 리버스 엔지니어링 전문가
  - Fuzzing IO 소유자
  - 고급 퍼징 및 크래시 분석 교육 제공
  - 연락처: rjohnson@fuzzing.io, Twitter: @richinseattle
  - Trellix 인턴들에게 감사: Kasimir Schulz (@abraxus7331), Andrea Fioraldi (@andreafioraldi)

#### 2. 개요 (Outline)

- eBPF의 기원과 응용 (Origins and Applications of eBPF)
- Windows용 eBPF의 아키텍처 및 설계 (Architecture and Design of eBPF for Windows)
- API 및 인터페이스의 공격 표면 (Attack Surface of APIs and Interfaces)
- 퍼징 방법론 및 결과 (Fuzzing Methodology and Results)
- 결론 (Concluding Thoughts)

#### 3. eBPF란 무엇인가 (What is eBPF)

**eBPF 정의**
- eBPF는 가상 CPU 아키텍처 및 VM으로, 원래는 네이티브 커널 모듈의 대안으로 개발된 “Berkley Packet Filter”를 확장한 것이다.
- eBPF 프로그램은 C 언어에서 LLVM을 통해 가상 CPU 명령어로 컴파일되며, 에뮬레이트된 모드 또는 JIT 실행 모드에서 실행될 수 있다.
- 로더의 일부로 정적 검증기를 포함하여 실행된다.
- eBPF 프로그램은 높은 속도의 네트워크 패킷 검사 및 수정, 프로그램 실행을 위해 설계되었다.

#### 4. eBPF의 기원 (Origins of eBPF)

**역사적 배경**
- 1992년에 네트워크 패킷을 필터링하기 위한 방법으로 Berkeley Packet Filter 기술이 개발되었다.
- 대부분의 Unix 스타일 운영 체제에 재구현되었으며 사용자 영역으로도 포팅되었다.
- 대부분의 사용자는 tcpdump, wireshark, winpcap, npcap 등을 통해 BPF와 상호 작용했다.
- “dst host 10.10.10.10 and (tcp port 80 or tcp port 443)”과 같은 필터 문자열을 제공하면 자동으로 고성능 BPF 필터로 컴파일된다.
- 이 오래된 BPF 인터페이스를 cBPF 또는 Classic BPF라고 한다.

**eBPF 확장**
- 2014년 12월, Linux 커널 3.18 버전에서 bpf() 시스템 호출을 통해 eBPF API가 추가되었다.
- eBPF는 BPF 명령어를 64비트로 확장하고, eBPF 프로그램과 사용자 공간 데몬 간에 공유할 수 있는 지속 가능한 데이터 구조 배열인 BPF 맵 개념을 추가했다.
- eBPF는 사용자가 일반적인 목적의 프로그램을 작성하고 커널 제공 API를 호출할 수 있도록 원래의 BPF 개념을 확장했다.

#### 5. eBPF의 응용 (Applications of eBPF)

**Linux eBPF 응용 프로그램**
- 다양한 프로젝트에서 eBPF가 사용되고 있으며, 더 많은 프로젝트 목록은 [eBPF 프로젝트](https://ebpf.io/projects)에서 확인할 수 있다.

**이전 eBPF 연구**
- Evil eBPF – Jeff Dileo, DEF CON 27 (2019): BPF_MAPS를 IPC로 사용하는 방법 설명, ROP 체인 주입 기술 설명.
- With Friends like eBPF, who needs enemies – Guillaume Fournier, et al, BH USA 2021: eBPF 루트킷 데모, 시스템 호출 반환 및 사용자 공간 API 후킹, HTTPS 요청 패킷 교체를 통한 정보 유출.
- Extra Better Program Finagling (eBPF) – Richard Johnson, Toorcon 2021: 프로세스 생성 가로채기 후킹, 공격자 제어 라이브러리로 libc 미리 로드.

#### 6. Windows용 eBPF의 타임라인 (eBPF for Windows Timeline)

**주요 사건**
- 2021년 5월: eBPF for Windows 발표
- 2021년 8월: Microsoft, Netflix, Google, Facebook, Isovalent이 eBPF Foundation 설립 발표
- 2021년 11월: libbpf 호환성 추가 및 추가 BPF_MAPS 지원
- 2022년 2월: Microsoft가 Cillium L4LB 로드 밸런서를 Linux에서 Windows로 포팅하는 노력 발표

#### 7. Windows용 eBPF의 아키텍처 (eBPF for Windows Architecture)

**구성 요소**
- Linux eBPF 시스템과 달리, Windows 버전은 여러 구성 요소로 분할되어 있으며 IO Visor uBPF VM 및 PREVAIL 정적 검증기와 같은 오픈 소스 프로젝트를 포함한다.
- Windows용 eBPF는 네트워크 패킷의 인스펙션 및 수정을 수행할 수 있으며, 이식성을 위한 libbpf API 호환 레이어를 노출한다.

#### 8. Windows에서 eBPF 프로그램 생성 (Creating eBPF Programs on Windows)

**컴파일 과정**
- Windows에서 eBPF 프로그램은 C 소스에서 LLVM을 사용하여 컴파일된다.
- 결과 출력물은 ELF 객체로, eBPF 바이트코드가 ELF 섹션에 저장된다.

**실제 예시**
- 특정 패킷을 드롭하는 eBPF 프로그램 예제:

```c
SEC("xdp_drop")
int xdp_drop_func(struct xdp_md *ctx) {
    return XDP_DROP;
}
```

#### 9. Windows용 eBPF의 보안 모델 (eBPF for Windows Security Model)

**보안 보장**

Windows용 eBPF는 커널에서 서명되지 않은 코드를 실행할 수 있게 한다.
현재 DACL은 관리자가 신뢰된 서비스와 상호 작용하거나 IOCTL을 통해 직접 드라이버와 상호 작용할 수 있도록 요구한다.
eBPF 바이트코드가 서비스에 의해 로드될 때 정적 검증기가 프로그램이 특정 명령어 수 내에서 종료되며, 컴파일 시 지정된 메모리 범위를 벗어나지 않도록 보장한다.
VM 엔진은 x64로 JIT 코드를 생성하고 네이티브 명령어를 커널에 전달하거나 디버그 모드에서 eBPF 바이트코드를 해석된 모드로 실행할 수 있다.

#### 10. eBPF 공격 시나리오 (eBPF for Windows Attack Scenarios)

**가능한 공격 시나리오**

- 서명되지 않은 eBPF 프로그램을 로드하여 관리자 권한으로 코드 실행
- RPC API 구현 오류를 통한 신뢰된 서비스에서 코드 실행
- 정적 검증기 또는 JIT 컴파일러 버그를 통한 코드 실행
- IOCTL 구현 오류를 통한 커널에서 코드 실행
- shim 훅 구현 오류를 통한 커널에서 코드 실행

#### 11. eBPF4Win API (ebpfapi.dll)

**구성 요소**

- ebpfapi.dll은 프로그램 로드 및 언로드, 맵 생성 및 삭제 등을 포함하는 API를 제공하며, bpftool.exe 및 netsh 인터페이스를 통해 노출된다.

#### 12. eBPF4Win 서비스 (ebpfsvc.dll)

**RPC 기반 API**

- ebpfsvc.dll은 PREVAIL 및 uBPF 코드를 포함하며, RPC 기반 API를 노출한다.
- `verify_and_load_program` API는 프로그램 검증 및 로드를 위한 단일 API를 제공한다.

#### 13. PREVAIL 정적 검증기 (PREVAIL Static Verifier)

**설명**

- PREVAIL은 폴리노미얼 런타임 EBPF 검증기로, 추상 해석 계층을 사용하여 더 빠르고 정밀한 검증을 제공한다.
- Linux 정적 검증기보다 빠르고 정밀하게 설계되었으며 MIT 및 Apache 이중 라이선스를 사용한다.

#### 14. uBPF (Userspace BPF)

**설명**

- uBPF는 eBPF 바이트코드 인터프리터 및 JIT 엔진의 독립적인 재구현으로, BSD 라이선스를 사용하며 사용자 또는 커널 컨텍스트에서 실행될 수 있다.

#### 15. 퍼징 (Fuzzing)

**퍼징 방법론**

- ELF 로딩 API를 퍼징하기 위해 PREVAIL 검증기 코드를 Linux에서 퍼징하고, ebpfapi.dll API를 libfuzzer로 직접 하네스하여 퍼징하였다.
- WTF 퍼저 도구를 사용하여 ebpfsvc.dll을 퍼징하였다.

#### 16. 결론 (Concluding Thoughts)

**요약**

- eBPF는 현대 운영 체제에서 텔레메트리 및 인스펙션을 위한 흥미로운 기술이다.
- Microsoft는 uBPF 및 PREVAIL 오픈 소스 프로젝트를 사용하여 eBPF 구현의 기초를 제공했다.
- 드라이버 및 사용자 로더 코드의 퍼징을 통해 몇 가지 중요한 취약점을 발견하였다.
- Microsoft는 2021년 5월 이후로 많은 버그를 수정하였으며, eBPF Foundation의 지원을 받아 eBPF는 데스크톱, 서버 및 클라우드 환경에서 핵심 기술로 자리 잡고 있다.
- Trellix는 커뮤니티에 도움이 되는 선제적인 취약점 연구에 전념하고 있다.
