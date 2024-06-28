# Diving into Windows Remote Access Service for Pre-Auth Bugs

1. 소개 (Introduction)
이 논문은 Windows Remote Access Service (RAS)의 원격 인증 전(pre-auth) 버그를 탐구한다. RAS는 클라이언트에게 원격 액세스를 제공하는 서비스로, Direct Access, VPN, 라우팅 및 프록시를 포함한다. RAS는 수많은 커널 드라이버와 사용자 모드 서비스로 구현되며, Windows Server 및 Azure Cloud에서도 사용된다.

2. PPTP (Point-to-Point Tunneling Protocol)
PPTP는 오래된 VPN 프로토콜로, 두 개의 채널(컨트롤 채널과 데이터 채널)을 사용한다. 컨트롤 채널은 TCP 1723 포트를 사용하고, 데이터 채널은 GRE를 사용하여 PPP(점대점 프로토콜)를 캡슐화한다.

2.1 주요 구성 요소 및 공격 표면 (Components & Pre-auth Attack Surfaces)
raspptp.sys: 메인 컨트롤 채널을 처리하고 콜 상태를 유지한다. TCP 포트 1723과 GRE 패킷을 처리한다.
PPP 엔진: rasppp.dll은 사용자 모드에서 PPP 프로토콜을 처리한다.
인증 프로토콜: raseap.dll, rastls.dll, raschap.dll 등이 있다.

2.2 취약점 사례 연구 (Case Study)

```
DWORD LcpMakeConfigResult(...) {
    // src와 dst 옵션 버퍼 크기는 동일함
    PPP_OPTION* pSrcOption; // 원격 클라이언트로부터 받음
    PPP_OPTION* pDstOption; // 출력 옵션

    while (lSrcLength > 0) {
        memcpy(pDstOption, pSrcOption, pSrcOption->length);
        pSrcOption += pSrcOption->length;
        pDstOption += pDstOption->length;
        lSrcLength -= pSrcOption->length;
    }
}
```

위 코드에서 pSrcOption->length가 2보다 작으면, pDstOption->length는 초기화되지 않은 데이터가 되고, pDstOption이 버퍼 끝을 초과할 수 있다.

3. SSTP (Secure Socket Tunneling Protocol)
SSTP는 HTTPS를 통해 PPP 트래픽을 캡슐화하는 프로토콜로, Microsoft에서 개발되었다. Azure Point-to-Site VPN에서 사용된다.

3.1 주요 구성 요소 및 공격 표면 (Components & Pre-auth Attack Surfaces)
rassstp.sys: SSTP 메시지를 처리하고 SSTP 콜의 라이프 사이클을 유지한다.
sstpsvc.dll: 클라이언트와 커널 드라이버 간의 통신을 유지한다.

3.2 취약점 사례 연구 (Case Study)

```
void __fastcall RadiusProxyEngine::onReceive {
    Attributes* pAttr = packet->pAttributes;
    DWORD dwAttrCount;
    // DoS 위치는 어디인가?
    for (; pAttr < pEnd; pAttr += pAttr->length) {
        dwAttrCount += pAttr->length;
    }
}
```

4. L2TP (Layer 2 Tunneling Protocol)
L2TP는 주로 IPsec 터널을 먼저 설정한 후 사용된다. IPsec 터널이 설정된 후, UDP 포트 1701을 통해 서버와 통신하며 UDP 페이로드는 IPsec에 의해 암호화된다.

4.1 주요 구성 요소 및 공격 표면 (Components & Pre-auth Attack Surfaces)
rasl2tp.sys: L2TP 제어 채널과 데이터 채널을 처리하고, 터널 및 콜의 라이프 사이클을 관리한다.
IKE: 인증 프로토콜로, IPsec 터널 설정에 사용된다.

4.2 취약점 사례 연구 (Case Study)

```
void __fastcall RadiusProxyEngine::onReceive {
    Attributes* pAttr = packet->pAttributes;
    DWORD dwAttrCount;
    // DoS
    for (; pAttr < pEnd; pAttr += pAttr->length) {
        dwAttrCount += pAttr->length;
    }
}
```

이 코드에서 pAttr->length가 0이면 무한 루프가 발생하여 DoS 공격이 가능하다.

5. IKE (Internet Key Exchange)
IKE는 IPsec의 보안 터널을 설정하기 위한 인증 프로토콜로, IKEv1과 IKEv2 두 가지 버전이 있다. Windows L2TP VPN은 터널 설정을 위해 IKEv1을 사용할 수 있으며, Azure Site-to-Site VPN에서는 IKEv2를 사용한다.

5.1 주요 구성 요소 및 공격 표면 (Components & Pre-auth Attack Surfaces)
ikeext.dll: IKE 패킷의 파싱과 처리를 담당한다.
UDP 포트 4500: IKEv1용
UDP 포트 500: IKEv2용

5.2 취약점 사례 연구 (Case Study)

```
vpn packet agilevpn.sys AgileVpnProcessPackets {
    // NDIS로 패킷을 인디케이트, 보호되지 않음
    NdisMCoIndicateReceiveNetBufferLists(NdisVcHandle, Alignment, 1u, 0);
}
```

6. 결론 (Conclusion)
Windows RAS VPN 프로토콜의 구현은 복잡하며, 커널 드라이버와 사용자 모드 서비스로 구성되어 있어 원격 인증 전(pre-auth) 버그를 찾기에 좋은 타겟이 된다. 이 논문에서는 레이스 조건 및 수동 감사를 통해 다수의 RCE 버그를 발견했다.

이 논문은 버그 헌팅의 기술적 접근 방식과 생각을 공유하며, 미래의 연구 방향과 교훈을 제공한다. 연구자들이 윈도우 원격 프로토콜을 연구할 때 레이스 조건을 시도하고, 퍼징과 수동 감사를 병행하는 것이 중요하다고 강조한다.
