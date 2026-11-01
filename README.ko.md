[中文](README.md) | [English](README.en.md) | [日本語](README.ja.md) | **한국어**

# oracle-snatch

Oracle Cloud 공식 API에 무료 ARM 인스턴스(VM.Standard.A1.Flex) 신청을 반복해서 보내다가, 재고가 풀리는 순간 API가 승인할 때까지 버티는 작은 Python 루프 스크립트입니다.

## 왜 만들었나

Always Free A1.Flex 무료 한도(합계 4코어 / 24GB)는 지금 기준으로 가장 가성비 좋은 무료 서버라고 진심으로 생각합니다. 문제는 인기 리전에서는 콘솔이 거의 항상 "Out of host capacity(재고 없음)"만 보여준다는 겁니다. 이 재고는 영원히 없는 게 아니라 비정기적으로 조금씩 풀립니다. 다만 아무도 알려주지 않습니다. 콘솔을 일주일 동안 새로고침하는 건 인생 낭비입니다. 물어보는 일은 스크립트에 맡기세요.

2026년 10월, 저는 이 루프의 초기 버전을 도쿄 리전에서 며칠 동안 돌렸습니다. 로그는 온통 "재고 없음"이었고, 간격을 너무 좁혔다가 429 속도 제한도 몇 번 맞았습니다. 그래도 결국 4코어 / 24GB 머신을 잡았습니다. 지금 셀프호스팅하는 서비스 대부분이 그 위에서 돌고 있고, 요금은 오늘까지도 0입니다. 이 저장소는 그 루프를 정리하고 설정을 전부 파일로 빼낸 버전입니다.

## 무엇을 하나

- Oracle 공식 API(Python `oci` SDK)만 사용합니다. 콘솔 자동 조작이나 브라우저 흉내는 하지 않습니다.
- 리전 안의 모든 가용성 도메인(AD)을 순회하며 라운드마다 AD당 한 번씩 시도합니다. 도쿄는 AD가 하나뿐이지만, AD가 많은 리전일수록 당첨 확률이 좋습니다.
- "재고 없음"은 "아직 때가 아님"으로 취급합니다. 잠깐 쉬고 다음 AD, 다음 라운드로 넘어갑니다.
- 429를 만나면 길게 백오프합니다(기본 300초). 속도 제한을 뚫고 밀어붙이면 제한만 더 오래갑니다.
- 할당량 오류(LimitExceeded)가 나오면 멈추고 이유를 분명히 적습니다. 무료 한도를 다 쓴 상태라 만 번을 다시 시도해도 결과는 같습니다.
- 잡고 나면 인스턴스가 RUNNING이 될 때까지 기다리고, 공인 IP를 가져와 `snatched.json`에 기록하고, 원하면 Telegram 알림을 보낸 뒤 종료합니다.
- `--dry-run`은 설정 검증만 합니다. AD 목록과 사용할 이미지를 확인하고 실제로는 아무것도 띄우지 않습니다. 처음에는 꼭 이걸 먼저 실행하세요.

하지 않는 것: 계정 가입, VCN 생성, API 키 발급은 해주지 않고, 없는 재고를 만들어내지도 못합니다. 그저 지치지 않고 대신 물어봐 줄 뿐입니다.

## 설치와 사용법

Oracle Cloud 계정과 API 키가 설정된 OCI 설정 파일(`~/.oci/config`)이 필요합니다. OCI CLI나 SDK를 써본 사람이라면 이미 있는 파일입니다.

서브넷 OCID도 필요합니다. 콘솔에서 Networking -> Virtual Cloud Networks -> 해당 VCN -> Subnets에서 확인합니다. 공인 IP가 필요하면(필요할 겁니다) 퍼블릭 서브넷을 고르세요.

```bash
git clone https://github.com/lucasforster1108-boop/oracle-snatch.git
cd oracle-snatch
pip install -r requirements.txt

cp config.example.env config.env
# config.env 편집: 필수 항목은 SUBNET_ID와 SSH_PUBLIC_KEY_FILE뿐

python3 snatch.py --dry-run
python3 snatch.py
```

## 장기간 돌리기

며칠씩 무인으로 돌리는 전제의 도구라, 이미 항상 켜져 있는 머신 아무 데나 올려두면 됩니다(다른 곳의 무료 VPS, 집 서버, 라즈베리 파이 등). systemd 서비스 예시도 함께 들어 있습니다.

```bash
sudo cp oracle-snatch.service /etc/systemd/system/
sudo systemctl enable --now oracle-snatch
tail -f snatch.log
```

## 직접 겪은 함정

- 무료 A1 한도는 인스턴스별이 아니라 합계입니다. 전부 합쳐 4코어 + 24GB. 4/24 한 대로 써도 되고, 2/12 두 대로 나눠도 됩니다. 넘으면 LimitExceeded가 뜨고, 스크립트는 일부러 멈춥니다. 스크립트 탓을 하지 마세요.
- 계정을 PAYG(종량제)로 업그레이드해도 무료 한도는 사라지지 않습니다. 한도 안의 A1 사용은 계속 무료이고, 재고도 체감상 조금 더 잘 잡힙니다. 제 계정이 PAYG인데 요금은 여전히 0이고, 만약을 위해 예산 알림도 걸어뒀습니다.
- 부트 볼륨도 무료 블록 스토리지 합계 200GB에 포함됩니다. 작은 서버는 50GB면 충분합니다. "어차피 무료니까"라며 200을 다 채우지 말고, 나머지는 두 번째 머신에 남겨두는 게 이득입니다.
- 재고는 비정기적으로 풀리고, 한밤중도 많습니다. 루프의 의미가 여기 있습니다. 새벽 4시에 깨어 있는 건 스크립트면 되지, 당신이 아니어도 됩니다.
- 재시도 간격을 몇 초로 줄이지 마세요. 더 빨리 잡히지 않고, 429만 먹고 한동안 아무것도 못 잡습니다.

## 파일

- `snatch.py`: 로직 전부, 파일 하나
- `config.example.env`: 모든 설정 항목, 주석 포함
- `oracle-snatch.service`: systemd 서비스 예시

## 라이선스

MIT
