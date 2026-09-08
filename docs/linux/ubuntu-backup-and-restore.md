# Ubuntu 백업 및 복원

## 문서 목적

- Ubuntu 재설치 또는 디스크·파티션 작업 전에 필요한 데이터를 보존
- 별도 물리 디스크에 사용자 데이터와 시스템 복구 정보를 백업
- 백업 결과를 검증하고 필요할 때 안전하게 복원

> [!IMPORTANT]
>
> - 원본 Ubuntu 설치 디스크와 다른 물리 디스크에 백업 필수
> - 같은 디스크의 다른 파티션 사용 시 디스크 고장·파티션 작업 실수로 동시 손실 가능

## 권장 백업 범위

| 구분 | 대상 | 용도 | 권장 여부 |
|---|---|---|---|
| 사용자 데이터 | `/home` | 문서, 사진, 프로젝트, 사용자 설정 | 필수 |
| 시스템 설정 | `/etc`, `/root` | 서비스 및 관리자 설정 참고 | 권장 |
| 설치 정보 | APT, Snap, Flatpak 목록 | 재설치 후 환경 재구성 | 권장 |
| 오프라인 root 사본 | Ubuntu root 파일시스템 전체 | 파티션 작업 전 파일 단위 복구 | 파티션 작업 시 권장 |
| EFI 부트 파일 | EFI 시스템 파티션 | 부트로더 복구 참고 | 파티션 작업 시 권장 |

- 데이터베이스, 가상 머신, 컨테이너처럼 실행 중에 계속 변경되는 데이터는 해당 프로그램의 내보내기 또는 스냅샷 기능도 함께 사용
- SSH 키, GPG 키, 브라우저 프로필, 비밀번호 관리자 복구 키 등 다시 만들기 어려운 자료가 `/home`에 포함되는지 확인
- 백업 디스크에 민감한 자료를 저장한다면 LUKS 등 디스크 암호화 사용 권장

## 백업 디스크 준비

### 1. 필요한 용량 확인

```bash
df -hT /
sudo du -xhd1 /home /etc /root 2>/dev/null
```

- 최소한 백업 대상의 사용량보다 여유가 큰 외장 디스크 준비
- root 전체를 복사할 때는 `df -h /`의 사용 공간 이상 필요
- 여러 세대의 백업을 보관하려면 변경량과 보관 횟수만큼 추가 공간 확보

### 2. 원본과 대상 디스크 구분

- 외장 디스크 연결 후 장치 및 마운트 정보 기록

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,TRAN
findmnt /
```

- `/`가 있는 원본 디스크와 외장 백업 디스크의 장치명·모델·용량이 다른지 확인
- 장치명은 연결 순서에 따라 달라질 수 있으므로 이름만 보고 판단하지 않음
- 이 문서의 예시는 파일 앱으로 마운트된 `UBUNTU_BACKUP` 라벨의 외장 디스크 사용

### 3. 파일시스템 확인

```bash
findmnt -T "/media/$USER/UBUNTU_BACKUP"
df -hT "/media/$USER/UBUNTU_BACKUP"
```

- 직접 파일을 복사하는 전체 시스템 백업은 Linux 권한, ACL, 확장 속성, 심볼릭 링크를 보존할 수 있는 ext4, XFS 또는 Btrfs 권장
- exFAT와 FAT 계열은 Linux 파일 메타데이터를 온전히 보존하지 못하므로 전체 시스템 백업 대상으로 사용하지 않음
- 기존 디스크를 포맷하면 그 안의 데이터가 모두 삭제되므로 포맷이 필요할 때는 대상 디스크를 다시 확인

## 실행 중인 Ubuntu에서 기본 백업

- 백업 대상: 사용자 데이터와 재설치에 필요한 정보
- 데이터베이스·가상 머신 등 일관된 스냅샷이 필요한 프로그램은 백업 전 안전하게 종료

### 1. 백업 경로 설정

- 예시 외장 디스크 라벨: `UBUNTU_BACKUP`
- 실제 마운트 경로에 맞게 `BACKUP_MOUNT` 값 수정

```bash
BACKUP_MOUNT="/media/$USER/UBUNTU_BACKUP"
BACKUP_DIR="$BACKUP_MOUNT/ubuntu-$(hostname)-$(date +%F)"

findmnt -T "$BACKUP_MOUNT"
df -h "$BACKUP_MOUNT"
sudo mkdir -p "$BACKUP_DIR/system-info"
sudo chown "$USER:$(id -gn)" "$BACKUP_DIR" "$BACKUP_DIR/system-info"
```

> [!CAUTION]
>
> - `findmnt` 결과가 외장 백업 디스크를 가리키지 않으면 작업 중단
> - 외장 디스크 미마운트 상태에서 경로 생성 시 원본 디스크에 백업 저장 가능

### 2. 시스템 재구성 정보 저장

```bash
lsblk -f > "$BACKUP_DIR/system-info/lsblk.txt"
findmnt > "$BACKUP_DIR/system-info/findmnt.txt"
sudo fdisk -l > "$BACKUP_DIR/system-info/fdisk.txt"
dpkg-query -W -f='${binary:Package}\t${Version}\n' \
  > "$BACKUP_DIR/system-info/dpkg-packages.tsv"
apt-mark showmanual | sort \
  > "$BACKUP_DIR/system-info/apt-manual.txt"
snap list > "$BACKUP_DIR/system-info/snap-list.txt" 2>/dev/null || true

if command -v flatpak >/dev/null 2>&1; then
  flatpak list --app --columns=application,origin \
    > "$BACKUP_DIR/system-info/flatpak-apps.tsv"
fi
```

- 패키지 목록은 프로그램 데이터 자체가 아니라 재설치할 항목을 찾기 위한 기록
- 별도로 설치한 AppImage, 수동 설치 프로그램, 개발 도구 버전도 필요한 경우 기록

### 3. 사용자 데이터와 설정 복사

```bash
sudo rsync -aAXHx --numeric-ids --info=progress2 \
  --exclude='/*/.cache/' \
  --exclude='/*/.local/share/Trash/' \
  /home/ "$BACKUP_DIR/home/"

sudo rsync -aAXH --numeric-ids --info=progress2 \
  /etc/ "$BACKUP_DIR/etc/"

sudo rsync -aAXH --numeric-ids --info=progress2 \
  /root/ "$BACKUP_DIR/root/"
```

- 옵션별 의미

| 옵션 | 의미 |
|---|---|
| `-a` | 디렉터리를 재귀 복사하고 기본 메타데이터 보존 |
| `-A` | ACL 보존 |
| `-X` | 확장 속성 보존 |
| `-H` | 하드 링크 보존 |
| `-x` | 다른 파일시스템의 마운트 지점으로 넘어가지 않음 |
| `--numeric-ids` | 사용자·그룹 이름 대신 숫자 ID 보존 |
| `--info=progress2` | 전체 진행률 표시 |

- 캐시와 휴지통을 포함해야 한다면 두 `--exclude` 줄 제거
- 별도의 데이터 파티션이나 `/home` 파티션이 있다면 해당 마운트 지점도 빠짐없이 백업
- 명령이 오류로 끝나면 마지막 화면만 보지 말고 오류가 발생한 파일과 원인을 확인한 뒤 다시 실행

## Live USB에서 오프라인 root 백업

- 파티션 이동·크기 변경 전 Live 환경에서 root 전체 복사 권장
- 목적: 실행 중인 프로그램의 파일 변경 없이 백업
- 예시 장치명 기준: [Windows 제거 후 Ubuntu root 파티션 확장](ubuntu-remove-windows-expand-root.md)의 시스템 구성

### 1. Live 환경에서 장치 확인

- Ubuntu 설치 USB로 UEFI 부팅
- 설치 화면에서 **Try Ubuntu** 선택
- 원본 root와 외장 백업 디스크 확인

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL,TRAN
```

- 예시 장치 및 경로

| 역할 | 장치 또는 경로 |
|---|---|
| 원본 Ubuntu root | `/dev/nvme0n1p5` |
| 원본 EFI 파티션 | `/dev/nvme0n1p1` |
| 외장 백업 디스크 | 파일 앱에서 연 `UBUNTU_BACKUP` |
| 외장 디스크 마운트 경로 | `/media/ubuntu/UBUNTU_BACKUP` |

> [!WARNING]
>
> - 실제 장치명·파티션 번호와 다른 예시 명령 실행 금지
> - `lsblk`의 파일시스템·크기·모델을 현재 구성과 대조

### 2. 원본을 읽기 전용으로 마운트

```bash
SOURCE_ROOT="/dev/nvme0n1p5"
SOURCE_EFI="/dev/nvme0n1p1"
BACKUP_MOUNT="/media/ubuntu/UBUNTU_BACKUP"
OFFLINE_DIR="$BACKUP_MOUNT/offline-ubuntu-$(date +%F)"

findmnt -T "$BACKUP_MOUNT"
df -h "$BACKUP_MOUNT"
sudo mkdir -p /mnt/ubuntu-root /mnt/ubuntu-efi "$OFFLINE_DIR"
sudo mount -o ro "$SOURCE_ROOT" /mnt/ubuntu-root
sudo mount -o ro "$SOURCE_EFI" /mnt/ubuntu-efi
findmnt /mnt/ubuntu-root
findmnt /mnt/ubuntu-efi
```

- `findmnt` 출력의 원본 두 파티션에 `ro` 옵션이 있는지 확인
- 암호화된 LUKS 볼륨은 먼저 잠금을 해제하고 내부의 논리 볼륨을 `SOURCE_ROOT`로 지정
- root와 분리된 `/home` 또는 `/boot` 파티션은 별도로 읽기 전용 마운트하여 추가 백업

### 3. root와 EFI 복사

```bash
sudo rsync -aAXH --numeric-ids --info=progress2 \
  /mnt/ubuntu-root/ "$OFFLINE_DIR/root/"

sudo rsync -aAXH --numeric-ids --info=progress2 \
  /mnt/ubuntu-efi/ "$OFFLINE_DIR/efi/"
```

- Live 환경에서 root 파티션만 직접 마운트했으므로 `/proc`, `/sys`, `/run` 같은 가상 파일시스템은 복사되지 않음
- 이 백업은 파일 단위 사본이며 파티션 테이블이나 미사용 공간까지 담는 전체 디스크 이미지는 아님
- 전체 디스크를 원래 배치 그대로 복구해야 한다면 Clonezilla 같은 이미지 백업을 추가로 사용

## 백업 검증

- 복사 완료 메시지 외에 체크섬 비교와 파일 열기·복원 시험으로 검증

### 1. `rsync` 재검사

- 기본 백업: 복사 시 사용한 제외 규칙을 동일하게 적용하여 검사

```bash
sudo rsync -aAXHxnci --numeric-ids \
  --exclude='/*/.cache/' \
  --exclude='/*/.local/share/Trash/' \
  /home/ "$BACKUP_DIR/home/"
```

- 오프라인 root 백업: Live 환경에서 원본과 백업 사본 비교

```bash
sudo rsync -aAXHnci --numeric-ids \
  /mnt/ubuntu-root/ "$OFFLINE_DIR/root/"
```

- `-n`은 실제 변경 없이 비교하는 dry run
- `-c`는 파일 크기와 시간만이 아니라 체크섬으로 내용 비교
- 아무 파일도 출력되지 않고 종료 코드가 `0`이면 원본에서 백업 대상으로 추가 복사할 차이가 없다는 의미
- 실행 중인 시스템은 검사 중 파일이 바뀔 수 있으므로 계속 변하는 캐시나 애플리케이션 데이터는 종료 후 다시 확인

### 2. 직접 열기와 복원 시험

- 문서, 사진, 압축 파일, 프로젝트 파일을 백업 디스크에서 몇 개 직접 열기
- 중요한 디렉터리 하나를 임시 위치에 복사하고 내용 확인
- `system-info`의 장치 및 패키지 목록이 비어 있지 않은지 확인
- 백업 디스크의 파일시스템 오류가 보고되지 않았는지 확인

### 3. 안전하게 분리

```bash
sync
mountpoint -q /mnt/ubuntu-efi && sudo umount /mnt/ubuntu-efi
mountpoint -q /mnt/ubuntu-root && sudo umount /mnt/ubuntu-root
sudo umount "$BACKUP_MOUNT"
```

- `target is busy` 오류가 나면 백업 디스크를 사용하는 터미널이나 파일 앱 창을 닫고 다시 시도
- 마운트가 해제된 것을 확인한 뒤 외장 디스크 분리

## 데이터 복원

### 일부 파일 복원

- 백업 디스크에서 필요한 파일을 먼저 별도 임시 디렉터리로 복사
- 내용과 날짜를 확인한 뒤 현재 파일과 교체
- 동일 경로에 바로 덮어쓰면 더 최신인 파일을 잃을 수 있으므로 주의

### `/home` 복원

- 복원할 Ubuntu의 사용자 계정과 사용자 ID 확인
- 예시 실행 환경: 설치된 Ubuntu에서 복원 대상 사용자 로그아웃 후 다른 관리자 계정 사용
- Live USB 사용 시 설치된 Ubuntu 파티션을 마운트하고 복원 목적지를 해당 파티션의 `home/`으로 변경
- 백업의 숫자 UID/GID와 복원할 계정의 UID/GID가 일치하는지 확인하고, 다르면 계정 매핑을 해결한 뒤 복원

```bash
id
getent passwd
```

- dry run으로 변경 대상 사전 확인

```bash
sudo rsync -aAXHnvi --numeric-ids \
  "/media/$USER/UBUNTU_BACKUP/백업-디렉터리/home/" /home/
```

- 출력 내용 확인 후 `-n`을 제거하여 실제 복원 실행

```bash
sudo rsync -aAXHvi --numeric-ids \
  "/media/$USER/UBUNTU_BACKUP/백업-디렉터리/home/" /home/
```

> [!CAUTION]
>
> - 복원 명령 실행 시 기존 파일 덮어쓰기 가능
> - `백업-디렉터리`를 실제 이름으로 변경하고 dry run 결과 확인 후 실행
> - 원본에 없는 대상 파일을 삭제하는 `--delete` 옵션 미사용

### 패키지 환경 복구

```bash
sudo apt update
xargs -r sudo apt install -y \
  < "/media/$USER/UBUNTU_BACKUP/백업-디렉터리/system-info/apt-manual.txt"
```

- Ubuntu 버전이 바뀌면 일부 패키지 이름이나 저장소가 달라질 수 있음
- 전체 목록을 한 번에 적용하기보다 필요한 프로그램을 확인하면서 설치 권장
- Snap과 Flatpak 목록은 기록을 참고하여 필요한 앱만 다시 설치

### 전체 시스템 복구

- 오프라인 root 사본은 자동 복구 이미지가 아니므로 새 파일시스템 준비, 파일 복사, `/etc/fstab`의 UUID 확인, GRUB 재설치가 별도로 필요
- 원래 디스크 구조 전체를 되돌리려면 Clonezilla 등으로 만든 디스크 이미지 사용
- 복구 대상 디스크를 잘못 선택하면 기존 데이터가 삭제될 수 있으므로 전체 복구 전 장치명·모델·용량을 재확인

## 최종 확인표

- [ ] 원본과 다른 물리 디스크에 백업
- [ ] 사용자 데이터와 숨김 설정 파일 포함 확인
- [ ] 별도 데이터 파티션과 애플리케이션별 내보내기 확인
- [ ] APT, Snap, Flatpak 및 디스크 구성 정보 저장
- [ ] 파티션 작업 전 오프라인 root와 EFI 백업
- [ ] `rsync` dry run 체크섬 검사 완료
- [ ] 중요 파일 직접 열기 또는 일부 복원 시험 완료
- [ ] `sync` 후 백업 디스크 안전하게 분리

## 참고 자료

- [rsync 공식 매뉴얼](https://download.samba.org/pub/rsync/rsync.1): 메타데이터 보존, 제외 규칙, dry run 및 체크섬 옵션
