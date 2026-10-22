# WarehousePG WEM 1.4 Ansible

**이미 설치되어 동작 중인 WarehousePG(WHPG) 클러스터**에 WEM(WarehousePG Enterprise Manager) 1.4 를 설치하고 삭제하는 Ansible 입니다.
WEM 1.2 가 있으면 지우고 1.4 로 전환합니다. WHPG 자체의 설치와 HA 구성은 다루지 않습니다.

> 기준 환경: Rocky Linux 9, WHPG 7.6, WEM 1.4.0, Host Agent 1.1.0, otelcol 0.158.0, ansible-core 2.14 이상
> 전제: WHPG 가 설치되어 동작 중이고, 모든 노드에 WEM 1.4 패키지를 받을 수 있는 yum/dnf repo 가 **이미 구성**되어 있어야 합니다(2-3절).

## 0. 한눈에 보기

| 하고 싶은 일 | 명령 (이 디렉터리에서 실행) |
|---|---|
| inventory.yml 생성 (WHPG 에서 읽기) | `ansible-playbook playbooks/generate_inventory.yml` |
| **WEM 1.4 설치** (1.2 가 있으면 삭제 후) | `ansible-playbook playbooks/wem_deploy.yml` |
| 위와 같고 standby 의 DB 메트릭 수집이 VIP 로 접속 | `ansible-playbook playbooks/wem_deploy.yml -e wem_use_vip=true` |
| 관련 rpm 의 설치 / repo 최신 버전 확인만 | `ansible-playbook playbooks/wem_deploy.yml -t upgrade --check` |
| **관련 rpm 을 repo 최신 버전으로 업그레이드** | `ansible-playbook playbooks/wem_deploy.yml -t upgrade` |
| **WEM 1.4 삭제** | `ansible-playbook playbooks/wem_uninstall.yml` |

`wem_deploy.yml` 과 `wem_uninstall.yml` 은 **확인 질문 없이 바로 삭제를 시작**합니다. 실행하기 전에 [10. 운영 환경 적용 전 점검](#10-운영-환경-적용-전-점검)을 확인합니다.

## 0-1. 수행 순서

모든 명령은 이 디렉터리(`ansible.cfg` 가 있는 곳)에서 실행합니다. 각 단계의 자세한 내용은 괄호 안의 절을 참고합니다.

### 처음 설치 (deploy)

**1단계. 사전 준비 확인** (2-2절, 2-3절)

- 관리 서버에서 모든 노드로 비밀번호 없이 SSH 접속이 되어야 합니다.
- 모든 노드에서 WEM 관련 패키지를 받을 수 있는 yum/dnf repo 가 구성되어 있어야 합니다.

**2단계. `inventory/cluster_vars.yml` 에 대상 클러스터 정보 수정** (2-4절)

```yaml
whpg_coordinator_hosts:                     # inventory 를 만들 때 접속할 서버 (순서대로 시도). standby 는 적지 않음
    - "10.0.15.200"                                 # VIP (IP 만)
    - { ip: "10.0.15.112", hostname: "whpg-cdw" }   # coordinator (실 IP 와 실제 hostname)
ansible_ssh_common_args: "-o IdentityFile=~/.ssh/id_rsa"   # 노드 접속 키 (여러 개면 IdentityFile 을 이어서 적음)
whpg_coordinator_data_directory: "/data/coordinator/gpseg-1"
whpg_default_database: "gpadmin"
whpg_vip_cidr: "10.0.15.200/20"             # VIP 가 없으면 비움
```

`whpg_coordinator_hosts` 에는 **VIP 와 coordinator 만** 적고 standby coordinator 는 적지 않습니다.
coordinator 의 `hostname` 은 실제 서버의 hostname 과 같아야 합니다. 다르면 다른 클러스터에 잘못 접속한 것으로 보고 중단합니다.
VIP 는 붙어 있는 서버가 바뀌므로 hostname 없이 IP 만 적습니다. VIP 가 없으면 coordinator 만 적습니다.
매번 같은 값으로 쓸 WEM 변수(`wem_port`, 비밀번호 등)도 여기에 함께 적을 수 있습니다(4절).

**3단계. `inventory.yml` 생성과 접속 확인** (2-4절)

```bash
ansible-playbook playbooks/generate_inventory.yml
ansible all -m ping
```

만들어진 `inventory.yml` 에서 노드 목록과 `whpg_role`(coordinator / standby / segment)이 맞는지 확인합니다.
WEM 은 standby 가 있으면 standby, 없으면 coordinator 에 설치됩니다.

**4단계. WEM 설치**

```bash
ansible-playbook playbooks/wem_deploy.yml                       # 기본
ansible-playbook playbooks/wem_deploy.yml -e wem_use_vip=true   # standby 의 DB 메트릭 수집을 VIP 로 접속 (HA 환경)
ansible-playbook playbooks/wem_deploy.yml -e @my_vars.yml       # 비밀번호 등을 파일로 지정 (4절)
```

WEM 1.2 가 있으면 먼저 지우고 1.4 를 설치합니다(3-1절).

**5단계. 설치 확인** (8절)

- 마지막 `Summary` 와 `Show service states` 출력에서 서비스가 모두 `active` 인지 확인합니다.
- WEM 화면(`http://<WEM host IP>:8080/`)의 **Management > Host Agents** 에서 모든 노드가 approved 인지 확인합니다.
  자동 승인이 안 된 노드가 있으면 화면에서 승인한 뒤 `ansible-playbook playbooks/wem_deploy.yml -t tune,querylogs` 를 실행합니다.

### 업그레이드 (upgrade)

설치되어 있는 WEM, ClickHouse, Host Agent, otelcol rpm 을 repo 의 최신 버전으로 올립니다(3-2절).

```bash
ansible-playbook playbooks/wem_deploy.yml -t upgrade --check   # 1) 설치 버전과 repo 최신 버전, 업그레이드 대상 확인 (변경 없음)
ansible-playbook playbooks/wem_deploy.yml -t upgrade           # 2) 업그레이드, 올라간 서비스만 재시작
```

1) 의 마지막 `Summary` 에 모든 노드의 업그레이드 대상(`호스트: 패키지 현재 버전 -> 최신 버전`)이 모여 나옵니다. 확인한 뒤 2) 를 실행합니다.
마이너 버전이 바뀌는 경우(예: WEM 1.4.0 → 1.5.0)에는 릴리스 노트를 먼저 확인합니다.
업그레이드 후에는 5단계와 같이 서비스 상태와 Host Agents 화면을 확인합니다.

### 삭제 (uninstall)

WEM 1.4 와 ClickHouse, Host Agent, otelcol, WHPG 의 WEM 계정 / DB 를 지웁니다(3-3절). WHPG 본체는 지우지 않습니다.

```bash
ansible-playbook playbooks/wem_uninstall.yml
```

마지막 출력에서 rpm, 서비스, 포트, WHPG 의 DB / 계정이 남지 않았는지 확인합니다.

### 설정을 바꿔 다시 설치할 때

```bash
ansible-playbook playbooks/wem_uninstall.yml          # 1) 기존 WEM 삭제 (Host Agent 인증 정보도 정리)
vi inventory/cluster_vars.yml                         # 2) 필요하면 클러스터 정보 / 변수 수정
ansible-playbook playbooks/generate_inventory.yml     # 3) cluster_vars.yml 을 고쳤으면 inventory.yml 다시 생성
ansible-playbook playbooks/wem_deploy.yml             # 4) 다시 설치
```

otelcol 조정 값처럼 일부 단계만 다시 적용하면 되는 경우에는 삭제하지 않고 해당 tag 만 실행합니다(7절, 예: `-t tune`).

## 1. 디렉터리 구성

```
samsung_ansible/
├── ansible.cfg                  # inventory / roles 경로 등 (이 디렉터리에서 실행해야 읽힘)
├── inventory.yml                # 호스트와 클러스터 변수 (playbooks/generate_inventory.yml 로 생성)
├── inventory/
│   ├── cluster_vars.yml         # 클러스터 공통 변수 (직접 수정)
│   └── inventory.yml.j2         # inventory.yml 템플릿
├── playbooks/
│   ├── generate_inventory.yml   # WHPG 의 gp_segment_configuration + /etc/hosts + cluster_vars.yml -> inventory.yml
│   ├── wem_deploy.yml           # WEM 1.2 삭제 + 1.4 설치
│   └── wem_uninstall.yml        # WEM 1.4 삭제
└── roles/
    └── warehouse-pg-wem/        # WEM 설치 / 삭제 role (11절)
```

두 playbook 은 role 하나(`warehouse-pg-wem`)만 호출하는 짧은 파일이고, 실제 작업은 `roles/warehouse-pg-wem/tasks/` 의 단계별 파일, 변수는 `roles/warehouse-pg-wem/defaults/main.yml` 에 있습니다.

## 2. 사전 구성 사항

### 2-1. 실행 서버

- ansible 은 **관리 서버 한 곳**에서만 실행합니다.
- 반드시 이 디렉터리(`ansible.cfg` 가 있는 곳)에서 `ansible-playbook playbooks/<파일>` 형식으로 실행합니다.
  `playbooks/` 안으로 들어가서 실행하면 `ansible.cfg` 를 읽지 못해 inventory 와 roles 를 찾지 못합니다.

### 2-2. 대상 서버에 root SSH 키 배포

비밀번호 입력 없이 root 로 접속할 수 있어야 합니다. 노드 수만큼 반복하고 `ping` 으로 확인합니다.

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
for ip in <노드 IP들>; do ssh-copy-id root@$ip; done
ansible -i inventory.yml all -m ping
```

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
for ip in 10.0.15.112 10.0.8.209 10.0.3.219 10.0.14.73 ; do ssh-copy-id root@$ip; done
ansible -i inventory.yml all -m ping
```

### 2-3. yum/dnf repository

이 Ansible 은 repo 를 구성하지 않습니다. 실행 전에 모든 노드에서 아래 패키지를 받을 수 있어야 합니다(EDB repo 또는 사내 로컬 repo).

| 노드 | 패키지 |
|---|---|
| WEM host (standby, 없으면 coordinator) | `whpg-enterprise-manager-1.4.0`, `edb-clickhouse-server`, `edb-clickhouse-client` (`edb-clickhouse-common-static` 의존) |
| 전 노드 | `edb-acp-host-agent`, `edb-otelcol` |
| (`wem_state_install=true` 일 때) WEM host | `postgresql-server`, `postgresql-contrib`, `policycoreutils-python-utils` |

```bash
ansible all -b -m shell -a 'dnf -q info edb-acp-host-agent edb-otelcol >/dev/null && echo OK'
```

### 2-4. inventory.yml

`inventory.yml` 은 **동작 중인 WarehousePG 에 접속해** 자동으로 만듭니다. 클러스터 공통 변수는 `inventory/cluster_vars.yml` 에 적습니다.
`cluster_vars.yml` 을 고쳤거나 노드가 바뀌면 다시 실행합니다(기존 `inventory.yml` 은 같은 디렉터리에 날짜를 붙여 백업).

```bash
ansible-playbook playbooks/generate_inventory.yml                                   # inventory.yml 생성
ansible-playbook playbooks/generate_inventory.yml -e whpg_coordinator=10.0.15.112   # 접속할 coordinator 를 이번만 바꿔 지정
ansible-playbook playbooks/generate_inventory.yml -e inventory_output=/tmp/inv.yml  # 미리 만들어 보기
```

동작:

1. `inventory/cluster_vars.yml` 의 `whpg_coordinator_hosts` 에 적힌 순서로 SSH 접속합니다. **VIP 와 coordinator 만** 적고 standby coordinator 는 적지 않습니다.
   VIP 로 먼저 접속하고, VIP 가 없거나 응답하지 않으면 coordinator 로 접속합니다.

```yaml
whpg_coordinator_hosts:
   - "10.0.15.200"                                 # VIP: IP 만 (붙어 있는 서버가 바뀌므로 hostname 을 적지 않음)
   - { ip: "10.0.15.112", hostname: "whpg-cdw" }   # coordinator: 실 IP 와 실제 hostname
```

   `-e whpg_coordinator=<IP 또는 이름>[,<IP>...]` 를 주면 그 값이 우선합니다. 기존 `inventory.yml` 은 보지 않으므로 `inventory.yml` 이 없어도 됩니다.
   SSH 접속 변수(`ansible_user`, `ansible_port`, `ansible_ssh_common_args` 등)도 `cluster_vars.yml` 에서 읽습니다. 적지 않으면 ansible 을 실행한 계정, `~/.ssh` 기본 키, `sudo` 로 root 입니다.
2. gpadmin 으로 `gp_segment_configuration` 을 조회하고(recovery 상태인 standby 에서는 실패하므로 건너뜀) 그 서버의 `/etc/hosts` 를 읽습니다.

   | gp_segment_configuration | whpg_role |
   |---|---|
   | `content = -1`, `role = p` | `coordinator` |
   | `content = -1`, `role = m` | `standby` |
   | `content >= 0` (primary / mirror) | `segment` |

3. `address` / `hostname` 을 `/etc/hosts` 에서 찾아 IP(`ansible_host`, `ip_address`)를 정합니다. 찾지 못하는 호스트가 있으면 중단합니다.
4. 각 IP 에 SSH 로 접속해 **실제 hostname** 을 읽어 inventory 호스트 이름과 `hostname` 으로 씁니다. 접속이 안 되면 `/etc/hosts` 의 첫 번째 이름을 쓰고 경고를 출력합니다.
   `hostname_alias` 는 `/etc/hosts` 에서 그 IP 에 붙은 다른 이름(예: `10.0.15.112 whpg-cdw cdw` 이면 `cdw`)입니다.
5. `whpg_clusters` 그룹과 함께 `coordinator`, `standby_coordinator`, `segments` 그룹도 만듭니다.

`inventory/cluster_vars.yml` 의 변수:

| 변수 | 필수 | 용도 |
|---|---|---|
| `whpg_coordinator_hosts` | 예 | `generate_inventory.yml` 이 접속할 서버 목록 (순서대로 시도). **VIP 와 coordinator 만** 적고 standby coordinator 는 적지 않음. 항목은 `"IP"` 또는 `{ ip: "IP", hostname: "이름" }`. SSH 접속은 항상 `ip` 로 하고, `hostname` 을 적으면 실제 hostname 과 비교해 다르면 중단 (다른 클러스터 오접속 방지). VIP 는 붙어 있는 서버가 바뀌므로 IP 만 적음 |
| `ansible_user`, `ansible_port`, `ansible_ssh_common_args` | 아니오 | SSH 접속 설정. `generate_inventory.yml` 의 접속과 생성된 `inventory.yml`(WEM 설치 / 삭제)에 함께 적용. 서버마다 키가 다르면 `ansible_ssh_common_args: "-o IdentityFile=~/.ssh/id_rsa -o IdentityFile=~/.ssh/<다른 키>"` 처럼 여러 개 적음 (ssh 가 순서대로 시도). `ansible_ssh_private_key_file` 은 키 하나만 쓰고 기본 키도 시도하지 않으므로 키가 섞인 환경에서는 쓰지 않음 |
| `whpg_coordinator_data_directory` | 예 | coordinator 데이터 디렉터리 (예: `/data/coordinator/gpseg-1`). `pg_hba.conf` 위치와 `gpstop -u` 에 사용 |
| `whpg_default_database` | 아니오 | 클러스터 기본 DB. WEM 삭제 시 지우지 않는 보호 목록에 포함 |
| `whpg_vip_cidr` | 아니오 | coordinator VIP (예: `192.168.10.88/24`). `wem_use_vip=true` 일 때 사용. **VIP 가 coordinator 에 실제로 올라와 있지 않으면 사용하지 않음** |

WEM 은 **standby 가 있으면 standby, 없으면 coordinator** 에 설치됩니다(`whpg_role` 기준).

## 3. 실행

### 3-1. WEM 1.4 설치 (WEM 1.2 가 있으면 삭제 후)

```bash
ansible-playbook playbooks/wem_deploy.yml
```

수행 순서는 다음과 같습니다. 각 단계의 이름이 [7. 단계별 실행](#7-단계별-실행-tags)의 tag 입니다.

1. 실행 전 확인(OS, VIP 활성 여부, 배치 계산)
2. WEM 1.2 삭제 (Prometheus, Loki, Alertmanager, Alloy, collector, WEM, WHPG 의 `wem` DB). 1.2 가 없으면 지울 것이 없어 그냥 지나감
3. WHPG `pg_hba.conf` 에 `wem_whpg_user` 접속 허용 추가 (`gpstop -u`)
4. ClickHouse 설치 (WEM host)
5. WEM 접속 계정 / DB / resource group 생성 (WHPG coordinator). `wem_state_install=true` 면 WEM host 에 별도 상태 DB(PostgreSQL) 설치
6. WEM 1.4 설치와 `wem.conf` 구성 (WEM host)
7. Host Agent 와 otelcol 설치, otelcol 조정 파일(`config_local.yaml`) 선적용, `join-token` 선등록으로 **자동 승인** (전 노드)
8. WEM 이 배포한 otelcol 설정과 병합 검사, 필요할 때만 otelcol 재시작
9. ClickHouse Coordinator Query Logs 객체 생성
10. 서비스 상태 확인

### 3-2. 관련 rpm 업그레이드

```bash
ansible-playbook playbooks/wem_deploy.yml -t upgrade --check   # 버전 확인만
ansible-playbook playbooks/wem_deploy.yml -t upgrade           # 확인 후 업그레이드
```

`-t upgrade` 를 지정할 때만 실행됩니다(`never` tag). 일반 설치 실행(tag 없음)에는 포함되지 않습니다.

1. 각 노드에서 repo 메타데이터를 새로 받고, 패키지별 설치 버전과 repo 최신 버전, 업그레이드 대상 여부를 출력
2. WEM host: ClickHouse(`edb-clickhouse-server`, `edb-clickhouse-client`) → WEM(`whpg-enterprise-manager`) 순으로 업그레이드하고, 올라간 서비스만 재시작
3. 모든 노드: `edb-acp-host-agent`, `edb-otelcol` 업그레이드 후, 올라간 노드만 한 대씩 `acp-host-agent`, `edb-otelcol` 재시작
4. 업그레이드 후 버전 출력
5. 마지막 `Summary` 에 모든 노드의 업그레이드 대상(`--check`) 또는 업그레이드한 패키지를 모아서 출력. 대상이 없으면 `없음`

- 이미 설치된 패키지만 올립니다(새로 설치하지 않음). 대상 목록은 `wem_upgrade_packages_wem_host`, `wem_upgrade_packages_all_hosts` 로 바꿉니다.
- `wem.conf`, `config.yaml`, `edb-otelcol.conf` 는 rpm 의 `%config(noreplace)` 이고 `config_local.yaml` 은 rpm 소유가 아니므로 설정은 유지됩니다.
- **repo 의 최신 버전으로 올라가므로 마이너 버전이 바뀔 수 있습니다** (예: WEM 1.4.0 → 1.5.0). 먼저 `--check` 로 대상 버전을 확인합니다.
- 업그레이드한 뒤에도 일반 설치 실행은 WEM 을 `wem14_package` 의 고정 버전으로 내리지 않습니다(설치된 버전이 같거나 높으면 설치 건너뜀).

### 3-3. WEM 1.4 삭제

```bash
ansible-playbook playbooks/wem_uninstall.yml
```

WHPG 본체, WEM 1.2(Prometheus/Loki/Alloy), `/etc/edb/pgaa` 는 지우지 않습니다.

- 서비스 / 파일: WEM, ClickHouse, 상태 DB(설치한 경우), Host Agent, otelcol 과 설정 / 데이터 디렉터리, `pg_hba.conf` 의 WEM 블록
- rpm: `whpg-enterprise-manager`(버전과 관계없이. `wem_remove_version_prefix: "1.4"` 로 두면 1.4 일 때만), `edb-acp-host-agent`, `edb-otelcol`, `edb-clickhouse-*`, 함께 설치된 뒤 쓰이지 않는 의존 패키지(`wem_remove_autoremove`)
- WHPG 데이터베이스: `wem_remove_databases`(기본 `wem`, WEM 상태 DB)는 소유자와 관계없이 접속 세션을 끊고 지우고, `wem_whpg_database` 는 `wem_whpg_user` 소유일 때만 지웁니다.
  `postgres`, `template0`, `template1`, `gpadmin` 과 클러스터 기본 DB(`wem_protected_databases`)는 **어떤 경우에도 지우지 않습니다.**
- WHPG 계정 / resource group: 각 DB 에서 `DROP OWNED BY wem_whpg_user` 후 `wem_whpg_user`, `wem_whpg_resource_group`, `wem` / `wemadmin` role 삭제
- 마지막에 rpm, 서비스, 포트, WHPG 의 DB / 계정 / resource group 이 남았는지 출력합니다.

이전 1.4 의 Host Agent 인증 정보가 남아 있으면 재설치할 때 등록이 꼬일 수 있어서, 1.4 를 다시 설치하기 전에는 이 playbook 을 먼저 실행합니다.

## 4. 변수 지정 방법

변수는 아래 세 가지 방법으로 바꿀 수 있습니다. 위에 있을수록 우선순위가 높습니다.

**방법 1. 실행할 때 `-e` 로 지정 (한 번만 바꿀 때)**

```bash
ansible-playbook playbooks/wem_deploy.yml -e wem_port=8090 -e wem_use_vip=true
```

**방법 2. 파일로 지정 (여러 값을 바꿀 때)**

값을 적은 파일을 만들고 `-e @파일` 로 넘깁니다. 비밀번호가 들어가면 파일 권한을 `600` 으로 하고 git 에 올리지 않습니다.

```yaml
# my_vars.yml
wem_admin_password: "새 비밀번호"
clickhouse_password: "새 비밀번호"
wem_whpg_password: "새 비밀번호"
wem_state_password: "새 비밀번호"
```

```bash
ansible-playbook playbooks/wem_deploy.yml -e @my_vars.yml
```

**방법 3. `inventory/cluster_vars.yml` 에 고정 (매번 같은 값을 쓸 때)**

`inventory/cluster_vars.yml` 에 변수를 추가하고 `ansible-playbook playbooks/generate_inventory.yml` 을 다시 실행해 `inventory.yml` 에 반영합니다.

어느 방법으로도 지정하지 않은 변수는 `roles/warehouse-pg-wem/defaults/main.yml` 의 기본값을 씁니다.

주의할 점은 다음과 같습니다.

- 비밀번호는 **설치할 때** 적용됩니다. WEM `admin` 의 비밀번호(`wem_admin_password`)는 최초 설정에만 쓰이므로, 설치 후에 값만 바꿔서 다시 실행해도 로그인 비밀번호는 바뀌지 않습니다.
- 값을 바꿔서 다시 설치할 때는 `wem_uninstall.yml` 로 지운 뒤 설치합니다.
- `true` / `false` 는 `-e wem_use_vip=true` 처럼 따옴표 없이 적습니다.

## 5. 자주 바꾸는 변수

전체 목록은 6절에 있습니다.

| 변수 | 기본값 | 바꾸는 경우 |
|---|---|---|
| `wem_admin_password` | `changeme_wem` | WEM 화면 `admin` 비밀번호. 운영에서는 반드시 변경 |
| `clickhouse_password` | `changeme` | ClickHouse `default` 사용자 비밀번호. 운영에서는 반드시 변경 |
| `wem_whpg_password` / `wem_state_password` | `changeme` | WEM 이 WHPG 에 접속하는 `wemon` 비밀번호. 운영에서는 반드시 변경 |
| `wem_whpg_database` | `gpadmin` | WEM 이 모니터링 접속에 쓰는 WHPG DB (Host Agent 의 `WEM_DATABASE`) |
| `wem_sqlquery_on_standby` | `true` | DB 메트릭(`sqlquery`)을 standby 에서 수집 (가이드 주의 2). coordinator 에서 수집하려면 `false` |
| `wem_use_vip` | `false` | standby 의 DB 메트릭 수집이 VIP 로 접속하려면 `true` (HA 환경) |
| `wem_drop_old_state_db_in_whpg` | `true` | 1.2 의 상태 DB(WHPG `wem` DB)를 보존하려면 `false` |
| `wem_port` | `8080` | WEM 웹 포트 변경. Host Agent 포트는 자동으로 +1 |
| `wem_whpg_resource_group_options` | CONCURRENCY 20, CPU_MAX_PERCENT 10 | `wem_group` resource group 설정 |
| `wem_extra_conf_lines` | `[]` | `wem.conf` 에 줄을 추가 (예: `LOG_SCAN_ENABLED=false`) |
| `clickhouse_data_dir` | 비움 | ClickHouse 데이터를 다른 디스크에 둘 때 |
| `coordinator_query_logs_ttl_days` | `90` | Query Logs 보관 일수 |

## 6. 변수 레퍼런스

모든 변수는 `roles/warehouse-pg-wem/defaults/main.yml` 한 파일에 있고, 파일 안에서 단계별 구역으로 나뉘어 있습니다. 괄호 안은 그 변수를 쓰는 태스크 파일입니다.

### 6-1. 공통 / WHPG 접속 / 상태 DB

| 변수 | 기본값 | 설명 |
|---|---|---|
| `DNF_TIMEOUT` | `900` | dnf 작업 최대 대기 시간(초) |
| `wem_sqlquery_on_standby` | `true` | standby 가 있으면 DB 메트릭(`sqlquery`)을 coordinator 의 otelcol 에서 빼고 standby 의 otelcol 로 옮김 (가이드 주의 2, 6-6절). standby 가 없으면 그대로 |
| `wem_use_vip` | `false` | `true` 면 standby 의 `sqlquery` 가 coordinator IP 대신 `whpg_vip_cidr` 주소로 접속. VIP 가 coordinator 에 활성이어야 함 |
| `wem_whpg_port` | `5432` | 모니터링 대상 WHPG 포트 |
| `wem_whpg_database` | `gpadmin` | WEM 이 접속하는 WHPG DB. Host Agent 의 `WEM_DATABASE` 와 같은 값. 없으면 `wem_whpg_user` 소유로 만듦 |
| `wem_whpg_user` | `wemon` | WHPG 접속 계정 |
| `wem_whpg_password` | `changeme` | WHPG 접속 비밀번호 (SCRAM 으로 저장, 서버 로그에 남지 않게 설정) |
| `wem_whpg_create` | `wem_state_install` 의 반대 | `true` 면 coordinator 에 `wem_whpg_user` 계정과 `wem_whpg_database` 를 만듦 |
| `wem_whpg_user_superuser` | `true` | `wem_whpg_user` 를 superuser 로 만듦 |
| `wem_whpg_resource_group` | `wem_group` | `wem_whpg_user` 에 지정할 resource group. 없으면 만들고, 비워두면 건드리지 않음. `gp_resource_manager` 가 `group` / `group-v2` 일 때만 적용 |
| `wem_whpg_resource_group_options` | `CONCURRENCY: 20`, `CPU_MAX_PERCENT: 10`, `CPU_WEIGHT: 100`, `MEMORY_QUOTA: -1` | resource group 설정. 이미 있으면 값이 다른 항목만 `ALTER` |
| `wem_state_install` | `false` | `true` 면 WEM host 에 별도 상태 DB(PostgreSQL) 설치. `false` 면 WHPG coordinator 에 상태 DB(`wem`)를 둠 |
| `wem_state_host` | true: standby 호스트 / false: coordinator IP | WEM 상태 DB 주소 |
| `wem_state_port` | true: `5433` / false: `5432` | WEM 상태 DB 포트 |
| `wem_state_database` | `wem` | WEM 상태 DB 이름 (WEM 이 처음 시작할 때 만듦) |
| `wem_state_user` / `wem_state_password` | `wemon` / `changeme` | WEM 상태 DB 계정 |
| `wem_state_data_dir` | `/var/lib/pgsql/data` | (`wem_state_install=true`) 상태 DB 데이터 디렉터리 |
| `wem_state_service` | `postgresql-wem14` | (`wem_state_install=true`) 상태 DB systemd 서비스 이름 |
| `clickhouse_user` / `clickhouse_password` | `default` / `changeme` | ClickHouse 계정 |
| `clickhouse_native_port` | `9000` | ClickHouse native 포트 |
| `clickhouse_data_dir` / `clickhouse_log_dir` | `""` | 비우면 기본 위치. 지정하면 `data-path.xml` / `log-path.xml` 생성 |
| `wem_port` | `8080` | WEM 웹 포트 |
| `wem_agent_port` | 웹 포트 + 1 (자동) | Host Agent 가 접속하는 포트 |
| `wem_admin_password` | `changeme_wem` | WEM `admin` 초기 비밀번호 |

자동 계산되어 보통 바꾸지 않는 변수: `coordinator_host`, `wem_host`(WEM 설치 호스트, standby 우선), `coordinator_ip`, `wem_host_ip`, `vip_addr`, `standby_db_host`.

### 6-2. WEM 1.2 삭제 (`01-cleanup-wem12.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `wem_drop_old_state_db_in_whpg` | `true` | WHPG 의 `wem_state_database`(1.2 상태 DB)를 삭제. 보호 목록에 있는 이름이면 지우지 않음 |
| `cleanup_services` | wem, prometheus, alertmanager, loki, alloy | 중지하고 비활성화할 서비스 |
| `cleanup_packages` | WEM, Prometheus, Alertmanager, Loki, Alloy, collector 패키지 | 삭제할 패키지 |
| `cleanup_paths_all_hosts` | `/etc/alloy`, `/var/lib/alloy`, `/var/lib/whpg-observability-collector` | 모든 노드에서 삭제할 경로 |
| `cleanup_paths_wem_host` | `/etc/wem`, `/etc/prometheus`, `/etc/loki`, `/data1/loki` 등 | WEM host 에서만 삭제할 경로 |
| `cleanup_users` | prometheus, alertmanager, loki | 삭제할 계정 |

운영 환경의 1.2 설치 경로가 다르면 `cleanup_*` 목록을 맞춥니다. 확인 방법은 `rpm -qa | grep -iE "wem|loki|prometheus|alloy"` 와 `systemctl list-unit-files | grep -iE "wem|loki|prometheus|alloy"` 입니다.

### 6-3. pg_hba / ClickHouse 패키지 (`03-pg-hba.yml`, `04-clickhouse.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `wem_manage_pg_hba` | `true` | coordinator 와 standby 의 `pg_hba.conf` 에 `wem_whpg_user` 의 접속 허용 블록(127.0.0.1 과 클러스터 노드 IP, scram-sha-256)을 추가하고 `gpstop -u` 로 반영 |
| `clickhouse_packages` | `edb-clickhouse-server`, `edb-clickhouse-client` | 설치할 ClickHouse 패키지 |

### 6-4. WEM 1.4 설치 (`06-wem.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `wem14_package` | `whpg-enterprise-manager-1.4.0` | 설치할 WEM 패키지 (버전 고정) |
| `wem_config_file` | `/etc/wem/wem.conf` | WEM 설정 파일 경로 |
| `wem_unix_group` | `gpadmin` | `wem.conf` 의 그룹 |
| `wem_insecure_cookies` | `"1"` | HTTP 로 접속하므로 쿠키 허용. 웹 포털 HTTPS 설정 파라미터는 공식 설정 참조에 없어 구성하지 않음 |
| `wem_high_load_mode` | `""` | 대규모 환경이면 `"true"` (`HIGH_LOAD_MODE`) |
| `wem_pool_max_conns` | `"40"` | WEM DB 연결 풀 최대 연결 수 (`WEM_POOL_MAX_CONNS`). `""` 이면 넣지 않음 |
| `wem_otel_prometheus_scrape_endpoint` | `0.0.0.0:8889` | 외부 Prometheus 가 메트릭을 가져가는 주소 |
| `wem_extra_conf_lines` | `[]` | `wem.conf` 에 그대로 추가할 줄 목록 |
| `wem_manage_firewall` | `true` | firewalld 가 실행 중이면 WEM, Agent, ClickHouse 포트를 연다 |

### 6-5. Host Agent / otelcol (`07-host-agent.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `host_agent_package` | `edb-acp-host-agent` | 버전을 고정하려면 `edb-acp-host-agent-1.1.0` |
| `otelcol_package` | `edb-otelcol` | 버전을 고정하려면 `edb-otelcol-0.158.0` |
| `host_agent_conf` | `/etc/edb/acp-host-agent/acp-host-agent.conf` | Host Agent 설정 파일 |
| `host_agent_join_token_file` | `/etc/edb/acp-host-agent/join-token` | 자동 승인용 토큰 파일 |
| `host_agent_user` | `acp-host-agent` | 서비스 계정. `join-token` 의 소유자 |

### 6-6. otelcol 설정 조정 (`otelcol-local-config.yml`, `08-otelcol-tune.yml`)

WEM 이 배포한 `config.yaml` 은 `acp-host-agent` 를 재시작하면 다시 렌더링되어 직접 고친 값이 원복됩니다.
그래서 `config.yaml` 은 건드리지 않고 override 파일 `config_local.yaml` 을 만들어 `edb-otelcol.conf` 의
`OTELCOL_OPTIONS` 에 `--config` 로 함께 읽게 합니다.

적용 순서:

1. `07-host-agent`: `acp-host-agent` 를 시작하기 **전에** `config_local.yaml` 과 `edb-otelcol.conf` 를 넣어 둡니다.
   agent 가 승인 후 otelcol 을 처음 띄울 때부터 조정 값(`start_at: end` 등)으로 동작하므로, WEM 기본값(`start_at: beginning`)으로
   기존 로그를 처음부터 읽는 구간이 없습니다. (이때는 WEM 의 `config.yaml` 이 없어 validate 하지 않고, standby 의 sqlquery 추가는 쿼리 파일이 생긴 뒤인 2 로 미룸)
2. `08-otelcol-tune`: WEM 이 `config.yaml` 을 배포한 뒤 `edb-otelcol validate` 로 병합 결과를 검사하고,
   바뀐 내용(standby 의 sqlquery 추가, 변수 변경 등)이 있을 때만 otelcol 을 재시작합니다.

- 모든 노드: `file_log` (`start_at: end`, `poll_interval`) 조정
- coordinator failover 대비 (가이드 "주의 2", `wem_sqlquery_on_standby: true` 이고 standby 가 있을 때):
  - 2-1: coordinator 의 `/etc/edb/otelcol/whpg_db_metrics.yaml` 을 standby 로 복사
  - 2-2: coordinator 의 `config_local.yaml` 에 `service.pipelines.metrics.receivers: [host_metrics]` 를 넣어 `sqlquery/whpg` 를 제외
    (WEM 의 `config.yaml` 에 `sqlquery/whpg` 정의는 남지만 파이프라인에서 빠져 수집하지 않음)
  - 2-3: standby 의 `config_local.yaml` 에 `sqlquery/whpg`(접속 `otelcol_sqlquery_host`, 계정 `wem_whpg_*`, `sslmode=prefer`, 30s)를 추가하고
    `receivers: [host_metrics, sqlquery/whpg]`
  - 접속 주소는 기본이 coordinator 실 IP 이고, `wem_use_vip=true` 면 VIP 입니다. coordinator 서버가 내려간 뒤에도 계속 수집하려면 VIP 를 씁니다.

| 변수 | 기본값 | 설명 |
|---|---|---|
| `otelcol_config` | `/etc/edb/otelcol/config.yaml` | WEM 이 배포하는 otelcol 설정 (직접 수정하지 않음) |
| `otelcol_local_config` | `/etc/edb/otelcol/config_local.yaml` | 조정 값을 넣는 override 파일 (`edb-otelcol` 소유 0600) |
| `otelcol_options_file` | `/etc/edb/otelcol/edb-otelcol.conf` | `OTELCOL_OPTIONS` 에 override 파일을 추가하는 서비스 환경 파일 |
| `otelcol_bin` | `/usr/edb/otelcol/bin/edb-otelcol` | `validate` / `print-config` 에 쓰는 실행 파일 |
| `otelcol_checkpoint_dir` | `/var/lib/edb/otelcol/checkpoint` | 로그 읽기 위치 저장소 |
| `otelcol_reset_checkpoint` | `false` | `true` 면 08 단계에서 otelcol 을 재시작할 때 checkpoint 를 초기화 (보통 필요 없음) |
| `agent_approval_wait_seconds` | `240` | WEM 이 otelcol 설정을 배포할 때까지 기다리는 최대 시간(초) |
| `otelcol_filelog_poll_interval` | `1s` | 로그 수집기의 `poll_interval` (`start_at` 은 `end`) |
| `otelcol_whpg_queries_file` | `/etc/edb/otelcol/whpg_db_metrics.yaml` | coordinator 에서 standby 로 복사하는 쿼리 정의 (주의 2-1) |
| `otelcol_sqlquery_host` | `wem_use_vip` 가 true 면 VIP, 아니면 coordinator IP | standby `sqlquery/whpg` 의 접속 주소 |
| `otelcol_sqlquery_collection_interval` | `30s` | standby `sqlquery/whpg` 수집 주기 |
| `otelcol_sqlquery_sslmode` | `prefer` | standby `sqlquery/whpg` 의 `sslmode` (접속 계정은 `wem_whpg_*`) |

적용 결과 확인:

```bash
/usr/edb/otelcol/bin/edb-otelcol print-config \
  --config=file:/etc/edb/otelcol/config.yaml --config=file:/etc/edb/otelcol/config_local.yaml
```

### 6-7. Coordinator Query Logs (`09-query-logs.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `coordinator_query_logs_enabled` | `true` | ClickHouse 객체(상태 테이블, Materialized View, View) 생성 여부 |
| `coordinator_query_logs_recreate` | `false` | `true` 면 기존 객체를 삭제하고 다시 생성 (수집된 데이터도 삭제됨) |
| `coordinator_query_logs_ttl_days` | `90` | 상태 테이블 TTL(일) |
| `coordinator_query_logs_wait_seconds` | `300` | `otel_logs` 테이블이 생길 때까지 기다리는 최대 시간(초). 초과하면 건너뜀 |

### 6-8. rpm 업그레이드 (`upgrade.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `wem_upgrade_packages_wem_host` | `whpg-enterprise-manager`, `edb-clickhouse-server`, `edb-clickhouse-client` | WEM host 에서 최신 버전으로 올릴 패키지 |
| `wem_upgrade_packages_all_hosts` | `edb-acp-host-agent`, `edb-otelcol` | 모든 노드에서 최신 버전으로 올릴 패키지 |

### 6-9. WEM 1.4 삭제 (`remove-wem14.yml`)

| 변수 | 기본값 | 설명 |
|---|---|---|
| `wem_remove_version_prefix` | `""` | `whpg-enterprise-manager` 를 이 버전으로 시작할 때만 삭제. 빈 값이면 버전과 관계없이 삭제 |
| `wem_remove_autoremove` | `true` | 삭제한 패키지와 함께 설치되었고 더 이상 쓰이지 않는 의존 패키지도 삭제 |
| `wem_remove_databases` | `["wem"]` (`wem_state_database`) | 소유자와 관계없이 지우는 WHPG 데이터베이스 (접속 세션은 끊음) |
| `wem_protected_databases` | `postgres`, `template0`, `template1`, `gpadmin`, 클러스터 기본 DB | 어떤 경우에도 지우지 않는 데이터베이스 |
| `wem_remove_pg_hba_block` | `true` | `pg_hba.conf` 의 WEM 블록 삭제 |
| `wem_drop_whpg_roles` | `true` | WHPG 의 WEM role / DB / resource group 삭제 |
| `wem_whpg_roles` | `["wem", "wemadmin"]` | 함께 삭제할 (1.2 등의) role 목록 |
| `remove_services`, `remove_packages`, `remove_paths_all_hosts`, `remove_paths_wem_host`, `remove_paths_wem14_only` | 파일 참고 | 삭제 대상 서비스, 패키지, 경로 |

## 7. 단계별 실행 (tags)

`-t` 로 일부 단계만 실행합니다. 실행 전 확인(`00-checks.yml`)과 마지막 상태 확인(`10-summary.yml`)은 항상 실행됩니다.

| tag | 태스크 파일 (`roles/warehouse-pg-wem/tasks/`) | 내용 |
|---|---|---|
| `cleanup` | `01-cleanup-wem12.yml` | WEM 1.2 삭제 |
| `pghba` | `03-pg-hba.yml` | WHPG `pg_hba.conf` 허용 추가 |
| `clickhouse` | `04-clickhouse.yml` | ClickHouse 설치 |
| `statedb` | `05-state-database.yml` | WEM 접속 계정 / DB / resource group, (선택) 별도 상태 DB |
| `wem` | `06-wem.yml` | WEM 1.4 설치와 `wem.conf` |
| `agent` | `07-host-agent.yml` | Host Agent, otelcol 설치, otelcol 조정 선적용, 자동 승인 |
| `tune` | `08-otelcol-tune.yml` | otelcol 설정 검사와 반영 |
| `querylogs` | `09-query-logs.yml` | ClickHouse Coordinator Query Logs 객체 |
| `upgrade` | `upgrade.yml` | 관련 rpm 을 repo 최신 버전으로 업그레이드 (`-t upgrade` 를 지정할 때만) |
| `remove` | `remove-wem14.yml` | WEM 1.4 삭제 (`wem_uninstall.yml` 에서만) |

```bash
ansible-playbook playbooks/wem_deploy.yml -t tune,querylogs
ansible-playbook playbooks/wem_deploy.yml --list-tasks            # 실행될 태스크 목록만 확인
```

조합할 때 주의할 점입니다.

- `agent` 는 `wem` 단계가 만든 join secret 을 쓰므로 **`-t wem,agent` 로 함께** 실행합니다. `-t agent` 만으로는 secret 이 없어 동작하지 않습니다.
- `-t wem` 을 다시 실행하면 join secret 과 cookie secret 이 새로 만들어지고 WEM 이 재시작됩니다. 에이전트의 `join-token` 도 같이 갱신하려면 `-t wem,agent` 로 실행합니다.
- `tune` 의 조정 값은 `config_local.yaml` 에 있으므로 WEM 이 `config.yaml` 을 다시 배포하거나 `acp-host-agent` 를 재시작해도 유지됩니다. 조정 값을 바꾼 경우에만 `-t tune` 으로 다시 적용합니다.
- 에이전트가 자동 승인되지 않으면(WEM 화면 Management > Host Agents 에서 수동 승인한 뒤) `-t tune,querylogs` 로 이어서 실행합니다.

## 8. 설치 후 확인과 접속 방법

### 8-1. WEM 화면

- 주소: `http://<WEM host IP>:<wem_port>/` (기본 8080)
- 계정: `admin`, 비밀번호는 `wem_admin_password`
- **Management > Host Agents** 에서 노드가 모두 승인(approved) 상태인지 확인합니다. 자동 승인에 실패하면 여기서 수동 승인합니다. 권한 오류가 나면 **Management > Permissions** 에서 Admin 의 Approve Host Agent 를 켭니다.
- 시간 그래프는 **데이터가 쌓여야** 보입니다. 설치 직후 `Last 24 Hours` / `Gap 15 min` 이면 점이 1개뿐이라 그래프가 비어 보입니다. 시간 범위를 `Last 15 minutes`, Gap 을 `1 min` 으로 줄이거나 30분 이상 기다립니다.

### 8-2. 서비스 상태

```bash
ansible all -m shell -a 'systemctl is-active acp-host-agent edb-otelcol'
ansible <WEM host 이름> -m shell -a 'systemctl is-active wem clickhouse-server'
```

### 8-3. ClickHouse 접속 (WEM host, 포트 9000 / 8123)

인자 없이 `clickhouse` 만 치면 로컬 모드가 열려 `acp_observability` 가 보이지 않습니다. 반드시 `clickhouse-client` 에 주소를 줍니다.

```bash
clickhouse-client --host <WEM host IP> --user default --password
```

```sql
SHOW DATABASES;                                -- acp_observability 가 보여야 함
USE acp_observability;
SHOW TABLES;
SELECT * FROM coordinator_query_logs LIMIT 5;
```

### 8-4. WEM 상태 DB 접속

- `wem_state_install=false`(기본): WHPG coordinator 의 `wem` DB 입니다. coordinator 에서 gpadmin 으로 `psql -d wem` 또는 `psql -h <coordinator IP> -U wemon -d wem`.
- `wem_state_install=true`: WEM host 의 PostgreSQL(포트 5433, `127.0.0.1` 에서만 열림). WEM host 에서 `cd /tmp; runuser -u postgres -- /usr/bin/psql -p 5433 -d wem`.

## 9. 문제 해결

| 증상 | 원인과 해결 |
|---|---|
| `generate_inventory.yml` 이 `primary coordinator 가 없습니다` 로 중단 | 메시지의 "서버별 원인"을 봅니다. `SSH 실패 - Permission denied` 면 그 서버가 받는 키가 `ansible_ssh_common_args` 의 `IdentityFile` 에 없음(키 추가) 또는 계정이 다름(`ansible_user`). `recovery 상태(standby)` 만 있으면 VIP 가 standby 에 붙어 있거나 failover 된 상태. `whpg_coordinator_hosts` 의 coordinator 를 현재 primary coordinator 로 고치거나 `-e whpg_coordinator=<IP>` 로 지정 |
| `wemon` 로그인 실패 (`does not have a valid SCRAM verifier`) | 비밀번호가 md5 로 저장됨. `-t statedb` 를 다시 실행하면 SCRAM 으로 다시 설정 |
| `Timeout when waiting for 127.0.0.1:5433` (`wem_state_install=true`) | SELinux 가 PostgreSQL 에 허용한 포트는 5432 와 9898 뿐. 이 role 이 `semanage port` 로 자동 허용하지만, 수동으로는 `semanage port -a -t postgresql_port_t -p tcp 5433` |
| Host Agent 로그에 `awaiting TLS material` | 자동 승인이 안 됨. `join-token` 이 `acp-host-agent` 계정 소유 0600 이어야 한다. 안 되면 WEM 화면에서 수동 승인한 뒤 `-t tune,querylogs` |
| `wem_use_vip=true 인데 조건이 맞지 않습니다` | VIP 가 비어 있거나, standby 가 없거나, VIP 가 coordinator 에 올라와 있지 않음. `wem_use_vip` 없이 실행 |
| 그래프가 비어 있음 | 8-1절 참고 (데이터 부족). Gap 과 시간 범위를 줄인다 |
| coordinator 로그에 `unexpected EOF on client connection` 오류, Host Agent 로그에 `whpg-helper ... unexpected COPY_IN result` | WEM 로그 스캔이 `COPY ... FROM STDIN` 을 섞어 보내는데 WHPG 7 의 `psql`(12.12)이 처리하지 못함. 임시 방법은 `wem_extra_conf_lines: ["LOG_SCAN_ENABLED=false"]` 이지만 **효과는 아직 확인하지 못함** |
| `wem setup --check` 가 `dashboard_users` 등 Missing 으로 표시 | `wem_whpg_database` 와 `wem_state_database` 가 같은 서버에서 이름만 다르면 검사 도구가 다른 DB 를 보는 것으로 보임. 실제 테이블은 `wem` DB 에 있고 WEM 은 정상 동작 |
| otelcol 조정이 사라짐 | `edb-otelcol.conf` 의 `OTELCOL_OPTIONS` 에 `--config .../config_local.yaml` 이 있는지 확인. 없으면 `-t tune` 으로 다시 적용 |
| `otelcol` 로그에 `5317` 연결 실패 경고 | 게이트웨이(WEM host)의 otelcol 이 재시작되는 동안 일시적으로 생김. 이후 계속되면 WEM host 의 5317 포트와 방화벽을 확인 |

## 10. 운영 환경 적용 전 점검

- `wem_drop_old_state_db_in_whpg`: 기본이 `true` 라 WHPG 의 `wem` DB 를 지웁니다. 1.2 의 알림 규칙 등 운영 데이터가 있으면 `false` 로 두거나 먼저 `pg_dump` 로 백업합니다.
- `wem_whpg_database` 를 업무 DB 로 지정하면 WEM 이 그 DB 에 `wem` 스키마 객체를 만듭니다. 삭제할 때는 `wem_whpg_user` 소유 객체만 지우고, 보호 목록(`wem_protected_databases`)의 DB 와 다른 계정 소유 DB 는 지우지 않습니다.
- 비밀번호(`wem_admin_password`, `clickhouse_password`, `wem_whpg_password`, `wem_state_password`)를 운영 환경 값으로 바꿉니다.
- 운영 환경의 1.2 설치 경로가 다를 수 있습니다. `cleanup_*` 목록(6-2절)을 확인합니다.
- WEM 은 standby(없으면 coordinator)에 설치됩니다.
- 모든 노드에서 `dnf info whpg-enterprise-manager-1.4.0 edb-acp-host-agent edb-otelcol edb-clickhouse-server` 로 패키지를 받을 수 있는지 확인합니다(repo 는 이 Ansible 이 구성하지 않음).
- DB 메트릭은 기본으로 standby 에서 수집합니다(주의 2). VIP HA 환경이면 `wem_use_vip=true` 로 실행해 failover 후에도 VIP 로 계속 수집하게 합니다.
- 확인 질문 없이 삭제가 시작되므로 대상 서버가 맞는지 `ansible all -m ping` 과 `--list-tasks` 로 먼저 봅니다.

## 11. role 구성

모든 작업은 role 하나(`roles/warehouse-pg-wem`)에 있고, 단계별로 태스크 파일이 나뉘어 있습니다.
`tasks/main.yml` 이 아래 파일을 순서대로 가져오며(`import_tasks`), 각 단계에 tag 가 붙어 있습니다.

```
roles/warehouse-pg-wem/
├── defaults/main.yml            # 변수 전체 (6절)
├── tasks/
│   ├── main.yml                 # 단계 순서와 tag 정의
│   ├── 00-checks.yml            # (always)    실행 전 확인: OS, VIP 활성 여부, wem_use_vip 조건, 배치 출력
│   ├── 01-cleanup-wem12.yml     # cleanup     WEM 1.2 삭제
│   ├── 03-pg-hba.yml            # pghba       pg_hba.conf 허용 추가
│   ├── 04-clickhouse.yml        # clickhouse  ClickHouse 설치
│   ├── 05-state-database.yml    # statedb     WEM 접속 계정 / DB / resource group, (선택) 별도 상태 DB
│   ├── 06-wem.yml               # wem         WEM 1.4 설치
│   ├── 07-host-agent.yml        # agent       Host Agent, otelcol 설치, otelcol 조정 선적용, 자동 승인
│   ├── 08-otelcol-tune.yml      # tune        otelcol 설정 검사 (WEM 배포 후 validate, 변경 시 재시작)
│   ├── otelcol-local-config.yml # (07, 08 에서 호출) config_local.yaml + edb-otelcol.conf 적용
│   ├── 09-query-logs.yml        # querylogs   Coordinator Query Logs 객체
│   ├── upgrade.yml              # upgrade     관련 rpm 최신 버전으로 업그레이드 (-t upgrade 지정 시에만)
│   ├── 10-summary.yml           # (always)    서비스 상태 확인
│   └── remove-wem14.yml         # remove      WEM 1.4 삭제 (wem_uninstall.yml 이 호출)
├── files/
│   └── query_logs_*.sql         # Coordinator Query Logs 객체 SQL
└── templates/
    ├── wem.conf.j2              # WEM 설정 파일
    └── otelcol_config_local.yaml.j2  # otelcol override (config_local.yaml)
```

`wem_uninstall.yml` 은 같은 role 에서 `00-checks.yml` 과 `remove-wem14.yml` 만 골라서 실행합니다(`tasks_from`).

## 12. 알려진 제약과 검증 현황

### 알려진 제약

- **웹 포털 HTTPS 는 구성하지 않습니다.** 공식 설정 참조에 웹 포털 HTTPS 를 켜는 파라미터가 없어 HTTP 와 `WEM_INSECURE_COOKIES=1` 로 구성합니다. 이 때문에 WEM 화면의 클립보드 기능(예: Copy Query)이 제한될 수 있습니다.
- **로그 스캔 오류**: 9절의 `unexpected COPY_IN result` 가 발생할 수 있습니다. WEM 1.4 와 WHPG 7 의 `psql` 호환 문제로 보입니다.
- **`DurationMs` 가 `NULL`**: coordinator 의 `log_min_duration_statement` 가 꺼져 있으면 Query Logs 의 `DurationMs` 가 비어 있습니다. 이 playbook 은 해당 설정을 바꾸지 않습니다.
- **coordinator failover 와 DB 메트릭**: 기본으로 DB 메트릭(`sqlquery`)을 standby 의 otelcol 이 coordinator 실 IP 로 수집합니다(주의 2). coordinator 서버가 내려가 standby 가 승격되면 이 접속이 끊기므로, failover 후에도 이어서 수집하려면 VIP 와 `wem_use_vip=true` 를 씁니다.

### 검증 현황

확인한 것 (4노드: coordinator, standby, segment 2대):

- `generate_inventory.yml` 로 만든 inventory 가 기존 inventory 와 호스트 이름 / IP / 역할 / hostname 이 같음. standby(recovery) 자동 건너뜀, coordinator 지정 / 미지정 오류 처리
- WEM 1.4 삭제 후 신규 설치 전체 흐름 반복 (기본 옵션, `wem_state_install=false`)
- Host Agent 자동 승인 (UI 승인 없이 4대 모두 approved)
- `wemon` SCRAM 로그인, resource group `wem_group` 생성 / 변경 / 지정, 비밀번호가 서버 로그에 남지 않음
- otelcol: `config_local.yaml` 이 agent 시작 전에 적용되어 otelcol 첫 기동부터 조정 값으로 동작, 설치 이전 로그가 수집되지 않음(0건), `acp-host-agent` 재시작 후에도 유지
- DB 메트릭(sqlquery) 수집 에러 0건, ClickHouse 에 메트릭과 로그 적재, WEM 웹 200
- 주의 2 (2-1 ~ 2-3): coordinator 는 metrics 파이프라인에서 sqlquery 제외, standby 가 coordinator IP 로 수집(쿼리 파일 복사). 전환 전후 분 단위로 coordinator → standby 로 넘어가고 겹치거나 비는 구간 없음, 다시 실행해도 변경 없음
- 삭제: rpm 0개, `wem` DB / `wemon` / `wem_group` 삭제, 보호 목록 DB 와 다른 계정 소유 DB 는 SKIP, 접속 세션이 있는 DB 도 삭제

아직 확인하지 못한 것:

- `wem_use_vip=true` 실행 (VIP 비활성 환경이라 병합 설정의 validate 까지만 확인)
- `wem_state_install=true` 를 최근 변경 이후 실행
- coordinator failover 때 수집이 이어지는지
- WEM 1.2 가 **설치된 상태에서** 바로 전환하는 흐름 (삭제 후 신규 설치로만 확인)
- `LOG_SCAN_ENABLED=false` 로 로그 스캔 오류가 사라지는지
