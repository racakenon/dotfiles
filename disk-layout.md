# 디스크 / 부트 레이아웃

Windows·Linux 멀티부트 구성과 그 설계 근거.

목표는 **OS별 설치·재설치 독립성**이다. 한쪽을 밀어도 다른 쪽이 영향을 받지 않아야 한다.

> 이 레이아웃은 **검증용 임시 구성**이다. 설계가 실제로 성립하는지 확인하기 위한 것이고,
> 확정된 배치가 아니다. 재검토 항목은 문서 끝에 있다.

## 전제

- UEFI + GPT
- 내장 NVMe 2개 — 하나는 Windows 전용, 하나는 데이터 + Linux 루트
- 외장 SSD 1개 — Linux의 유일한 부트 매체

장치 이름은 역할로 부른다. NVMe 열거 순서(`nvme0n1`/`nvme1n1`)는 부팅마다 뒤바뀔 수 있어
식별자로 쓸 수 없다. 마운트는 전부 UUID/PARTUUID로 건다.

---

## 레이아웃

```
내장 A   Windows 전용
  p1  200M    ESP     Windows 부트로더 전용
  p2   16M    MSR
  p3  464.7G  NTFS    C:
  p4  872M    NTFS    WinRE

내장 B   데이터 + Linux 루트          ★ EFI System 타입 파티션 없음
  p1  500M    exFAT   Windows·Linux 공용 (SHARED)
  p2  465.3G  btrfs   배포판별 서브볼륨 (@ @home @snapshots @var_log)

외장     Linux 유일한 부트 매체
  p1  2G      FAT32   ESP — rEFInd + 배포판별 커널·initramfs
  p2  230.9G  btrfs   비상 배포판 (@rescue @rescue-home @rescue-log)
```

### 설계 근거

**내장 B에 EFI System 타입 파티션을 두지 않는 것이 핵심.**
Windows Setup과 `bcdboot`는 부트로더를 심을 때 EFI System 타입 파티션을 찾는다. 대상이 없으면
침입 자체가 불가능하고, Setup은 설치 대상 디스크에 새로 만든다. 공용 파티션은 Microsoft basic
data 타입이라 대상이 되지 않는다.

**Linux 부트를 외장으로 빼는 이유.**
Windows 재설치 시 외장을 분리하면 Linux 부트로더가 물리적으로 존재하지 않는 상태가 된다.
대신 외장이 SPOF가 되므로 ESP 백업과 예비 부팅 매체가 필요하다.

**커널이 외장 ESP에 올라간다.**
rEFInd가 EFI stub 커널을 직접 로드하므로 `/boot`가 곧 외장 ESP다. 외장이 연결되지 않은
상태에서는 커널 업데이트가 실패한다. ESP를 2G로 잡은 것은 배포판이 늘어나도 커널·initramfs
세트를 여러 벌 올릴 수 있게 하기 위해서다.

**비상 배포판을 외장에 두는 이유.**
내장 B의 btrfs는 모든 배포판이 공유한다. 그 파일시스템을 수리해야 할 때 올라탈 수 있는,
어디에도 의존하지 않는 시스템이 하나 필요하다.

---

## ESP 공유 구조

하나의 ESP를 모든 배포판이 공유한다. 배포판마다 ESP 루트 아래 자기 디렉터리를 갖는다.

```
ESP/
  EFI/refind/            rEFInd 본체 + refind.conf + themes/
  EFI/tools/
  amd-ucode.img          내용이 동일하므로 루트에서 공유
  arch/
    vmlinuz-linux
    initramfs-linux.img
    refind_linux.conf    root=PARTUUID=<내장B p2>, rootflags=subvol=@
  rescue/
    vmlinuz-linux
    initramfs-linux.img
    refind_linux.conf    root=PARTUUID=<외장 p2>, rootflags=subvol=@rescue
```

- 커널 파일명이 배포판끼리 겹치므로 **디렉터리로 분리**한다. 각 배포판의
  `mkinitcpio` preset이 자기 디렉터리를 가리키게 한다.
- `refind_linux.conf`는 커널과 같은 디렉터리에 둔다. rEFInd가 자동으로 짝지어 읽는다.
  `initrd=` 경로는 ESP 루트 기준이다.
- rEFInd 기본 스캔 대상은 ESP 루트와 `EFI/*`뿐이다. 하위 디렉터리는 `refind.conf`에
  `also_scan_dirs +,rescue,arch`로 명시해야 메뉴에 뜬다.
- rEFInd 설치본을 `EFI/refind`와 `EFI/BOOT` 두 벌 두지 않는다. 각자 자기 디렉터리의
  `refind.conf`를 읽기 때문에 설정이 갈린다.

---

## Linux 공간 분배 정책

내장 B의 btrfs 파일시스템 **하나**를 배포판들이 공유하고, 배포판마다 서브볼륨을 갖는다.
서브볼륨은 크기 개념이 없어 풀의 남은 공간을 자유롭게 쓴다. 사전 분할 없음, 추가·삭제 즉시 반영.

### 서브볼륨 명명

```
@<distro>          → /
@<distro>-home     → /home
@<distro>-snap     → /.snapshots
@<distro>-log      → /var/log
@data              → 배포판 간 공유 데이터
```

- 서브볼륨은 **전부 최상위에 평평하게** 둔다. 중첩하면 부모 스냅샷에 딸려 들어간다.
- `@home`을 배포판끼리 공유하지 않는다. 파일시스템 문제가 아니라 설정 버전 스큐 문제다.
- `df`는 모든 서브볼륨에 동일한 총량을 보고한다. 실제 배분은 `btrfs filesystem usage /`로 확인.
- `btrfs check --repair`는 전 배포판이 걸린 파일시스템을 건드리므로 비상 배포판에서만 실행한다.
- 서브볼륨을 마운트할 때는 최상위(`subvolid=5`)를 먼저 풀어라. 파일시스템 단위 옵션
  (`compress` 등)은 첫 마운트가 결정하며 이후 마운트의 값은 무시된다.

---

## Secure Boot

자체 키로 서명한다. shim은 쓰지 않는다.

```
PK / KEK / db   자체 키 + Microsoft 키 + 펌웨어 기본 db
서명 대상       rEFInd, 배포판별 커널 (stub 커널을 rEFInd가 직접 로드하므로 커널도 대상)
```

- 서명이 필요한 것은 **부트로더와 커널**이다. initramfs는 대상이 아니다.
- 자체 키를 등록하려면 펌웨어를 Setup Mode로 넣어야 한다. BIOS에서 Secure Boot Mode를
  Custom으로 바꾸고 PK를 지우는 것 외에 방법이 없다. OS에서는 PK를 지울 수 없다.
- 공장 키 재주입 옵션(`Factory Default Key Provisioning`)이 켜져 있으면 지워도 다음 POST에
  되돌아온다. 이 옵션을 먼저 끄고 저장·재부팅한 뒤에 키를 지운다.
- `refind` 패키지에는 ESP 갱신 pacman hook이 없다. `refind-install`을 돌릴 때마다 ESP의
  바이너리가 패키지 원본(미서명)으로 덮어써지므로 **매번 재서명해야 한다.**
- GRUB은 `SecureBoot=1`을 감지하면 스스로 락다운에 들어가고, shim이 제공하는 프로토콜이
  없으면 모듈을 하나도 로드하지 못한다. 이 구조에서 GRUB을 부트로더로 쓸 수 없는 이유다.

---

## UEFI 부팅 순서

외장 매체를 부트 디바이스로 쓸 때 두 경로가 있다.

```
NVRAM Boot#### 항목        →  EFI/refind/refind_x64.efi
이동식 매체 fallback 경로   →  EFI/BOOT/BOOTX64.EFI  (NVRAM 항목 불필요)
```

fallback 경로는 UEFI 규격상 **이동식 매체 전용 규칙**이다. USB 플래시 드라이브는
`removable=1`로 보고되어 펌웨어가 이 경로를 알아서 찾지만, **외장 SSD는 `removable=0`,
즉 고정 디스크로 보고된다.** 따라서 외장 SSD에서는 fallback 자동 발견이 일어나지 않고
NVRAM 항목이 반드시 있어야 한다.

그런데 이 펌웨어는 OS에서 만든 NVRAM 항목(`efibootmgr` 등)을 다음 POST에서 유지하지 않는다.
펌웨어 자신이 BootOrder를 쓴 경우만 남는다.

**결론 — 부트로더 설치 후 BIOS에 직접 들어가 부팅 순서를 확정해야 한다.** 이 하드웨어에서
외장 SSD 부팅을 유지하는 유일한 방법이다.

---

## 검증된 것

이 구성으로 다음이 성립함을 확인했다.

- **자체 키 서명으로 Secure Boot 활성 상태 부팅.** rEFInd와 커널을 자체 db 키로 서명하고
  `SecureBoot=1`에서 정상 부팅. Windows 부팅도 유지(Microsoft 키 병행 등록).
- **하나의 ESP에 여러 Linux 배포판 공존.** 커널 디렉터리 분리 + 배포판별 `refind_linux.conf`로
  충돌 없이 부팅. 각 배포판이 자기 커널만 갱신한다.
- **내장 ESP 없이 외장만으로 부팅.** 내장 디스크에서 EFI System 타입 파티션을 제거한 뒤에도
  Linux가 정상 부팅.
- **Windows·Linux 공용 exFAT 파티션 읽기·쓰기.** 양쪽 OS에서 동일 파티션에 읽고 쓰기 확인.
- **외장 SSD 부팅 순서 유지 방법.** 위 절의 내용.

## 재검토 대상

지금 구성은 여기까지 확인하기 위한 것이고, 다음을 정리한 뒤 레이아웃을 다시 정한다.

- **부트 매체를 외장 SSD에서 내장 SATA SSD로 바꿀 가능성.** 물리 분리라는 이점은
  줄지만 부팅 순서 문제와 지연이 사라진다. Windows 설치 시 SATA를 분리하는 방식으로
  독립성을 유지할 수 있는지 검토.
- **성능과 부팅 지연.** 외장 USB 경로에서 부팅 시점에 지연이 발생한다.
- **공용 파티션 크기.** 현재 500M은 접근 확인용이다. 실사용 크기는 내장 B의 btrfs를
  축소해 확보한다.
- **설치 스크립트 반영.** `install-phase1`은 단일 디스크에 ESP + btrfs를 만드는 구성이라
  이 레이아웃을 구현하지 않는다.
