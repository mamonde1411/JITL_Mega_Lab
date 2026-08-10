
[증상] 본사 PC → 지사 PC ping 실패
[확인함] show ip route에 10.2.20.0 없음
[예상] OSPF 인접성 문제인 것 같음
[시도] network 명령 다시 확인함

[증상]
[확인함]
[예상] 
[시도]

------------------------------------

CSW1(config-if-range)#channel-group 1 mode desirable
Creating a port-channel interface Port-channel 1
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/2, changed state to down
...

CSW1(config-if)#ip add 10.0.0.41 255.255.255.252
                                            ^
% Invalid input detected at '^' marker.

[증상] 포트에 ip 할당이 안됨
[확인함] sh ip int br 이상 없음
[원인] no switchport를 안했음.
[해결] switchport 함.
[정리] po1은 독립적은 설정 객체 channel-group으로 같이 만들어지지만 등등. powerpoint에 정리함. 가져다 쓰기


! ======= ASW-A1 =======
! default route
ip route 0.0.0.0 0.0.0.0 10.0.0.1

! Vlan 99 (management)
int vlan 99
  ip add 10.0.0.4 255.255.255.240
  no shut

만약 ip route을 한다면 이렇게 에러
ASW-A1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.1
                               ^
% Invalid input detected at '^' marker.
	
[문제] L2 스위치에 ip route 0.0.0.0 0.0.0.0 10.0.0.1을 하려함
[확인] 스크립트를 ASW-A1에 붙여넣기 하기 전 claude로 잘 썼는지 확인함.
[원인] L2 스위치라는 것을 잊었고 L2 스위치에 default-gateway 지정하는 것도 익숙하지 않았음.
[해결] ip default-gateway 10.0.0.1로 고침.
[정리] L2 스위치에는 스위치 관리 목적(SSH/SNMP/NTP/Syslog 등)으로 ip주소가 필요함. 관리 목적용 vlan을 만들고 ip 주소를 SVI에 할당함. 통신할 수 있도록 gateway를 정해주는데 L2는 route를 못하므로 route table도 없음. 그래서 ip default-gateway [ip add]으로 설정 해주어야함.

%HSRP-4-DIFFVIP1: Vlan40 Grp 4 active routers virtual IP address 10.6.0.1 is different to the locally configured address 10.6.0.3

DSW-A2(config-if)# standby 4 ip 10.6.0.1

%HSRP-4-DIFFVIP1: Vlan40 Grp 4 active routers virtual IP address 10.6.0.1 is different to the locally configured address 10.6.0.3

[문제] HSRP state가 Speak에서 Standby로 넘어가지 않음
[확인] 에러 메세지가 나와 고칠 수 있었음
[원인] SVI ip add가 10.6.0.1인데 실수로 10.6.0.3으로 지정
[해결] standby 4 ip 10.6.0.1 configure
[정리] 휴먼 에러가 정말 많이 나온다... 빨리 자동화 하고싶다.

DSW-B1: %IP-4-DUPADDR: Duplicate address 10.5.0.1 on Vlan30, sourced by 0000.0C9F.F004

DSW-B2: %IP-4-DUPADDR: Duplicate address 10.5.0.1 on Vlan30, sourced by 0000.0C9F.F003


[문제] 에러 메세지
[확인] 10.5.0.1이 다른 어디에 또 설정되었나 확인해봄. DSW-B1의 int vlan 30, SRV1. 그리고 스크립트를 다시 봄
[원인] DSW-B1:  standby 3 ip 10.5.0.1 DSW-B2:  standby 4 ip 10.5.0.1
DSW-B1을 그룹 3으로 지정해서 둘 다 active가 되고 10.5.0.1로 응답하는 장비가 두개가 되어버려 duplicate 에러
[해결] 
DSW-B1: no int vlan 30후 그룹 4로 다시 만들음.
DSW-B1(config)#no int vlan 30
%LINK-5-CHANGED: Interface Vlan30, changed state to administratively down

%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan30, changed state to down
int vlan 30
DSW-B1(config-if)#  ip add 10.5.0.2 255.255.255.0
DSW-B1(config-if)#  standby version 2
DSW-B1(config-if)#  standby 4 ip 10.5.0.1
DSW-B1(config-if)#  standby 4 priority 105
DSW-B1(config-if)#  standby 4 preempt



------------------------------------
## 소재 후보
1. L3 EtherChannel — Port-channel이 L2로 생성되는 문제
   - 상태: 실험 미완. A안/B안 로그 필요
2. SVI 개념 정리 — L2 vs L3 스위치
   - 상태: 이해 완료. 로그 없음, 개념 글로 쓸 것
3. ip default-gateway vs ip route
   - 상태: 에러 출력 있음 (% Invalid input)
4. Native VLAN과 allowed vlan의 관계
   - 상태: show interfaces trunk 출력 캡처 필요
5. CAPWAP 캡슐화 — AP 포트가 access인 이유
6. HSRP 그룹 번호 불일치 → Duplicate address 로그
   - 증상: %IP-4-DUPADDR
   - 단서: 소스 MAC이 0000.0C9F.F0XX (HSRP 가상 MAC, 끝자리=그룹 번호)
   - 원인: 양단 그룹 번호 불일치 → 둘 다 Active
   - 로그 확보: 있음 ✓
7. Extension 쓰니 이렇게 편할수가...