```bash
┌─────────────────── [main] ───────────────────┐
                          │                                              │
server1 ──┐  node_exporter ───────────────► Prometheus (9090) ─┐          │
          │  promtail ──────────────────►─┐                    │          │
          │                               │                    ▼          │
server2 ──┤  node_exporter ──────────────►─┤            Alertmanager (9093)│
          │  promtail ──────────────────►─┤                    │          │
          │                               │                    ▼          │
win2012 ──┘  windows_exporter ───────────►─┘            postfix (25)       │
             (promtail 없음)               │                    │          │
                                           ▼                    │          │
                                       Loki (3100)               │          │
                                           │                    │          │
                                           ▼                    │          │
                                     Grafana (3000)              │          │
                                          (시각화)                │          │
                          └───────────────────────────────────────┼────────┘
                                                                   ▼
                                                        원타임 메일
                                                  (medalo3920@itquoted.com)
```

<aside>

http://main.example.com:9090

medalo3920@itquoted.com

ALERTS{alertname="InstanceDown"}

ALERTS{alertname="HighCpuUsage"}

ALERTS{alertname="ServerRebooted"}

</aside>

## 사전 상태

- **main**: Prometheus(9090), Loki(3100), Grafana(3000)
- **server1/2**: node_exporter(9100), promtail
- **win2012**: windows_exporter 설치 아직 안함

이 위에 "임계점 초과 감지 → 메일 발송" 체계를 얹는 게 이번 작업.

**전체 순서**: Alertmanager 설치 → 규칙 작성 → Prometheus와 연결 → 메일 설정 → 테스트

**핵심 원칙 — 알림 판단은 Prometheus가 한다**
CPU·다운·재부팅 같은 조건을 "넘었는지" 판단하는 건 숫자 메트릭을 가진 Prometheus의 몫이다. Alertmanager는 판단하지 않고, Prometheus가 보낸 알림을 받아 메일 발송·중복 묶음만 담당한다. Grafana는 시각화 전용이고 알림 경로에는 관여하지 않는다.

---

## 1단계 — Alertmanager 설치 (main)

Prometheus는 감지만 한다. 실제 메일 발송은 Alertmanager가 담당하므로 별도 설치가 필요하다.

```bash
# 전용 사용자 생성
useradd --no-create-home --shell /bin/false alertmanager

# 다운로드 및 압축 해제 (버전은 릴리스 페이지에서 확인)
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64

# 바이너리 배치
cp alertmanager amtool /usr/local/bin/
chown alertmanager:alertmanager /usr/local/bin/alertmanager /usr/local/bin/amtool

# 디렉터리 생성
mkdir /etc/alertmanager
mkdir -p /var/lib/alertmanager
chown alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```

**systemd 서비스 등록** 

```bash
vi /etc/systemd/system/alertmanager.service
```

```bash
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now alertmanager
systemctl status alertmanager           # active (running) 확인
ss -tlnp | grep 9093                     # 9093 LISTEN 확인
curl http://main.example.com:9093/-/ready   # OK 확인
```

---

## 2단계 — Prometheus 알림 규칙 작성 (main)

"언제 알림을 띄울지" 판단 조건은 Prometheus가 가진다. 그 조건을 적는 파일이 규칙 파일이다.

```bash
vi /etc/prometheus/alert.rules.yml
```

```yaml
groups:
  - name: node-alerts
    rules:
      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU 사용률 높음 ({{ $labels.instance }})"
          description: "{{ $labels.instance }} CPU 사용률이 5분간 80%를 초과했습니다."

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "서버 다운 ({{ $labels.instance }})"
          description: "{{ $labels.instance }} 가 1분 이상 응답하지 않습니다."

      - alert: ServerRebooted
        expr: changes(node_boot_time_seconds[5m]) > 0
        labels:
          severity: warning
        annotations:
          summary: "서버 재부팅 감지 ({{ $labels.instance }})"
          description: "{{ $labels.instance }} 가 재부팅되었습니다."
```

```bash
chown prometheus:prometheus /etc/prometheus/alert.rules.yml
```

| 알림 | 의미 |
| --- | --- |
| HighCpuUsage | 모든 코어 평균 사용률이 5분 지속 시 |
| InstanceDown | node_exporter 무응답 1분 지속 시 |
| ServerRebooted | 최근 5분 내 부팅시각 변화 = 재부팅 |

> 
> 
> 
> **1분(`offset 1m`)으로 안 한 이유** : 
> 
> 재부팅하면 node_exporter도 꺼졌다 켜짐. `offset 1m`이면 "1분 전 값"이 재부팅 중 구간이라 데이터가 없음 → 비교 대상이 없어서 식이 빈 값 → 알림 안 뜸.
> 

> 
> 
> 
> **`changes()` 쓴 이유 :**
> 
> 두 시점을 빼서 비교하는 방식이 아니라, 한 메트릭이 시간창 안에서 변했는지만 셈. "옛날 값"에 의존하지 않으니 비교 대상이 사라지는 문제 자체가 없음.
> 

---

## 3단계 — Prometheus와 Alertmanager 연결 (main)

Prometheus가 규칙 파일을 읽고, 알림 발생 시 Alertmanager로 보내도록 연결한다.

```bash
vi /etc/prometheus/prometheus.yml
```

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "localhost:9093"

rule_files:
  - "/etc/prometheus/alert.rules.yml"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: 'linux-servers'
    static_configs:
      - targets: ['server1.example.com:9100', 'server2.example.com:9100']
  - job_name: 'windows-servers'
    static_configs:
      - targets: ['win2012.example.com:9182']
```

```bash
promtool check config /etc/prometheus/prometheus.yml   # SUCCESS 확인
systemctl restart prometheus
systemctl status prometheus
```

---

## 4단계 — win2012 모니터링 대상 추가

windows_exporter = node_exporter의 윈도우 버전. 윈도우 서버의 CPU·메모리·디스크 지표를 9182 포트로 내보내 Prometheus가 수집하게 한다.

**main의 `/etc/hosts`에 win2012 등록:**

```bash
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.10.10  main.example.com     main
192.168.10.20  server1.example.com  server1
192.168.10.30  server2.example.com  server2

192.168.10.201   win2012.example.com win2012
```

```bash
# 테스트
ping -c1 win2012.example.com
```

**windows_exporter 설치 (win2012)**

https://github.com/prometheus-community/windows_exporter/releases

```bash
# 설치 전 확인
echo %PROCESSOR_ARCHITECTURE%     :: AMD64 확인
winver                            :: Windows Server 2012 R2 확인
```

```bash
# 설치 후 확인
sc query windows_exporter         :: STATE 4 RUNNING 확인
netstat -ano | findstr 9182       :: LISTENING 확인
```

!image.png

**윈도우 방화벽 9182 허용 (관리자 PowerShell):**

```powershell
# 설치 후 확인
New-NetFirewallRule -DisplayName "windows_exporter" -Direction Inbound -LocalPort 9182 -Protocol TCP -Action Allow
```

!image.png

`http://main.example.com:9090` → Alerts 메뉴에서 규칙 3개, Status → Targets에서 타겟 UP 확인.

!image.png

---

## 5단계 — 메일 설정 (main)

알림을 관리자에게 자동 통보한다. 경로는 `Alertmanager → postfix → 원타임 메일`.

### **postfix 설치**

```bash
dnf install -y postfix
systemctl enable --now postfix
systemctl status postfix          # active (running) 확인
ss -tlnp | grep :25               # 25번 포트 LISTEN 확인
```

### **alertmanager.yml 메일 설정**

```bash
vi /etc/alertmanager/alertmanager.yml
```

```yaml
global:
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@main.example.com'
  smtp_require_tls: false

route:
  receiver: 'email-notify'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h

receivers:
  - name: 'email-notify'
    email_configs:
      - to: 'medalo3920@itquoted.com'
        send_resolved: true
```

| 항목 | 의미 |
| --- | --- |
| `smtp_smarthost` | 메일을 로컬 postfix(25)로 넘김 |
| `to` | 수신지 = 원타임 메일 |
| `repeat_interval: 1h` | Firing 지속 시 1시간마다 재발송 |

**검증 및 적용**

```bash
amtool check-config /etc/alertmanager/alertmanager.yml   # SUCCESS 확인
systemctl restart alertmanager
systemctl status alertmanager
```

---

## 6단계 — 테스트

### **1) InstanceDown 테스트** — win2012에서 node_exporter 중지:

```bash
sc stop windows_exporter

# STOPPED 확인
sc query windows_exporter
```

<img width="1099" height="383" alt="image" src="https://github.com/user-attachments/assets/72481ba6-9c0c-444d-8b53-ee7b2603a7ab" />

<img width="1111" height="340" alt="image" src="https://github.com/user-attachments/assets/1f31938b-89ee-4879-8fd1-7bcfdc4695b0" />


<img width="591" height="605" alt="image" src="https://github.com/user-attachments/assets/41708563-de87-4bc1-a0fc-e74335ffc2a2" />


**✅ win2012 exporter 중지 후, Prometheus Alerts에서 Pending → Firing 전환을 거쳐 원타임 메일함에 알림 메일이 도착함을 확인**

**복구:**

```bash
sc start windows_exporter
```

---

### **2) CPU 테스트** — server1 에서 코어 수만큼 부하:

```bash
nproc                    # 코어 수 확인 
yes > /dev/null &        # 코어 수만큼 반복 실행
top                      # 사용률 100% 유지 확인
# 5분 유지 → HighCpuUsage Firing
killall yes              # 부하 종료
```

<img width="1270" height="790" alt="image" src="https://github.com/user-attachments/assets/301e7cef-686f-4682-b5ad-25412358748d" />


<img width="1136" height="797" alt="image" src="https://github.com/user-attachments/assets/f94306a7-ec8d-4177-adf1-e7b3ef530c88" />


<img width="1119" height="779" alt="image" src="https://github.com/user-attachments/assets/d8421f50-f5a6-4b7c-bec5-e6d49f063af1" />


<img width="549" height="786" alt="image" src="https://github.com/user-attachments/assets/0fd02b64-480d-4e3b-80a4-ce258946c4cf" />

**✅ server에 코어 수만큼 부하를 준 뒤, Prometheus Alerts에서 5분 경과 후 Pending → Firing 전환을 거쳐 원타임 메일함에 알림 메일이 도착함을 확인**

---

### **3) 재부팅 테스트** — server1에서 `reboot` 실행

<img width="1122" height="307" alt="image" src="https://github.com/user-attachments/assets/72b499db-2b5f-49f3-9798-5fffa957ebbb" />


<img width="577" height="478" alt="image" src="https://github.com/user-attachments/assets/da27839b-0284-4cd1-8629-624ba2fdd6f8" />


**✅**  **server 재부팅 후, Prometheus Alerts에서 ServerRebooted가 warning되어 원타임 메일함에 알림 메일이 도착함을 확인**

---

## 알림 상태 흐름

```
조건 충족 → Pending → (for 시간 경과) → Firing → (group_wait) → 메일 발송
```

| 단계 | 설명 | 메일 |
| --- | --- | --- |
| 조건 충족 | CPU 80% 초과 등 조건 참 | 발송 안 함 |
| Pending | 규칙의 `for` 시간 동안 조건 유지 관찰 (InstanceDown 1분 / HighCpuUsage 5분) | 발송 안 함 |
| Firing | `for` 시간 경과, Prometheus가 Alertmanager로 전달 | — |
| group_wait | Alertmanager가 30초 대기 (유사 알림 묶음용) | — |
| 메일 발송 | postfix 거쳐 원타임 메일로 도착 | 발송 |
- **Pending 중에는 메일이 나가지 않는다.** Firing 전환 후 `group_wait`(30초)를 거쳐야 발송된다.
- 부하가 `for` 시간을 못 채우고 끊기면 타이머는 처음부터 다시 시작된다.
