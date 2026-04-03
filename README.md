# Ubuntu Ops Scripts

> Ubuntu / Linux 운영 환경에서 실제로 사용한 자동화 Shell Script 모음입니다.
> 서버 초기 구축부터 Kubernetes 클러스터 설치, Nginx SSL 설정까지
> 현장에서 반복 수행하는 작업을 스크립트로 자동화한 경험을 담고 있습니다.

---

## 목차

- [저장소 소개](#저장소-소개)
- [사용 배경](#사용-배경)
- [디렉토리 구조](#디렉토리-구조)
- [카테고리별 설명](#카테고리별-설명)
- [예시 실행 방법](#예시-실행-방법)
- [사용 시 주의사항](#사용-시-주의사항)
- [개선 예정 항목](#개선-예정-항목)

---

## 저장소 소개

이 저장소는 Ubuntu 기반 서버 운영 자동화를 위해 작성한 Shell Script 모음입니다.  
AWS EC2 인스턴스 프로비저닝, Kubernetes 클러스터 구축, Nginx + Let's Encrypt SSL 설정 등  
인프라 운영 시 반복되는 작업을 자동화하고 재현 가능하게 만드는 데 초점을 맞췄습니다.

| 항목 | 내용 |
|------|------|
| OS | Ubuntu 22.04 LTS |
| 주요 대상 환경 | AWS EC2 (x86_64) |
| 주요 기술 | Bash, Docker, Kubernetes (kubeadm), Nginx, Certbot |

---

## 사용 배경

- **신규 서버 구축** 시 apt 업데이트 ~ 필수 도구 설치 ~ Docker 세팅까지 매번 반복되는 작업을 자동화
- **Kubernetes 클러스터 구성** 시 마스터/워커 노드 설치 절차를 스크립트화하여 실수 없이 재현
- **Nginx + HTTPS 설정** 시 도메인별 SSL 인증서 발급 및 갱신을 Docker 기반 Certbot으로 자동화
- **파티션 확장**, **Mattermost 배포** 등 단발성이지만 절차가 복잡한 작업을 문서화

---

## 디렉토리 구조

```
ubuntu-ops-scripts/
├── 01_server_build/        # 서버 초기 구축 자동화
│   ├── 1_server_build.md   # 각 스크립트 설명 문서
│   ├── 1-1_apt.sh          # apt 업데이트
│   ├── 1-2_Linux_tool.sh   # 자주 쓰는 네트워크·파일시스템 도구 설치
│   ├── 1-3_Python.sh       # Python3 + pip + venv + Jupyter 설치
│   ├── 1-4_Anaconda.sh     # Anaconda 설치 및 환경변수 등록
│   ├── 1-5_Docker.sh       # Docker Engine + Docker Compose 설치
│   ├── 1-6_mattermost.sh   # Mattermost (오픈소스 메신저) Docker 배포
│   ├── 1-7_ex_user_data.sh # AWS EC2 user-data 예시
│   └── 1-8_partition.sh    # 디스크 파티션 마운트 / LVM 확장 참고
│
├── 10_k8s_install/         # Kubernetes 클러스터 설치
│   ├── 10-1_master_node.sh # 마스터 노드 설치 (containerd + kubeadm init + Calico)
│   ├── 10-2_worker_node.sh # 워커 노드 설치 (containerd + kubeadm 준비)
│   ├── 10-3_worker_node_connect.sh  # 워커 노드 클러스터 조인
│   └── 10-4_kube_status.sh # 클러스터 상태 확인 명령어 모음
│
├── 11_k8s_script/          # Kubernetes 운영 명령어
│   └── 11-1_tutorial.sh    # 기본 kubectl 명령어 모음
│
├── 20_nginx/               # Nginx + SSL 설정
│   ├── 20-1_certbot_webroot.sh   # Certbot webroot 방식 SSL 발급
│   ├── 20-2_certbot_route53.sh   # Certbot Route53 DNS 방식 SSL 발급 (와일드카드)
│   ├── 20-3_nginx_conf.sh        # Nginx reverse proxy 설정 파일 생성
│   ├── 20-4_renew_ssl_webroot.sh # SSL 갱신 (webroot, cron용)
│   └── 20-5_renew_ssl_route53.sh # SSL 갱신 (Route53 DNS, cron용)
│
└── 99_etc/                 # 기타 유용한 스크립트
    └── 0-1_expand_root.sh  # NVMe LVM 루트 파티션 확장
```

---

## 카테고리별 설명

### 01 서버 초기 구축 (`01_server_build/`)

AWS EC2 인스턴스를 새로 띄운 직후 또는 베어메탈 서버 초기 세팅 시 사용합니다.  
`1-1` → `1-2` → `1-5` 순서로 순차 실행하면 Docker가 갖춰진 기본 서버 환경이 완성됩니다.  
`1-7_ex_user_data.sh`는 EC2 user-data에 삽입하여 인스턴스 최초 부팅 시 자동 실행하는 예시입니다.

### 10 Kubernetes 클러스터 설치 (`10_k8s_install/`)

kubeadm 기반 Kubernetes v1.33 클러스터를 구성합니다.  
컨테이너 런타임으로 `containerd`를 사용하고, CNI로 `Calico`를 적용합니다.

| 스크립트 | 실행 노드 | 역할 |
|---------|----------|------|
| `10-1_master_node.sh` | Master | containerd 설치 → kubeadm init → Calico 설치 |
| `10-2_worker_node.sh` | Worker  | containerd 설치 → kubeadm 설치 (join 대기) |
| `10-3_worker_node_connect.sh` | Worker | 마스터가 생성한 join 명령어로 클러스터 합류 |
| `10-4_kube_status.sh` | Master | 클러스터 상태 일괄 확인 |

> `set -euxo pipefail` 적용으로 오류 발생 시 즉시 중단됩니다.

### 20 Nginx + SSL 설정 (`20_nginx/`)

Docker 기반 Certbot을 활용해 Let's Encrypt 인증서를 발급하고 Nginx에 적용합니다.  
Route53 DNS 방식은 와일드카드(`*.domain.com`) 발급이 가능합니다.

```
발급  → 20-1 (webroot) 또는 20-2 (Route53 DNS ← 와일드카드 지원)
설정  → 20-3 (nginx reverse proxy conf 자동 생성)
갱신  → 20-4 / 20-5 (cron에 등록하여 자동 갱신)
```

### 99 기타 (`99_etc/`)

운영 중 단발성으로 필요한 스크립트입니다.  
`0-1_expand_root.sh`는 AWS EC2에서 EBS 볼륨 확장 후 NVMe LVM 루트 파티션을 온라인으로 확장합니다.

---

## 예시 실행 방법

### 서버 초기 세팅 (apt + tool + Docker)

```bash
# 1. 실행 권한 부여
chmod +x 01_server_build/*.sh

# 2. apt 업데이트
sudo bash 01_server_build/1-1_apt.sh

# 3. 네트워크 도구 설치
sudo bash 01_server_build/1-2_Linux_tool.sh

# 4. Docker 설치
sudo bash 01_server_build/1-5_Docker.sh
```

### Kubernetes 마스터 노드 초기화

```bash
# root 권한 필요
sudo bash 10_k8s_install/10-1_master_node.sh

# 완료 후 join 명령어 확인
cat /home/ubuntu/kubeadm-join-command.sh
```

### SSL 인증서 발급 (Route53 DNS 방식)

```bash
# AWS 자격증명이 설정된 환경에서 실행
# IAM 권한: Route53 ChangeResourceRecordSets 필요
chmod +x 20_nginx/20-2_certbot_route53.sh
sudo bash 20_nginx/20-2_certbot_route53.sh -d example.com -o ./ssl
```

### Nginx reverse proxy 설정 파일 생성

```bash
sudo bash 20_nginx/20-3_nginx_conf.sh example.com /etc/letsencrypt/live
sudo nginx -t && sudo systemctl reload nginx
```

### SSL 자동 갱신 cron 등록 (Route53)

```bash
# 매일 새벽 3시에 갱신 시도
echo "0 3 * * * root /bin/bash /path/to/20_nginx/20-5_renew_ssl_route53.sh" \
  | sudo tee /etc/cron.d/certbot-renew
```

---

## 사용 시 주의사항

> **스크립트 실행 전 반드시 내용을 확인하고, 실제 환경에 맞게 변수를 수정하세요.**

1. **하드코딩된 값 확인 필요**
   - 사용자명(`ubuntu`), Docker Compose 버전, Anaconda 버전, K8s 버전, 디스크 경로(`/dev/nvme0n1`) 등이 스크립트에 직접 명시되어 있습니다. 환경에 따라 변경이 필요합니다.

2. **root / sudo 권한**
   - 대부분의 스크립트는 `sudo` 또는 `root` 권한이 필요합니다.
   - Kubernetes 설치 스크립트(`10-1`, `10-2`)는 반드시 **root로 실행**해야 합니다.

3. **AWS 환경 전제**
   - `1-6_mattermost.sh`는 공인 IP 자동 취득을 위해 `ifconfig.me`를 사용합니다. 내부망 환경에서는 IP 취득 방식을 변경하세요.
   - `20-2_certbot_route53.sh`는 AWS Route53 권한이 설정된 IAM 자격증명이 필요합니다.

4. **1-8_partition.sh 주의**
   - 이 파일은 실행 스크립트가 아닌 **참고용 명령어 메모**입니다. 그대로 실행 시 오류가 발생합니다.

5. **멱등성(Idempotency) 미보장**
   - 일부 스크립트는 이미 설치된 환경에서 재실행 시 충돌이 발생할 수 있습니다.

---

## 개선 예정 항목

- [ ] 버전 정보를 변수로 분리하여 스크립트 상단에 명시 (Docker Compose, K8s, Calico, Anaconda 등)
- [ ] `common.sh` 또는 `.env.example`로 공통 환경변수 분리
- [ ] 모든 스크립트에 `#!/bin/bash` 셔뱅 및 `set -euo pipefail` 추가
- [ ] `1-8_partition.sh` 실행 가능한 스크립트로 전면 재작성
- [ ] `10-3_worker_node_connect.sh` bash 문법으로 수정
- [ ] `20-3_nginx_conf.sh`의 하드코딩된 proxy_pass 포트를 파라미터화
- [ ] 각 카테고리 폴더에 개별 `README.md` 추가
- [ ] CI에서 `shellcheck`으로 정적 분석 자동화
- [ ] `99_etc` → `90_storage` 등 역할이 명확한 폴더명으로 변경

---

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [performance-test-lab](../performance-test-lab) | JMeter / Locust 성능 테스트 시나리오 |
| [provisioning-monitor](../provisioning_monitor) | Terraform + Lambda 인프라 모니터링 |
| [random-draw-bluegreen-deploy](../random-draw-bluegreen-deploy) | Nginx Blue/Green 무중단 배포 |
