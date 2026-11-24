# OpenNebula Private Cloud 구축 사전 기술 분석

- 조사 기준일: 2026-11-24
- 개정: v2 — 외부 검토본(별도 작성 분석서)과 교차 검증 후 병합. 차이가 난 항목은 공식 문서와 소스(`one` release-7.4.1, `one-deploy` release-2.0.0)로 재확인해 반영했다.
- 조사 범위: 공식 문서(docs.opennebula.io 7.4), 공식 GitHub(OpenNebula/one `release-7.4.1` 태그, OpenNebula/one-deploy `release-2.0.0`/master, terraform-provider-opennebula), 공식 패키지 저장소(downloads.opennebula.io), OS·Ceph 공식 사이트
- 실행 범위: 조사·설계·문서화만 수행. 설치·설정 변경·패키지 설치·실측 없음.
- 표기 규칙
  - **[문서]** 공식 문서에 명시
  - **[코드]** 공식 저장소 소스에서 확인(문서 미기재)
  - **[추론]** 위 근거로부터의 판단(사실 아님)
  - **[미확인]** 근거 확보 실패 → PoC 검증 필요
  - 출처 번호 `[S#]`는 12장 참조

---

## 1. 최종 결론

| 항목 | 결론 | 근거 |
|---|---|---|
| OpenNebula 버전 | **CE 7.4.1** (CE로 설치 가능한 최신 패치), OneDeploy에서 `one_version: '7.4.1'`로 패치 고정. 7.4.2는 EE 전용 | [S1][S2][S3] |
| OS | **Ubuntu Server 24.04 LTS (minimized 설치)** | OpenNebula 7.4 지원 + OneDeploy "tested and verified" + 표준지원 2029-05 [S4][S20][S30] |
| 아키텍처 | **A안: 물리 3대, 관리·Compute 통합, Front-end 3중화(Raft, 호스트 OS에 직접 설치)** | 공식 Raft HA 3대 권장 [S7], 순환 의존 없음 |
| 스토리지 | **외부 NFS(이중화 NAS 권장)** — Image/System datastore 공유 | Front-end HA가 공유 파일시스템을 요구 [S7], `-m` 자동 재배치는 공유 스토리지만 가능 [S8] |
| 네트워크 | **Linux Bridge + 802.1Q VLAN(프로젝트별 VLAN)** | 추가 데몬·멀티캐스트 불필요, OVS는 Security Group 미지원 [S13][S14][S16] |
| 자동 설치 | **OneDeploy release-2.0.0** (CE: `one_token` 미지정) | [S20][S21] |
| VM HA | `host_error` hook + `fence_host.sh`(사용자 작성, fence-agents로 IPMI/Redfish 호출) | [S8][코드] |
| IaC | Terraform Provider v1.5.0 + GitLab CI/CD (7.4 호환성 PoC 필요) | [S25] |
| 서비스 범위 | **CLI/XML-RPC 중심 최소 구성.** OneDeploy `features`의 `gui`(FireEdge/Sunstone), `gate`, `flow`, `form`, `ks`, `prometheus`, `ceph`, `evpn`을 모두 `false`로 시작 | 요구사항 "불필요 서비스 배제" + CE 7.4.1의 FireEdge 보안 공백 [S3]. 기본값은 gate/flow/form/gui가 `true` [S21b] |

**전제 조건(중요)**

1. 서버당 RAM 32GiB에서 A안은 **기술적으로 구성 가능하지만 권장하지 않음**. 공식 Front-end 최소 RAM 16GiB [S5]를 노드마다 확보하면, 1대 장애를 견디는 VM 용량은 클러스터 전체 **≤ 약 29GiB**로 줄어든다(4장 계산). **A안 채택 시 서버당 64GiB 이상을 권장한다.** 32GiB를 유지해야 하면 B안(관리 1대 + Compute 3대, Front-end 단일)이 VM 용량에서 유리하다. 대신 관리 계층 HA는 없다.
2. NFS 서버는 물리 3대 밖의 별도 장비다. NFS 자체를 이중화하지 않으면 **NFS가 SPOF**가 된다.
3. CE 7.4.1에는 7.4.2(EE)의 보안 수정이 없다(FireEdge XSS, 인증 padding-oracle 등)[S3]. **FireEdge(Sunstone)는 배포하지 않는다.** UI가 필수가 되면 공개 CE 수정판이 나오거나 영향이 없다는 증적을 확보할 때까지 도입을 보류한다. XML-RPC(2633)는 **관리망/VPN 안에서만 노출**한다.

---

## 2. 검증된 최신 버전 및 지원 범위

### 2.1 버전 현황

| 계열 | 최신 패치(EE) | CE 저장소 최신 | 비고 | 출처 |
|---|---|---|---|---|
| 7.4 "Helix" | 7.4.2 (2026-11-12 공지, EE) | **7.4.1** (2026-09-11) | 공식 사이트: "Latest Enterprise Stable: 7.4.2 LTS", "Latest Community Stable: 7.4" | [S1][S2][S3][S6] |
| 7.2 | 7.2.1 (EE) | 7.2.0 | LTS 여부 [미확인] | [S2] |
| 7.0 | 7.0.2 (EE) | 7.0.1 | LTS 여부 [미확인] | [S2] |
| 6.10 | 6.10.5 LTS (EE) | 6.10 | — | [S6] |

- CE 저장소 디렉터리: 7.0.0, 7.0.1, 7.2.0, 7.4.0, 7.4.1 (7.0.2 / 7.2.1 / 7.4.2 없음) [S2]
- `/repo/7.4/Ubuntu/24.04/pool/opennebula/`에 `opennebula_7.4.1-1` 존재 [S2]
- GitHub `OpenNebula/one`에는 Release 객체가 없고 태그만 있다. 최신 태그는 `release-7.4.1`이며 `release-7.4.2` 태그는 없다 [S1]

### 2.2 CE / EE 구분

| 항목 | CE | EE | 출처 |
|---|---|---|---|
| 코드베이스 | 동일("There is no separate Enterprise code base") | 동일 + 유지보수 패키지 | [S9] |
| 유지보수 릴리스 | 공개 유지보수 릴리스 없음. "critical bug fix" 패치만 | LTS 최소 36개월, 단기 릴리스 9개월 | [S10] |
| Front-end Raft HA | 사용 가능(공개 소스, 문서 에디션 표기 없음) | 동일 | [S7][코드] |
| host_error hook / fencing 연동 | 사용 가능(fence 스크립트는 사용자 작성) | 동일. EE 전용 fencing 기능 [미확인/발견 안 됨] | [S8][코드] |
| OneDeploy | `one_token` 미지정 시 CE 저장소 사용 | `one_token` 지정 시 EE 저장소 | [S21][코드] |
| Terraform Provider | 사용 가능(Apache 2.0 공개) | 동일 | [S25] |
| Prometheus 연동 | CE 7.4.1 pool에 `opennebula-prometheus*` 존재 | 동일 | [S2] |
| EE 전용(7.4 문서 표기) | — | Veeam Backup, NetApp/Everpure SAN datastore, WHMCS, Support Utilities, Long-term forecasting, NVIDIA NICo | [S11][S12] |

### 2.3 공식 지원 플랫폼 (7.4 Platform Notes [S4])

| 구분 | 지원 목록 |
|---|---|
| Front-end / KVM Node OS | RHEL 9·10, AlmaLinux 9·10, Rocky 9·10("Available through Red Hat packages"), Ubuntu 24.04·26.04 LTS, Debian 12·13, SLES 15·16, openSUSE 16 |
| KVM/libvirt/QEMU/커널 | "Support for version included in the Linux distribution". 별도 버전 고정 없음 |
| DB | SQLite(기본), MariaDB/MySQL(배포판 버전) |
| Ceph | Reef 18.2.x, Squid 19.2.x |
| NFS | v4.2 |
| 미지원 | Ubuntu 22.04, RHEL 8 (7.4 목록에 없음) |
| 변경 | RHEL/AlmaLinux 9·10 KVM 노드는 modular libvirt daemon 사용 [S12a] |

### 2.4 보안·알려진 결함 (7.4)

| 항목 | 상태 | 출처 |
|---|---|---|
| 7.4.2 보안 수정: FireEdge Node.js(Brotli), Lodash CVE-2025-13465 / CVE-2026-4800, Restic 비밀번호 노출, 이미지 템플릿 MD5/SHA1 명령 주입, Sunstone 저장형 XSS, 서버 인증 padding-oracle / target-user bypass, FireEdge VNC 오접속 | **EE 7.4.2에서만 수정 → CE 7.4.1 미반영.** 대응: FireEdge 미배포, API 관리망 한정 | [S3] |
| 7.4.2 기능 결함: "oned crashing when a KVM Host reports a NUMA node without cores"(#8133), "loss of VM firewall rules when a Security Group update is immediately followed by a commit"(#8143) | **EE 전용 수정 → CE 7.4.1 미반영.** 후보 CPU로 호스트 등록·모니터링을 시험한다. Security Group 변경 후 실제 차단 여부를 검사한다. 프로젝트 간 기본 격리는 VLAN + 상위 라우터 ACL에 둔다 | [S3] |
| CVE-2026-84165 (`one.vm.exec`로 타 사용자 VM 명령 실행, qemu-agent 필요, CVSS 4.0 8.7) | **영향 범위 미확인.** INCIBE 공지에 "Affected Resources: OpenNebula 7.4"와 "all versions prior to 7.4"가 함께 적혀 있어 서로 모순된다. 해결책은 "Update to OpenNebula version 7.4". 7.4.1 적용 여부는 공급사 확인이 필요하다. 그 전까지 qemu-agent 사용 VM과 VM exec 권한(ACL)을 제한한다 | [S3b] |
| 7.4.1 수정: HA 리더 선출 시 ARP 갱신 | "on RHEL-based distributions"로 한정 → **선정 OS(Ubuntu)와 무관** | [S3a] |
| 7.4.1 수정: NVRAM 사용 VM의 live storage migration, `TM_MAD_SYSTEM=ssh`일 때 NFS automount, Ceph 백업 재시도 | CE 7.4.1에 포함. 회귀 시험 범위에 넣는다 | [S3a] |
| 업그레이드 시 사용자 fencing 스크립트(`hooks/ft/fence_host.sh`) 덮어쓰기 | 7.4.1에서 수정. 7.4.1부터 fencing은 `fence_host.sh.example`로 `fence_host.sh`를 만들어 활성화 | [S3a][S12b] |
| `onevm updateconf`를 `--append` 없이 쓰면 Secure Boot 설정 삭제 | 알려진 이슈 | [S12b] |
| LVM 백업 스냅샷 잔존 → 마이그레이션 시 데이터 손실 | 7.4.2(EE)에서 수정. 본 설계는 LVM 미사용 | [S12b] |

### 2.5 지원 종료·업그레이드 경로

| 항목 | 내용 | 출처 |
|---|---|---|
| CE EOL 날짜 | 공식 표 없음 [미확인] | [S10] |
| 업그레이드 | "You can upgrade to OpenNebula 7.4 from any previous installation of OpenNebula EE or CE". 6.0.x 이상만 해당 | [S12c] |
| 마이그레이터 | `onedb` / `onecfg`(`opennebula-migration`)는 CE와 EE 모두 제공 | [S12c] |
| 규칙 | 중간 버전 가이드를 모두 따를 것. **유지보수 버전 업그레이드에도 `onecfg upgrade` 실행** | [S12a][S12c] |
| HA 업그레이드 | 공식 HA 전용 절차가 있다. Follower부터 중지하고 Leader는 마지막에 중지 → Leader 업그레이드 → `onedb backup` → Follower 업그레이드 후 `onedb restore -f`로 복제. 상세는 10.2 | [S12d] |
| 롤백 | **공식 절차 있음**("Restoring Previous Version"). 7.4가 설치된 상태에서 `onedb restore -f`로 DB 복원 → 7.4 제거 후 이전 버전 재설치 → `/etc/one` 백업 복원 | [S12e] |

---

## 3. 권장 OS 및 선정 근거

| 기준 | Ubuntu 24.04 | Rocky 9 | AlmaLinux 9 | Debian 13 |
|---|---|---|---|---|
| OpenNebula 7.4 공식 지원 [S4] | O | O(RH 패키지) | O | O |
| OneDeploy "tested and verified" [S20] | **O** | **X — precheck에서 실패.** `integrity.yml`의 `ansible_distribution` 허용 목록이 AlmaLinux / Debian / openSUSE Leap / RedHat / SLES / Ubuntu뿐이다. 저장소 URL을 바꿔도 assert 단계에서 막힌다 [코드: release-2.0.0] | O(RHEL 9 파생, NetworkManager 필수) | △ — precheck는 허용, 검증 목록엔 없음 |
| 커널 / QEMU / libvirt (2026-11-24 저장소) [S31] | 6.8.0 / 8.2.2 / 10.0.0 | 9.9 = Alma 동일 예상 [미확인] | 5.14.0-741 / 10.1.0 / 11.10.0 | 6.12.118 / 10.0.13 / 11.3.0 |
| 표준 / 보안 지원 종료 [S30] | 2029-05 / ESM 2034-05 | 2027-05-31 / 2032-05-31 | 2027-05-31 / 2032-05-31 | 2028-08-09 / LTS 2030-06-30 |
| 최소 설치 | `ubuntu-server-minimal` 소스 [S32] | Minimal Install | Minimal Install | netinst |
| 공식 최소 RAM | 1.5GB(ISO 설치) [S33] | [미확인] | [미확인] | [미확인] |
| 실측 RAM | **미측정 — PoC에서 측정** | 미측정 | 미측정 | 미측정 |

**선정: Ubuntu Server 24.04 LTS (minimized)**

1. OpenNebula 7.4와 OneDeploy가 모두 지원·검증한 OS다. 4개 후보 중 Ubuntu 24.04와 AlmaLinux 9만 해당한다.
2. 표준 지원이 AlmaLinux 9보다 약 2년 길다(2029-05 vs 2027-05).
3. OneDeploy 문서가 플레이북 실행 OS로 Ubuntu 24.04/22.04를 권장한다 [S20a].
4. 주의할 점:
   - minimized 이미지는 "not intended to be comfortable to use at the command line"이다 [S32]. 운영 도구(`vim`, `less` 등)를 따로 설치해야 한다.
   - netplan 0.105 이상이 필요하다 [S20].
   - Ubuntu 26.04도 OpenNebula 7.4 지원 목록에 있지만 OneDeploy 검증 목록에는 없으므로 제외했다.

---

## 4. 최소/권장 서버 사양

### 4.1 공식 수치 (실측 아님)

| 구성요소 | 공식 수치 | 출처 |
|---|---|---|
| OpenNebula Front-end | RAM 16GiB, 디스크 80GiB, CPU 수 미기재 | [S5] (7.4) |
| Front-end (참조 아키텍처 r2.4, 2024-06, 6.x 시기) | 8코어, 16GB, 200GB, NIC 2 | [S34] |
| KVM 호스트 | 7.4 문서에 최소 사양 없음. 참조 아키텍처: 코어당 RAM 1GB 이상 | [S34] |
| Ubuntu Server 24.04 | RAM 1.5GB 최소 / 3GB 이상 권장, 디스크 5GB 최소 / 25GB 이상 권장 | [S33] |
| Ceph OSD | `osd_memory_target` 기본 4GB, 2GB 미만 비권장 | [S35] |
| Ceph MON | RAM 5GB 이상, 2코어 | [S35] |
| Ceph 네트워크 | 10Gb/s 이상 | [S35] |

### 4.2 측정이 필요한 값 (임의 산정하지 않음)

| 항목 | 상태 |
|---|---|
| oned + MariaDB + FireEdge + monitord 실사용 RAM/CPU | **[미확인] PoC 실측 필요** |
| VM당 QEMU 오버헤드 | **[미확인] PoC 실측 필요** |
| Linux bridge / VLAN 오버헤드 | **[미확인]** 공식 수치 없음 |
| 모니터링 부하 | 공식 수치 없음. 주기: `MONITORING_INTERVAL_HOST` 기본 180s, `MONITOR_VM` 30s [S36] |

### 4.3 권장 사양 (A안 기준)

| 항목 | 최소(32GiB 시나리오) | 권장 | 근거 |
|---|---|---|---|
| CPU | 하드웨어 가상화(VT-x/AMD-V), 3대 동일 CPU 세대 | 동일 모델 3대, 16코어 이상 | 동일 사양 권장 [S7]. 라이브 마이그레이션 CPU 호환 [S17] |
| RAM | 32GiB | **64GiB 이상** | 4.4 계산 |
| OS 디스크 | 80GiB 이상 SSD RAID1 | 240GiB 이상 SSD RAID1 | Front-end 80GiB [S5] + OS. 로그·DB 여유는 [추론] |
| VM 디스크 | NFS(외부) | NFS(외부, 이중화 컨트롤러) | 7장 |
| NIC | 10GbE 2포트 + BMC 전용 1GbE | 10/25GbE 4포트(2×bond) + BMC 전용 | 참조 아키텍처: 10Gbps 이상, 서비스망과 VM망 분리 [S34] |

### 4.4 RAM 예산 — 서버당 32GiB 기준 시나리오

계산식은 공식 최소값만 사용한다. 하이퍼바이저, `RESERVED_MEM`, 네트워크·스토리지 오버헤드는 **실측 전이라 0으로 두었다.** 따라서 아래 값은 **상한**이다.

| 구성 | 노드당 RAM | OS(공식 최소) | 관리 서비스 | Workload 상한/노드 | 클러스터 합계 상한 | **1대 장애 시 수용 상한** |
|---|---|---|---|---|---|---|
| A (3대 통합, FE 3중화) | 32 | 1.5 | FE 16 [S5] | ≤14.5 | ≤43.5 | **≤29**(2노드분) |
| A @64GiB | 64 | 1.5 | FE 16 | ≤46.5 | ≤139.5 | **≤93** |
| B (관리 1 + Compute 3) | Compute 32 | 1.5 | Compute 0, 관리 서버 별도 ≥16 | ≤30.5 | ≤91.5 | **≤61** |
| C (3대, FE VM 3개) | 32 | 1.5 호스트 + FE VM 게스트 OS | FE VM 16 + 게스트 OS | <14.5 | <43.5 | **<29** |
| (참고) Ceph HCI 추가 | 32 | 1.5 | FE 16 + MON 5 + OSD 4 이상 | ≤5.5 | — | **비현실적** [S35] |

> Front-end 16GiB는 단독 Front-end의 공식 요구치다. 통합 노드에서 실제로 덜 쓰는지는 **실측해야 한다.** 실측 결과로 이 값을 줄이는 것은 공식 기준을 벗어나는 운영 판단이므로 기록해 두어야 한다.

---

## 5. 3대·4대 아키텍처 비교

| 항목 | A. 3대 통합(FE on host) | B. 관리 1 + Compute 3 | C. 3대 + FE VM 3개 |
|---|---|---|---|
| 관리 계층 HA | **O** — Raft 3대, 1대 장애 허용 [S7] | **X** — 단일 FE | O — Raft 3대 |
| Front-end 위치 | 각 호스트 OS(bare metal) | 전용 서버 | 호스트별 libvirt 도메인. OpenNebula가 관리하지 않음, `autostart` [S22] |
| 공식 근거 | "Front-end … can also be used as a workload Cluster node simultaneously" [S5] | 표준 구성 | OneDeploy `infra` 플레이북 [S22] |
| 순환 의존 | 없음 | 없음 | OpenNebula 밖 libvirt로 회피. 단 fencing 시 FE VM도 함께 종료 [추론] |
| 노드 1대 장애 | VM 재배치 + FE 2/3 쿼럼 유지 | Compute 장애 시 VM 재배치 | A와 같음 |
| 관리 노드 장애 | 다른 FE가 리더 승계 | **API·스케줄러·VM 자동복구 중단**. 실행 중 VM은 계속 동작 [추론]. 백업으로 FE 재구축 | 리더 승계 |
| 관리 오버헤드 | FE × 3 | FE × 1 | FE × 3 + 게스트 OS × 3 |
| 운영 복잡도 | 중 | **낮음** | 높음(VM 계층 추가) |
| SPOF | NFS, 스위치(단일 시) | **FE**, NFS, 스위치 | NFS, 스위치 |
| 추가 하드웨어 | NFS 장비 | 서버 1대 + NFS | NFS 장비 |
| 32GiB에서 1대 장애 시 VM 수용 상한 | ≤29GiB | ≤61GiB | <29GiB |
| 판정 | **선정(64GiB 이상 조건)** | 32GiB 고정 시 대안 | 배제(오버헤드·복잡도가 가장 큼) |

**C안 배제 사유:** A안과 같은 RAM을 쓰면서 FE VM의 게스트 OS, 부팅 순서, 장애 의존성 계층이 더 생긴다. 공식 예시 `inventory/infra.yml`도 FE가 2대뿐이라 문서의 3대 최소와 맞지 않는다 [S22][코드].

### 5.1 부팅 순서·순환 의존 분석

| 단계 | A안 | C안 |
|---|---|---|
| 1 | NFS 서비스 가동(외부) | NFS 가동 |
| 2 | 호스트 부팅 → NFS 마운트 | 호스트 부팅 → libvirt → FE VM autostart |
| 3 | MariaDB → `opennebula` | FE VM 안에서 NFS 마운트 → MariaDB → `opennebula` |
| 4 | 2/3 노드 가동 시 리더 선출 → VIP 할당(`raft/vip.sh`) | 같음 |
| 5 | 리더에서 OneFlow / OneGate / OneForm 기동(`vip.sh`) [코드] | 같음 |
| 위험 | 3대 동시 정전 후 1대만 부팅되면 쿼럼 없음 → 쓰기 불가 [추론: Raft 과반] | FE VM이 OpenNebula 관리 VM이면 순환 의존 → **OpenNebula가 관리하는 VM으로 FE를 두면 안 됨** [추론] |

---

## 6. CPU·RAM·스토리지 오버헤드 분리

| 계층 | 구성요소 | 공식 수치 | 실측 |
|---|---|---|---|
| 관리 서비스 | oned(스케줄러 내장 [코드]), MariaDB, FireEdge, monitord, OneFlow / OneGate(리더만) | FE 전체 16GiB, 80GiB [S5] | 미측정 |
| Hypervisor | libvirtd, QEMU, opennebula-node-kvm probe | 없음 | 미측정. `RESERVED_MEM` / `RESERVED_CPU`로 예약 [S37] |
| 스토리지 | NFS 클라이언트 | 없음 | 미측정 |
| 스토리지(대안) | Ceph MON / MGR / OSD | MON 5GB 이상, OSD 4GB 이상 [S35] | — |
| 네트워크 | Linux bridge, 8021q 모듈 | 없음 | 미측정 |
| OS | Ubuntu minimal | 1.5GB 설치 최소 [S33] | 미측정 |

예약 설정 [S37]:
- `RESERVED_MEM`(KB 단위)과 `RESERVED_CPU`(CPU share)를 클러스터 또는 호스트 템플릿에 지정한다. 호스트 값이 우선한다.
- 문서 예시 "-10240 adds 1GB"는 단위 표기와 맞지 않는다. 적용 전에 실제 동작을 검증해야 한다.

---

## 7. 네트워크 및 스토리지 선정

### 7.1 스토리지

| 항목 | Local(`TM_MAD=local`) | **NFS(shared/qcow2)** | Ceph RBD |
|---|---|---|---|
| Fault tolerance(공식 표) [S13] | **No** | Yes | Yes |
| host_error `-m` 자동 재배치 | X — local VM은 건너뜀 [S8] | O | O |
| 라이브 마이그레이션 | power-off/live [S13]. 디스크 복사 동반 [코드] | power-off/live | 스토리지 마이그레이션 No |
| Front-end HA 공유 FS 요건 충족 | X | **O** | 이미지 DS만 해당. FE 공유 FS는 별도 필요 [S7][추론] |
| 운영 부담 | 낮음 | 낮음(NAS 운영) | **높음** — 최소 MON 3, MGR 2, OSD 3 [S38] |
| 버전 정합성 | — | NFS v4.2 공식 지원 [S4] | **주의:** OneDeploy 2.0.0의 기본 Ceph release는 `tentacle`이다 [코드: `roles/common/defaults/main.yml`]. 그런데 7.4 Platform Notes가 명시한 지원 버전은 Reef 18.2.x / Squid 19.2.x다 [S4]. 기본값을 그대로 쓰면 안 된다 |
| 메모리 오버헤드(노드) | 공식 수치 없음 | 공식 수치 없음 | OSD당 4GB 이상, MON 5GB 이상 [S35] |
| 장애 영향 | 노드 장애 시 VM 정지·대기 또는 상태 유실 재생성 | NAS 장애 시 전체 영향(SPOF) | 1노드 장애 허용 |
| 공식 권고 | — | 디스크 집약 워크로드에서는 병목 가능 [S14a] | "recommended to not collocate Ceph services … with OpenNebula Nodes or the Front-end" [S14b] |

**선정: NFS v4.2(외부 장비)를 Image + System datastore로 사용**

- Front-end HA의 "A shared filesystem" 요건 [S7]과 VM 자동 재배치(`-m`) [S8]를 함께 만족하는 최소 구성이다.
- Ceph는 HCI를 하면 공식 비권장 [S14b]과 32GiB RAM 한계(4.4)에 걸린다. 별도 3대로 두면 서버 수가 늘어난다.
- 7.4에서 CE로 바뀐 SAN-LVM [S11]은 SAN 장비가 있을 때만 대안이다.

### 7.2 네트워크

| 항목 | Linux Bridge(`bridge`/`fw`) | **802.1Q VLAN** | VXLAN | Open vSwitch |
|---|---|---|---|---|
| 테넌트 L2 격리 | X — 태깅은 VM 책임 [S15] | **O**(VLAN_ID별 브리지) [S16] | O(VNI) | O |
| 추가 구성요소 | 없음 | 8021q 모듈, 스위치 trunk | 멀티캐스트 / IGMP 또는 EVPN(BGP, FRR) | OVS 데몬 |
| 제약 | — | VLAN_IDS 전역 풀, 0/1/4095 예약 | 호스트당 기본 멀티캐스트 그룹 20개, 브리지명 15자 [S17a] | **Security Group 미지원** [S14][S18] |
| Security Group | `fw` 지원 [S15] | 7.4 문서 명시 없음 **[미확인]** | 7.4 문서 명시 없음 **[미확인]** | 미지원 |
| 운영 부담 | 최저 | 낮음 | 중~높음 | 중 |

**선정: Linux Bridge + 802.1Q.** 프로젝트 10개 × 몇 개 VLAN은 4094 범위에서 충분하다. VXLAN(멀티캐스트/EVPN)과 OVS 데몬은 불필요한 구성요소라 제외했다. 참조 아키텍처 Basic 설계도 같은 선택이다 [S34]. 802.1Q에서 Security Group 동작은 PoC로 확인해야 한다.

### 7.3 최소 망 분리

참조 아키텍처 [S34] 기준이다. 7.4 문서 본문에는 망 분리 규정이 없다.

| 망 | 용도 | 최소 요건 | 비고 |
|---|---|---|---|
| 관리/서비스망 | FE↔노드 SSH·모니터링, Raft, VIP, 라이브 마이그레이션 | 10GbE, untagged 또는 전용 VLAN | VIP는 이 망. `MONITOR_ADDRESS`에 VIP 지정 [S7] |
| 스토리지망 | NFS | 10GbE. 관리망과 VLAN 분리(최소)·물리 분리(권장) | NFS 장애는 전체 장애 |
| VM 서비스망 | 프로젝트별 802.1Q VLAN | 스위치 trunk 포트, 전용 NIC | 참조 아키텍처: VM망 전용 NIC [S34] |
| BMC망 | IPMI / Redfish | 전용 포트, 격리 VLAN. **모든 FE 노드에서 도달 가능**해야 함 | fencing 실패 시 VM 복구가 중단됨 [S8] |

---

## 8. HA·Fencing 지원 검증 (CE 7.4 기준)

| 기능 | 판정 | 근거·내용 |
|---|---|---|
| 3노드 Front-end HA | **공식 지원, 별도 구성 필요**(OneDeploy가 `one_vip*`로 자동화) | "three or five servers", 홀수, MySQL 권장, 공유 파일시스템, floating IP [S7]. OneDeploy는 HA에 MariaDB 필수 [코드] |
| Raft / Quorum | **공식 기본 지원** | 기본값 `ELECTION_TIMEOUT_MS=5000`, `BROADCAST_TIMEOUT_MS=500` [코드]. 과반 복제 후 DB 기록 [S7]. 2대 장애 시 쓰기 불가 [추론] |
| 관리 서비스 장애 복구 | **공식 지원, 별도 구성 필요** | 리더 장애 시 재선출. VIP 이동과 OneFlow/OneGate/OneForm 재기동은 `vip.sh`가 처리 [코드]. 장애 팔로워는 DB 재백업 → 복원 → `onezone server-reset` [S7]. VM 로그는 복제되지 않음 |
| Compute 노드 장애 감지 | **공식 기본 지원** | 모니터링 → 호스트 ERROR. `BEACON_HOST`, `MONITORING_INTERVAL_HOST`로 조정 [S8] |
| VM 자동 재시작 | **공식 지원, 별도 구성 필요** | `onehook create`로 `host_error` hook 등록. `-m`(공유 스토리지만), `-r`(상태 유실 재생성) [S8] |
| Live Migration | **공식 기본 지원** | NFS: live 가능 [S13]. 캐시 모드 `none`이 아니면 `MIGRATE_OPTIONS=--unsafe` 필요 [S17] |
| IPMI / Redfish 전원 제어 | **외부 도구 필요** | CE 소스에 IPMI/Redfish 코드 없음 [코드]. `fence_host.sh`에서 fence-agents를 호출한다 [S8][코드]. Ubuntu 24.04 패키지는 `fence-agents-redfish`(`fence_redfish`)와 `fence-agents-ipmilan`(`fence_ipmilan`, `fence_ipmilanplus`), 버전 4.12.1 [S27]. BMC 모델·펌웨어별 동작은 **[미확인]** |
| Fencing / STONITH | **공식 지원, 별도 구성 필요**(스크립트는 사용자 작성) | 7.4.x는 `fence_host.sh.example`만 배포. 스크립트가 없거나 실패하면 VM 복구를 중단 [S8][코드] |
| Split-Brain 방지 | **부분 지원(알려진 위험 있음)** | VM: fencing 성공 후에만 복구 [S8]. 수동 UNKNOWN VM 이동·재스케줄은 fencing을 호출하지 않는다 [S8]. FE: 소수 측은 커밋 불가 [추론: Raft 과반]. **고립된 구 리더는 VIP를 내려놓지 않는다.** `RaftManager::timer_action()`에서 LEADER 상태는 heartbeat 전송만 하고, 과반 상실 시 물러나는 로직이 없다 [코드: 7.4.1 확인] → VIP 중복 가능. A안에서는 고립 노드가 ERROR 판정 후 BMC fencing으로 꺼져 완화된다 [추론]. 장애 시험에서 VIP 중복을 **합격 조건이 아니라 관찰 대상 위험**으로 다룬다 |
| 장애 노드 복귀 | **수동 절차 필요 / 일부 미확인** | `onehost disable` / `offline` / `enable` / `flush` [S19]. ERROR에서 자동 복귀하는지, 재생성된 VM의 구 도메인 처리는 **[미확인]**. `onevm recover` 옵션 [S19a] |
| BMC 원격 전원(운영용 on/off) | **CE 미지원(OpenNebula 기능 없음)** | 외부에서 ipmitool / Redfish로 운영 [코드] |
| Front-end in VM(자기 관리 클러스터 위) | **OneDeploy 지원(libvirt 도메인)** | `arch_infra`: OpenNebula 밖 libvirt VM, autostart [S22]. 순환 의존 경고 문서 없음 **[미확인]** |

**VM 자동 복구 지연 [추론]:** `host_error.rb`는 `-p n` × `MONITORING_INTERVAL_HOST`만큼 대기한 뒤 실행한다 [S8][코드]. 기본값(180s)에 문서 예시 `-p 5`를 쓰면 ERROR 판정 후 약 15분을 더 기다린다. 목표 RTO에 맞춰 조정하되, 너무 짧으면 오탐 위험이 있다 [S8].

---

## 9. 단계별 구축 절차 (A안)

범례:
- **[자동]** OneDeploy 또는 OpenNebula가 수행
- **[수동]** 운영자가 수행
- 명령은 공식 문서·공식 저장소에서 확인한 것만 적었다. 버전 특정 값은 PoC로 확정한다.

| # | 단계 | 구분 | 내용 | 근거 |
|---|---|---|---|---|
| 1 | BIOS/UEFI·BMC | 수동 | 가상화(VT-x/AMD-V, VT-d) 활성화, 3대 CPU 설정 동일화. BMC 전용 포트·고정 IP·전용 계정, BMC망 격리 | CPU 동일성 [S7][S17] |
| 2 | OS 설치 | 수동(autoinstall 가능) | Ubuntu 24.04 Server, 설치 소스 `ubuntu-server-minimal`, OS 디스크 RAID1 | [S32] |
| 3 | 네트워크·SSH·스토리지 | 수동 | netplan(0.105 이상)으로 관리 / 스토리지 / VM trunk 인터페이스 구성. 설치용 sudo 계정과 SSH 키 배포. NFS 서버 export 준비 | [S20] |
| 4 | OneDeploy 실행 환경 | 수동 | 아래 9.1 | [S20][S21] |
| 5 | Inventory | 수동 | 아래 9.2 | [S21][S22] |
| 6 | OpenNebula·KVM 배포 | **자동** | `make I=inventory/prod.yml` | [S21] |
| 7 | 관리 계층 HA | **자동**(OneDeploy `one_vip*`) + 수동 검증 | `onezone show 0`으로 리더 1, 팔로워 2와 index/commit 수렴 확인. 비활성화한 서비스(FireEdge 등)가 실제로 꺼져 있는지 확인 | [S7][S22] |
| 8 | VM 네트워크·Datastore | 자동(`vn`, `ds`) + 수동 | `ds.mode: shared` + `fstab`. 프로젝트 VLAN은 `VN_MAD: 802.1Q` + `VLAN_ID` (OneDeploy 예시 없음 → 수동 또는 Terraform) | [S21][S16] |
| 9 | BMC Fencing | **수동** | 아래 9.3 | [S8] |
| 10 | VDC / Group / Quota | 수동 또는 Terraform | 아래 9.4 | [S23][S24] |
| 11 | Terraform·GitLab CI | 수동 | 아래 9.5 | [S25] |
| 12 | 장애 테스트 | 수동 | 10.3 | — |
| 13 | 백업·업데이트·롤백 | 수동 | 10장 | [S7][S12c] |

### 9.1 OneDeploy 실행 환경 [S20][S21]

대상 클러스터에 의존하지 않는 별도 Ubuntu 제어기(운영 PC/점프 호스트)에서 실행한다.

```bash
git clone --recursive --branch release-2.0.0 https://github.com/OpenNebula/one-deploy.git
cd one-deploy
pipx install hatch
pipx ensurepath
make requirements
```

- Ansible 버전 표기가 문서와 코드에서 다르다. wiki는 `>=2.14,<2.16`(Ceph 프로비저닝 시), `pyproject.toml`은 `ansible-core<2.17`이다. **hatch 환경을 그대로 쓰는 것을 권장한다.**
- Ansible Galaxy의 `opennebula` 컬렉션(0.1.0)은 유지보수 중단 상태이므로 사용하지 않는다 [S21].

### 9.2 Inventory 예 (A안, CE)

공식 예시 `inventory/shared.yml`과 `arch_ha` [S21][S22]의 구조를 따랐다. IP·DNS·인터페이스는 예시 값이다(문서용 192.0.2.0/24). `vault_*` 값은 Ansible Vault 등 암호화 변수로 전달한다.

```yaml
all:
  vars:
    ansible_user: deploy                 # 사전 준비한 SSH/sudo 계정
    ensure_keys_for: [deploy]
    one_version: '7.4.1'                 # 기본값 '7.2' → 반드시 지정. CE 저장소 /repo/7.4.1/ 고정
    one_pass: "{{ vault_one_pass }}"
    # one_token 미지정 → Community Edition 저장소
    db_backend: MariaDB                  # HA는 MariaDB 필수
    db_password: "{{ vault_db_password }}"   # 기본값 'opennebula' → 반드시 변경
    one_vip: 192.0.2.20
    one_vip_cidr: 24
    one_vip_if: bond0.100                # 관리망 인터페이스
    features:
      gui: false
      gate: false
      flow: false
      form: false
      ks: false
      ceph: false
      evpn: false
      prometheus: false
    fstab:
      - src: ha-nfs.example.net:/opennebula
        path: /mnt/one
    ds:
      mode: shared
      config:
        mounts:
          - { type: system, path: /mnt/one/0 }
          - { type: image,  path: /mnt/one/1 }
          - { type: file,   path: /mnt/one/2 }

frontend:
  hosts: &cluster_hosts
    one-a: { ansible_host: 192.0.2.21 }
    one-b: { ansible_host: 192.0.2.22 }
    one-c: { ansible_host: 192.0.2.23 }

node:
  hosts: *cluster_hosts
```

변수 근거 [코드: release-2.0.0]:
- `db_backend` / `db_password`: `roles/database/defaults/main.yml`. 기본값이 `db_password: opennebula`이므로 지정하지 않으면 기본 비밀번호로 배포된다.
- `features`: `roles/common/defaults/main.yml`.
- `fstab`, `ds.config.mounts`: `inventory/shared.yml`.
- `one_version` 고정 경로: `roles/repository/defaults/main.yml`의 `https://downloads.opennebula.io/repo/{{ one_version }}/...`.

주의:
- NFS export 아래 `0`, `1`, `2` 디렉터리와 `oneadmin` UID/GID·쓰기 권한을 미리 맞춰 둔다. 공유 마운트가 빠졌을 때 같은 경로에 로컬 쓰기가 생기지 않는지 확인한다.
- 프로젝트 VNet은 인벤토리가 아니라 Terraform(9.5)으로 관리한다.
- "Conversion from non-HA to HA is not implemented … please plan ahead!" — **처음부터 3대 HA로 배포해야 한다** [S22].
- `one_vip`, `one_vip_cidr`, `one_vip_if`는 모두 지정하거나 모두 빼야 한다 [코드].
- 동일 호스트를 `frontend`와 `node`에 함께 두는 구성은 `infra.yml` 예시에 근거한다 [S22]. bare-metal 통합 배치는 **PoC로 확인**한다.

### 9.3 Fencing 구성 [S8][코드]

1. 모든 FE 노드에 fence-agents를 설치한다. Ubuntu 24.04 패키지는 `fence-agents-redfish` 또는 `fence-agents-ipmilan`이다 [S27]. 각 FE 노드의 Hook 실행 계정(`oneadmin`)으로 BMC 상태 조회가 되는지 먼저 확인한다.
2. `/var/lib/one/remotes/hooks/ft/fence_host.sh.example`을 바탕으로 `fence_host.sh`를 작성한다. 호스트 템플릿의 `FENCE_IP`를 읽어 `fence_redfish` 또는 `fence_ipmilan`으로 `--action=off`를 실행한다. **BMC가 전원 꺼짐을 확인했을 때만 성공(0)으로 종료하고, 그 밖의 경우는 실패로 종료한다.** 스크립트는 모든 FE 노드에 같게 배포한다. 어느 노드가 Leader가 되어도 동작해야 한다.
3. 각 호스트에 `onehost update <id>`로 `FENCE_IP`를 추가한다. BMC 자격증명은 스크립트·템플릿에 평문으로 두지 않고 권한 600 파일 등으로 분리한다 [추론].
4. `onehook create /usr/share/one/examples/host_hooks/error_hook`로 hook을 등록한다(예시 파일 경로 [미확인]). 인자는 `$TEMPLATE -m -p <n>`이다.
5. 업그레이드 후 fence 스크립트가 보존되는지 확인한다. 7.4.0에서는 덮어쓰기 결함이 있었다 [S12b].

> ⚠ **파괴적 작업:** fence 스크립트는 실제 서버 전원을 끈다.
> - 실행 전 확인: `FENCE_IP`와 호스트가 1:1로 맞는지, 테스트 시 대상 노드에 운영 VM이 없는지, 자격증명이 맞는지.
> - fencing을 끄는 `--no-fencing`은 split-brain 위험이 있어 사용하지 않는다 [S8].

### 9.4 프로젝트(VDC / Group / Quota) [S23][S24]

```bash
onegroup create --name projA --admin_user projA-admin --admin_password <pw> --resources TEMPLATE+VM
onevdc create projA-vdc
onevdc addgroup projA-vdc projA
onevdc addcluster projA-vdc 0 <cluster_id>
onevdc addvnet   projA-vdc 0 <vnet_id>
onegroup quota projA
```

- 새 그룹은 `default` VDC에 자동으로 소속된다. **프로젝트 VDC에 넣기 전에 `default` VDC에서 제거해야 한다** [S23].
- VDC ACL은 자동 생성되며 수동으로 편집하지 않는다 [S23].
- 프로젝트별 VLAN은 VNet 단위로 VDC에 할당한다.

### 9.5 Terraform·GitLab CI/CD [S25][S26]

| 항목 | 내용 |
|---|---|
| Provider | `OpenNebula/opennebula` v1.5.0 (2025-06-26). README 기준 "~> 7.0". **7.4 호환성 [미확인]** |
| 리소스 | `opennebula_group`, `opennebula_group_quotas`, `opennebula_group_admins`, `opennebula_virtual_data_center`, `opennebula_virtual_network`, `opennebula_security_group`, `opennebula_acl`, `opennebula_template`, `opennebula_virtual_machine` 등 |
| 인증 | `OPENNEBULA_ENDPOINT=https://<VIP>:2633/RPC2`(XML-RPC), `OPENNEBULA_USERNAME`, `OPENNEBULA_PASSWORD` |
| CI 자격증명 | 프로젝트별 사용자에 `oneuser token-create --time <초> --group <gid>` 토큰 발급 → GitLab CI masked/protected 변수에 저장. 토큰을 password로 쓰는 방식은 OpenNebula 세션 형식 [S24a] 기준이며, provider 문서 명시는 **[미확인]** |
| 구조 | 공통 인프라(oneadmin 권한) 저장소와 프로젝트별(그룹 관리자 토큰) 저장소 분리 [추론]. state 백엔드는 GitLab-managed Terraform state(HTTP backend, 잠금 지원) [S28] |
| 파이프라인 | MR에서 `validate`·`plan` → 보호 브랜치·환경 승인 후 `apply`. Provider 버전 고정과 `.terraform.lock.hcl` 커밋. 내부 Runner만 VIP API에 접근. state·plan artifact는 비밀로 취급(접근 제한) |
| 관리 주체 | 기존 객체는 `import` 검토 후 관리 주체를 하나로 제한한다(CLI 수동 변경과 Terraform 혼용 금지). GitLab·Runner·state 백업은 OpenNebula 전체 정지 중에도 접근 가능한 위치에 둔다 |

> ⚠ Terraform의 리소스 삭제·교체(`-/+`)는 VM과 데이터에 영향을 준다. `apply` 전에 plan의 삭제·교체 대상과 VM 디스크 영속성, 백업 시점을 확인한다.

---

## 10. 운영·업데이트·장애 복구 절차

### 10.1 백업

| 대상 | 방법 | 근거 |
|---|---|---|
| OpenNebula DB | 리더에서 `onedb backup -u oneadmin -p <pw> -d opennebula` | [S7] |
| 설정 | `/etc/one`, `/var/lib/one/remotes`, `/var/lib/one/.one`(FireEdge 키 포함) | [S7] |
| VM 로그 | 리더에서 별도 수집(HA로 복제되지 않음) | [S7] |
| VM 디스크 | NFS 장비 스냅샷/복제, 또는 OpenNebula 백업 datastore(restic 0.17.3 [S4]; CE 사용 범위 **[미확인]**) | — |
| IaC | GitLab 저장소, Terraform state | — |

### 10.2 업데이트

**OpenNebula 업그레이드 (HA, 공식 절차 [S12d] 요약)**

| 단계 | 내용 |
|---|---|
| 0 | 다음 **공개 CE** 패키지의 릴리스 노트·Known Issues·Compatibility Guide를 확인한다. CE는 critical 패치만 제공한다 [S10]. 스테이징에서 먼저 시험한다 |
| 1 | 전이 상태(prolog, migrate, epilog, save)의 VM이 없는지 확인 |
| 2 | `onehost disable <host_id>`, `onezone disable <zone_id>` |
| 3 | **Follower부터 중지, Leader는 마지막.** `systemctl list-units \| grep opennebula`의 출력이 비어야 한다 |
| 4 | Leader: 설정·DB 백업 → 패키지 업데이트 → `onedb` 마이그레이션, `onecfg upgrade` [S12c] → `onedb backup`으로 Follower용 덤프 생성 |
| 5 | Follower: 같은 업데이트를 적용하고 서비스는 중지 상태를 유지 |
| 6 | Follower에 Leader 덤프를 `onedb restore -f <backup_file>`로 복원. `/etc/one`과 `/var/lib/one/remotes/etc`를 동기화하고 소유자를 `oneadmin`으로 맞춘다 |
| 7 | Leader, Follower 순으로 서비스 기동 → `onezone show`로 상태 확인 → `onezone enable <zone_id>` |
| 8 | 하이퍼바이저 패키지 업데이트 → `onehost enable`. **fence 스크립트가 보존됐는지 확인**하고 VIP·fencing을 재시험한다 [S12b] |
| 참고 | 7.4.x 유지보수 업데이트(예: 7.4.1→7.4.2)는 DB 스키마 업그레이드가 필요 없다고 공식 페이지에 적혀 있다 [S12d]. OneDeploy `upgrade` 타깃(2.0.0 신규)은 PoC 검증 후에만 사용한다 [S21] |

> ⚠ 6단계 `onedb restore -f`는 Follower DB를 덮어쓴다. 대상이 Follower이고 덤프가 방금 만든 Leader 백업인지 확인한 뒤 실행한다.

**OS 업데이트:** 노드별로 순차 진행한다. `onehost flush <id>`(VM 이동 후 disable) → 패치·재부팅 → `onehost enable <id>`. 한 번에 1대만 진행하고, 다음 노드로 넘어가기 전에 Raft 과반, NFS 접근, Host 모니터 상태가 회복됐는지 확인한다 [S19][추론].

> ⚠ `onehost flush`의 기본 동작은 `/etc/one/cli/onehost.yaml`에 따라 **delete-recreate**일 수 있다(VM 상태 유실) [S19]. 실행 전에 설정을 확인하고, NFS 환경에서는 resched로 맞춘다.

**롤백 (공식 "Restoring Previous Version" [S12e])**

1. 새 버전이 설치된 상태에서 업그레이드 전 DB 백업을 `onedb restore -f`로 복원한다.
2. 새 버전을 제거하고 이전 버전 패키지를 재설치한다.
3. 업그레이드 전에 백업한 `/etc/one`을 복원한다.

> ⚠ 데이터 덮어쓰기와 변경 유실 위험이 있다. 관리 DB만 되돌리면 그 이후 바뀐 VM 디스크나 Terraform state와 어긋날 수 있다. 복원 범위는 스테이징 리허설에서 정한다.

### 10.3 장애 대응·복구 테스트

시험은 승인된 시험 창과 스테이징 데이터에서만 수행한다. **RTO**는 장애 감지부터 관리 API 쓰기 회복, 애플리케이션 회복까지 각각 기록한다. **RPO**는 쓰기 검증 데이터와 실제 복구 시점을 비교해 기록한다.

| 시나리오 | 기대 동작 | 확인 |
|---|---|---|
| `oned` 리더 프로세스만 중지 | Follower 1대가 Leader로 전환, VIP가 한 곳에만 존재, API 쓰기 회복. 기존 VM I/O는 정상 | `onezone show 0`, API 쓰기 |
| 팔로워 노드 전원 차단 | 쿼럼 유지. 해당 노드 VM은 fencing 성공 후 `-m`으로 **정확히 한 번** 재시작 | VM 위치, 중복 기동 여부 |
| 리더 노드 전원 차단 | 재선출(`ELECTION_TIMEOUT_MS` 기준), VIP 이동, 이후 host_error 처리 | VIP 응답, hook 로그 |
| Compute 관리망만 단절(BMC망 정상) | fencing 성공 전에는 새 인스턴스가 시작되지 않는다. 고립 노드가 fencing된다 | VM 중복 여부 |
| 리더 쪽 네트워크 분할 | 소수 측은 쓰기 불가. **구 리더의 VIP 중복을 관찰**한다(코드상 step-down 없음, 8장) | ARP 테이블, VIP 응답 노드 |
| Front-end 2대 중단 | 과반 상실로 쓰기 불가. 두 번째 노드가 돌아온 뒤에만 제어 기능 재개 | API 쓰기 |
| BMC 불통 | fencing 실패 → VM 복구 중단(의도된 동작) + 운영 경보 | hook 로그, 경보 |
| NFS 컨트롤러·링크 절체 | 기존 VM I/O 복구와 데이터 정합성. 무관한 Compute의 연쇄 fencing이 없어야 한다 | VM I/O 지연, Host 상태 |
| 정상 Live Migration | 동종 CPU, 같은 VNet·스토리지에서 양방향 성공. NVRAM VM도 포함 | 서비스 연결, 쓰기 |
| 계획 점검과 장애 복귀 | 복귀 후 구 VM 중복 부팅 없음. Follower 동기화, NFS 쓰기, VLAN 도달성 정상 | `onezone show`, `onevm list` |
| 전체 전원 장애 | NFS·스위치 → 호스트 → MariaDB/`oned` 순으로 기동. 과반·VIP·Host 상태 회복 | 5.1 부팅 순서 |
| 장애 팔로워 복구 | 리더에서 새 DB 백업 → 팔로워 `onedb restore -f …` → 리더 `onezone server-reset <zone> <id>` | [S7] |
| 노드 FE 교체 | `onezone server-del` 후 재추가(한 번에 1대) | [S7] |
| 백업 복원·권한 | 별도 경로로 관리 DB·설정과 대표 VM을 복원. 프로젝트 간 접근 거부, fencing 대상 매핑, GitLab state 정합성 확인 | 복원 리허설 |

> ⚠ **파괴적 명령**
> - `onedb restore -f`는 대상 서버의 DB를 덮어쓴다. 실행 전 대상 호스트명과 리더가 아님을 확인한다.
> - `onezone server-add`는 추가되는 서버의 DB를 폐기한다 [S7].
> - `onezone server-del`은 쿼럼 구성을 바꾼다. 남은 서버가 과반인지 확인한다.
> - `onevm recover --delete` / `--recreate`는 VM 상태를 잃는다 [S19a]. 원래 인스턴스가 fencing으로 멈췄는지 확인하기 전에는 실행하지 않는다(split-brain, 데이터 손상 위험 [S8]).

---

## 11. 미확정 사항 및 검증 필요 항목

| # | 항목 | 영향 | 검증 방법 |
|---|---|---|---|
| 1 | **CE 7.4.1의 보안·결함 패치 공백**(7.4.2 EE에만 CVE·XSS·인증·NUMA 크래시·SG 규칙 손실 수정) | 높음 | FireEdge 미배포, 관리망 격리. 후보 CPU로 NUMA 보고 시험. 차기 CE 패치 공지 모니터링. EE 구독 검토 |
| 1a | CVE-2026-84165의 7.4.x 영향 여부(INCIBE 공지 내용 자체가 모순) | 높음 | 공급사 확인. 그 전까지 VM exec 권한과 qemu-agent 사용 제한 |
| 2 | FE 실사용 RAM/CPU(공식 16GiB 대비) | 높음 — 32GiB 적합성 | PoC 실측 |
| 3 | OneDeploy 2.0.0 + `one_version: '7.4'` + FE·Node 동일 호스트 bare-metal 배포 | 높음 | PoC |
| 4 | Terraform Provider v1.5.0의 7.4 호환성 | 중 | PoC(리소스별 apply/destroy) |
| 5 | 802.1Q에서 Security Group 동작 | 중 | PoC |
| 6 | 후보 BMC 모델·펌웨어에서 `fence_redfish` / `fence_ipmilan`의 OFF 동작과 확인(패키지 존재는 확인됨 [S27]) | 중 | 실장비 시험 |
| 7 | `error_hook` 예시 파일 경로 | 낮음 | 설치 후 확인 |
| 8 | 장애 호스트가 ERROR에서 자동 복귀하는지, 재생성 VM의 구 도메인 처리 | 중 | 장애 테스트 |
| 9 | Raft 분할 시 구 리더 VIP 중복(코드상 step-down 없음 — 확인됨) | 중 | 망 분할 시험으로 실제 영향과 fencing에 의한 완화를 확인 |
| 10 | Ceph 대안 채택 시 OneDeploy 기본값(Tentacle)과 지원 버전(Reef/Squid)의 차이 | 낮음(선정안 아님) | `ceph.release` override 시험 |
| 11 | CE 버전별 EOL 날짜 | 중 | OpenNebula 공지 |
| 12 | 공식 롤백 절차의 실제 소요 시간과 VM 디스크·state 정합성 | 중 | 스테이징 리허설 |
| 13 | NFS 장비 이중화 방식(제품 의존) | 높음 — SPOF | 장비 선정 |
| 14 | `RESERVED_MEM` 단위 표기 불일치(KB vs "1GB" 예시) | 낮음 | PoC |
| 15 | Ubuntu minimized 설치 후 실측 RAM | 낮음 | PoC |

---

## 12. 공식 출처 URL

**버전·에디션·플랫폼**

| ID | URL | 적용 |
|---|---|---|
| S1 | https://github.com/OpenNebula/one/tags | 태그 |
| S2 | https://downloads.opennebula.io/repo/ | CE 저장소 |
| S3 | https://docs.opennebula.io/7.4/software/release_information/release_notes/resolved_issues_742/ | 7.4.2 EE |
| S3a | https://docs.opennebula.io/7.4/software/release_information/release_notes/resolved_issues_741/ | 7.4.1 EE+CE |
| S3b | https://www.incibe.es/en/incibe-cert/notices/aviso/lack-authorisation-opennebula-opennebula-systems | CVE-2026-84165 (INCIBE-2026-596) |
| S4 | https://docs.opennebula.io/7.4/software/release_information/release_notes/platform_notes/ | 7.4 |
| S5 | https://docs.opennebula.io/7.4/software/installation_process/frontend_installation/overview/ | 7.4 |
| S6 | https://opennebula.io/platform/ | 현재 |
| S6a | https://forum.opennebula.io/t/opennebula-7-4-2-maintenance-release-is-here/14652 | 7.4.2 공지 |
| S9 | https://docs.opennebula.io/7.4/software/release_information/release_notes/subscriptions/ | 7.4 |
| S10 | https://github.com/OpenNebula/one/wiki/Release-Policy | 현재 |
| S11 | https://docs.opennebula.io/7.4/software/release_information/release_notes/whats_new/ | 7.4 |
| S12 | https://docs.opennebula.io/7.4/software/release_information/release_notes/ | 7.4 |
| S12a | https://docs.opennebula.io/7.4/software/release_information/release_notes/compatibility/ | 7.4 |
| S12b | https://docs.opennebula.io/7.4/software/release_information/release_notes/known_issues/ | 7.4 |
| S12c | https://docs.opennebula.io/7.4/software/upgrade_process/upgrade_guide/overview/ , https://docs.opennebula.io/7.4/software/upgrade_process/upgrade_guide/start_here/ | 7.4 |
| S12d | https://docs.opennebula.io/7.4/software/upgrade_process/upgrade_guide/upgrading_ha/ | 7.4 |
| S12e | https://docs.opennebula.io/7.4/software/upgrade_process/upgrade_guide/restoring_version/ | 7.4 |

**HA·장애 복구·호스트 운영**

| ID | URL | 적용 |
|---|---|---|
| S7 | https://docs.opennebula.io/7.4/product/control_plane_configuration/high_availability/frontend_ha/ | 7.4 CE |
| S8 | https://docs.opennebula.io/7.4/product/control_plane_configuration/high_availability/vm_ha/ | 7.4 CE |
| S17 | https://docs.opennebula.io/7.4/product/operation_references/hypervisor_configuration/kvm_driver/ | 7.4 |
| S19 | https://docs.opennebula.io/7.4/product/cluster_configuration/hosts_and_clusters/hosts/ | 7.4 |
| S19a | https://docs.opennebula.io/7.4/product/operation_references/opennebula_services_configuration/troubleshooting/ | 7.4 |
| S19b | https://docs.opennebula.io/7.4/product/operation_references/configuration_references/host_states/ | 7.4 |
| 코드 | https://github.com/OpenNebula/one/tree/release-7.4.1 (`share/hooks/ft/host_error.rb`, `share/hooks/raft/vip.sh`, `share/etc/oned.conf`, `src/vmm_mad/remotes/kvm/migrate`, `src/rm/RaftManager.cc`) | 7.4.1 |

**스토리지·네트워크**

| ID | URL | 적용 |
|---|---|---|
| S13 | https://docs.opennebula.io/7.4/product/cluster_configuration/storage_system/overview/ | 7.4 |
| S13a | https://docs.opennebula.io/7.4/product/cluster_configuration/storage_system/local_ds/ | 7.4 |
| S14a | https://docs.opennebula.io/7.4/product/cluster_configuration/storage_system/nas_ds/ | 7.4 |
| S14b | https://docs.opennebula.io/7.4/product/cluster_configuration/storage_system/ceph_ds/ | 7.4 |
| S14 | https://docs.opennebula.io/7.4/product/cluster_configuration/networking_system/overview/ | 7.4 |
| S15 | https://docs.opennebula.io/7.4/product/cluster_configuration/networking_system/bridged/ | 7.4 |
| S16 | https://docs.opennebula.io/7.4/product/cluster_configuration/networking_system/vlan/ | 7.4 |
| S17a | https://docs.opennebula.io/7.4/product/cluster_configuration/networking_system/vxlan/ | 7.4 |
| S18 | https://docs.opennebula.io/7.4/product/virtual_machines_operation/virtual_machines_networking/security_groups/ | 7.4 |

**용량·예약·참조 아키텍처**

| ID | URL | 적용 |
|---|---|---|
| S34 | https://docs.opennebula.io/7.4/getting_started/understand_opennebula/cloud_architecture_and_design/open_cloud_reference_architecture/ (PDF r2.4: https://support.opennebula.pro/hc/en-us/article_attachments/19958759982493) | 2024-06, 6.x 시기 |
| S36 | https://docs.opennebula.io/7.4/product/control_plane_configuration/large-scale_deployment/scalability/ | 7.4 |
| S37 | https://docs.opennebula.io/7.4/product/cloud_system_administration/capacity_planning/overcommitment/ | 7.4 |

**설치 자동화·멀티테넌시·IaC**

| ID | URL | 적용 |
|---|---|---|
| S20 | https://github.com/OpenNebula/one-deploy/wiki/sys_reqs | OneDeploy |
| S20a | https://docs.opennebula.io/7.4/getting_started/install_opennebula/one_deploy/one_deploy_overview/ | 7.4 |
| S21 | https://github.com/OpenNebula/one-deploy/wiki/sys_use , https://github.com/OpenNebula/one-deploy/releases/tag/release-2.0.0 | 2.0.0 |
| S21b | https://github.com/OpenNebula/one-deploy/tree/release-2.0.0 (`roles/common/defaults/main.yml`, `roles/database/defaults/main.yml`, `roles/repository/defaults/main.yml`, `roles/precheck/pre_reboot/tasks/integrity.yml`, `inventory/shared.yml`) | 2.0.0 코드 |
| S22 | https://github.com/OpenNebula/one-deploy/wiki/arch_ha , https://github.com/OpenNebula/one-deploy/wiki/arch_infra | 2.0.0 |
| S23 | https://docs.opennebula.io/7.4/product/cloud_system_administration/multitenancy/manage_vdcs/ , https://docs.opennebula.io/7.4/product/cloud_system_administration/multitenancy/manage_groups/ | 7.4 |
| S24 | https://docs.opennebula.io/7.4/product/cloud_system_administration/capacity_planning/quotas/ | 7.4 |
| S24a | https://docs.opennebula.io/7.4/product/cloud_system_administration/multitenancy/manage_users/#tokens | 7.4 |
| S25 | https://registry.terraform.io/providers/OpenNebula/opennebula/latest/docs , https://github.com/OpenNebula/terraform-provider-opennebula | v1.5.0 |
| S26 | https://docs.opennebula.io/7.4/product/cloud_system_administration/multitenancy/chmod/ | 7.4 |
| S27 | https://manpages.ubuntu.com/manpages/noble/man8/fence_redfish.8.html , https://manpages.ubuntu.com/manpages/noble/man8/fence_ipmilanplus.8.html | Ubuntu 24.04, fence-agents 4.12.1 |
| S28 | https://docs.gitlab.com/user/infrastructure/iac/terraform_state/ | GitLab |

**OS·Ceph**

| ID | URL | 적용 |
|---|---|---|
| S30 | https://ubuntu.com/about/release-cycle , https://docs.rockylinux.org/10/releases/ , https://wiki.almalinux.org/release-notes/ , https://www.debian.org/releases/trixie/ , https://wiki.debian.org/LTS | 2026-11 |
| S31 | packages.ubuntu.com / packages.debian.org / repo.almalinux.org (2026-11-24 조회) | 패키지 버전 |
| S32 | https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html , https://documentation.ubuntu.com/public-cloud/all-clouds-explanation/ubuntu-base-and-minimal-images/ | 24.04 |
| S33 | https://ubuntu.com/server/docs/reference/installation/system-requirements/ | 24.04 |
| S35 | https://docs.ceph.com/en/umbrella/start/hardware-recommendations/ | Ceph Umbrella / Tentacle |
| S38 | https://docs.ceph.com/en/tentacle/start/ | Ceph Tentacle |
