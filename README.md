# 🔍 DevOps 상황에서 Linux 명령어 실전 가이드

## 📌 개요
### Linux 기반 로그 분석 실전 가이드

Linux 명령어를 활용해 로그파일과 설정파일을 분석하는 데이터 처리 및 검증 실습 프로젝트입니다.

- 1 `find`, `grep`, `awk`, 정규표현식을 활용해 파일 탐색과 로그 필터링을 수행

- 2 JSON·YAML에서 `jq`와 `yq`를 이용해 형태의 데이터를 가공하고 검증하는 흐름으로 구성

---
 
## 📚 목차
1. [linux 명령어 개념 설명](#-개요)
2. [CSV 로그 데이터 이해](#2-csv-로그-분석--awk)
3. [로그 데이터 기반 문제 구성](#3-json-로그-분석--jq)
4. [JSON, YAML 설정 분석 및 문제 구성](#5-yaml-설정-분석--yq)
5. [핵심 개념](#6-핵심-개념--치트시트)
 
---

# 1. Linux 명령어 개요 (find, chmod, grep, awk)

## 🔎 find

파일 시스템에서 조건에 맞는 파일을 탐색할 때 사용

```bash
find /path -type f -name "*.log"
```

* 파일 검색
* 조건 기반 필터링
* 일괄 작업 (`-exec`)

---

## 🔐 chmod

파일 권한을 변경할 때 사용

```bash
chmod 644 file.log
```

* 777 → 모든 사용자 권한 (위험)
* 644 → 로그 파일 권장 권한

---

## 🔍 grep

파일 내용에서 특정 문자열을 검색

```bash
grep "ERROR" app.log
```

* 로그 필터링
* 특정 패턴 추출
* 정규표현식 기반 검색

---

## 📊 awk

텍스트 데이터를 필드 단위로 분석 및 가공

```bash
awk -F',' '{print $1, $3}' file.csv
```

* CSV 분석
* 조건 필터링
* 통계 및 집계 처리

---

# 2. CSV 로그 데이터 이해

## 📌 데이터셋 선정 이유 (Kaggle)

이번 실습에서는 Kaggle의 **CI/CD Pipeline Failure Logs Dataset**을 기반으로 문제를 구성했습니다.

선정 이유는 다음과 같습니다.

* 실제 CI/CD 환경을 가정한 로그 구조 → DevOps 상황 반영
* 단순 텍스트가 아닌 **운영 로그 기반 데이터 처리 흐름 경험 가능**
* 실제 서비스 장애 로그 분석 상황을 재현하기 위해 선택

---

## 📊 데이터 구조 (컬럼 설명)

| 순서 | 필드명 | 설명 |
|------|--------|------|
| 1 | pipeline_id | 파이프라인 ID |
| 2 | run_id | 실행 ID |
| 3 | timestamp | 실행 시간 |
| 4 | ci_tool | CI/CD 도구 (Jenkins 등) |
| 5 | repository | 저장소 이름 |
| 6 | author | 실행 사용자 |
| 7 | language | 사용 언어 |
| 8 | cloud_provider | 실행 환경 (AWS 등) |
| 9 | build_duration_sec | 빌드 시간 (초) |
| 10 | test_duration_sec | 테스트 시간 (초) |
| 11 | deploy_duration_sec | 배포 시간 (초) |
| 12 | failure_stage | 실패 단계 (build/test/deploy) |
| 13 | failure_type | 실패 유형 |
| 14 | severity | 장애 심각도 |
| 15 | is_flaky_test | flaky 테스트 여부 |
| 16 | rollback_triggered | 롤백 여부 |

---

# 3. 로그 데이터 기반 문제 구성

로그 분석에서 다루는 문제는 다음과 같이 나뉩니다.

---

## 🔴 장애 분석

### Q1. CRITICAL 장애 로그 추출

```bash
awk -F',' 'NR>1 && $14=="CRITICAL" {print $1, $12, $13}' cicd_logs.csv
```

---

### Q2. 장애 발생 단계별 집계

```bash
awk -F',' 'NR>1 && $14=="CRITICAL" {stage[$12]++} END {for (s in stage) print s, stage[s]}' cicd_logs.csv
```

---

## 🟡 성능 분석

### Q3. 평균 테스트 시간 계산

```bash
awk -F',' 'NR>1 {sum+=$10; cnt++} END {print sum/cnt}' cicd_logs.csv
```

---

## 🟢 품질 분석

### Q4. flaky 테스트 평균 시간

```bash
awk -F',' 'NR>1 && $15=="True" {sum+=$10; cnt++} END {print sum/cnt}' cicd_logs.csv
```

---

## 🔵 특정 조건 필터링

### Q5. 특정 시간대 로그 분석

```bash
grep "2026-01-12T01" cicd_logs.csv | awk -F',' '{print $6, $9}'
```

---

## ⚫ 파일 보안 점검

### Q6. 권한이 777인 로그 파일 찾기

```bash
find /logs -type f -perm 777
```

---

# 4. JSON · YAML 설정 검증 및 문제 구성

로그 분석 이후, 설정 파일을 통해 문제를 확인하고 수정하는 단계입니다.

---

## 📦 JSON (jq)

---

### Q1. CRITICAL 로그 필터링

```bash
jq '.[] | select(.severity=="CRITICAL")' logs.json
```

---

### Q2. 특정 필드만 추출

```bash
jq '.[] | {pipeline_id, error_code}' logs.json
```

---

### Q3. 롤백 발생 로그 필터링

```bash
jq '.[] | select(.rollback_triggered==true)' logs.json
```

---

### Q4. 특정 문자열 포함 여부 확인

```bash
jq '.[] | select(.error_message | contains("Security"))' logs.json
```

---

### Q5. 언어별 장애 횟수 집계

```bash
jq -r '.[].language' logs.json | sort | uniq -c
```

---

## 📦 YAML (yq)

---

### Q1. Deployment 이미지 조회

```bash
yq '.spec.template.spec.containers[0].image' deploy.yaml
```

---

### Q2. replicas 값 확인

```bash
yq '.spec.replicas' deploy.yaml
```

---

### Q3. HPA CPU 기준 변경 (30 → 50)

```bash
yq -i '(.spec.metrics[] | select(.resource.name=="cpu") | .resource.target.averageUtilization)=50' hpa.yaml
```

---

### Q4. env 값 수정

```bash
yq -i '(.spec.template.spec.containers[0].env[] | select(.name=="SPRING_PROFILES_ACTIVE") | .value)="prod"' deploy.yaml
```

---

### Q5. namespace 변경

```bash
yq -i '.metadata.namespace="prod"' deploy.yaml
```

---

### Q6. YAML 파일 일괄 수정

```bash
find . -name "*.yaml" -exec yq -i '.metadata.namespace="prod"' {} \;
```

---

# ✅ 한줄 정리

👉
**로그(CSV)는 grep/awk로 분석하고, 설정(JSON/YAML)은 jq/yq로 검증하고 수정한다**

---

원하시면 다음 단계로
👉 **이걸 “포트폴리오용 + 면접용 README” 수준으로 더 고급스럽게 다듬어 드릴 수 있습니다**
👉 또는 **문제만 따로 분리한 시험지 형태**도 만들어드릴게요






ㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡㅡ


## 1. 파일 시스템 보안 — `find` + `chmod`
 
### 개념
 
| 옵션 | 설명 |
|------|------|
| `-type f` | 일반 파일만 탐색 (디렉터리 제외) |
| `-perm 777` | 권한이 정확히 `777`인 파일 |
| `-name "패턴"` | 파일 이름 패턴 매칭 (`*` 와일드카드) |
| `-exec cmd {} \;` | 검색된 파일마다 명령 실행 |
 
> ⚠️ **보안 원칙**: 권한 `777`(rwxrwxrwx)은 운영 환경에서 **사용 금지**.  
> 로그 파일 권장 권한: `644` (소유자 읽기·쓰기 / 그룹·기타 읽기 전용)
 
---
 
### Q1. 권한 777인 모든 파일 찾기
 
```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777
```
 
### Q2. 파일명에 `failure` 포함된 파일만 필터링 (`grep` 없이)
 
```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777 -name "*failure*"
```
 
### Q3. 권한 777 → 644 일괄 변경
 
```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777 -exec chmod 644 {} \;
```
 
- `{}` : find가 찾은 파일 경로로 치환되는 플레이스홀더
- `\;` : `-exec` 블록 종료 (셸이 `;`를 해석하지 않도록 이스케이프)
- 💡 `{} +` 로 바꾸면 파일을 묶어 한 번에 실행하여 더 빠름
 
---
 
## 2. CSV 로그 분석 — `awk`
 
### 개념
 
```
awk -F'구분자' 'condition { action }' 파일
```
 
| 변수/패턴 | 설명 |
|-----------|------|
| `-F','` | 필드 구분자를 쉼표로 지정 (CSV) |
| `NR` | 현재 행 번호 |
| `NR>1` | 헤더(1번째 줄) 건너뜀 |
| `$N` | N번째 필드 |
| `$0` | 전체 행 |
| `!seen[$0]++` | 중복 행 제거 |
| `END { }` | 전체 처리 후 실행 (집계 출력) |
 
### 주요 필드 구조 (cicd_logs.csv)
 
| 필드 | 이름 |
|------|------|
| `$1` | pipeline_id |
| `$13` | test_duration_sec |
| `$14` | failure_stage |
| `$15` | failure_type |
| `$19` | severity |
| `$23` | is_flaky_test |
| `$24` | rollback_triggered |
 
---
 
### Q1. CRITICAL 장애 로그 추출
 
```bash
awk -F',' 'NR>1 && $19=="CRITICAL" {print $1, $14, $15}' /home/ubuntu/cicd_logs/cicd_logs.csv
```
 
### Q2. CRITICAL + rollback=TRUE, 중복 제거 후 failure_stage별 집계
 
```bash
awk -F',' 'NR>1 && $19=="CRITICAL" && $24=="TRUE" && !seen[$0]++ {stage[$14]++} END {for (s in stage) print s, stage[s]}' /home/ubuntu/cicd_logs/cicd_logs.csv
```
 
**`!seen[$0]++` 동작 원리**
 
```
처음 등장 → seen["행"] = 0 → !0 = true  → 처리 ✅
두 번째~  → seen["행"] = 1 → !1 = false → 건너뜀 (중복 제거) ❌
```
 
### Q3. flaky 테스트의 평균 수행 시간 계산
 
> `is_flaky_test`($23)가 `True`인 로그의 `test_duration_sec`($13) 평균
 
```bash
awk -F',' 'NR>1 && $23=="True" {sum+=$13; cnt++} END {if(cnt>0) print "평균:", sum/cnt, "sec"}' /home/ubuntu/cicd_logs/cicd_logs.csv
```
 
---
 
## 3. JSON 로그 분석 — `jq`
 
### 개념
 
```
jq '필터' 파일.json
```
 
| 표현식 | 설명 |
|--------|------|
| `.[]` | 배열의 모든 요소를 펼침 |
| `.field` | 특정 필드 접근 |
| `select(조건)` | 조건에 맞는 요소만 통과 |
| `{key: .field}` | 새로운 객체 생성 |
| `\| length` | 배열 길이 |
| `contains("str")` | 문자열 포함 여부 |
| `-r` | 따옴표 없이 순수 텍스트 출력 |
| `[ ... ]` | 결과를 배열로 감싸기 |
 
---
 
### Q1. severity = CRITICAL 로그 필터링
 
```bash
cat pipeline_logs.json | jq '.[] | select(.severity == "CRITICAL")'
```
 
### Q2. CRITICAL 로그에서 특정 필드만 추출
 
```bash
cat pipeline_logs.json | jq '.[] | select(.severity == "CRITICAL") | {pipeline_id: .pipeline_id, error_code: .error_code}'
```
 
### Q3. 언어별 장애 횟수 집계
 
```bash
cat pipeline_logs.json | jq -r '.[].language' | sort | uniq -c
```
 
- `jq -r` : 따옴표 없이 텍스트 추출
- `sort` : 같은 언어끼리 인접 정렬
- `uniq -c` : 중복 합치고 앞에 횟수 표시
 
### Q4. incident_created=true 로그를 새 JSON 배열로 변환
 
```bash
jq '[.[] | select(.incident_created == true) | {run_id: .run_id, os: .os, cloud_provider: .cloud_provider}]' pipeline_logs.json
```
 
### 기타 유용한 `jq` 명령어
 
```bash
# 배열 전체 크기 확인
cat pipeline_logs.json | jq '. | length'
 
# 에러 메시지에 특정 단어 포함 여부
cat pipeline_logs.json | jq '.[] | select(.error_message | contains("Security"))'
```
 
---
 
## 4. 복합 파이프라인 — `grep` + `awk`
 
### 파이프라인 설계 원칙
 
```
[1차 필터: grep] → [2차 처리: awk] → [3차 정렬: sort]
```
 
> `grep`으로 행을 먼저 줄이면 `awk` 처리 속도가 향상됨
 
---
 
### Q1. 야간 배포 비용 감사 (2026-01-12 새벽 01~03시, Python)
 
출력 형식: `[실행시간] 사용자ID - $비용` + 마지막 줄에 총합
 
```bash
grep -E "2026-01-12T0[1-3]:" deploy.csv | awk -F',' '$9=="Python" {print "["$3"] "$8" - $"$20; s+=$20} END{printf "Total Cost: $%.2f\n", s}'
```
 
### Q2. AWS 환경에서 CI/CD 툴별 평균 빌드 시간 (빠른 순)
 
```bash
grep ',AWS,' emp.csv | awk -F',' '{sum[$4]+=$12; cnt[$4]++} END{for(i in sum) print i, sum[i]/cnt[i]}' | sort -k2,2n
```
 
- `sort -k2,2n` : 2번째 필드(평균 빌드 시간) 기준 숫자 오름차순 → 첫 줄 = 가장 빠른 툴
 
---
 
## 5. YAML 설정 분석 — `yq`
 
### 개념
 
```bash
yq '.path.to.field' file.yaml           # 값 읽기
yq -i '.path.to.field = 값' file.yaml   # 파일 직접 수정 (in-place)
```
 
---
 
### Q1. 컨테이너 이미지 값 조회
 
```yaml
# app.yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/web:1.0
          ports:
            - containerPort: 8080
```
 
```bash
yq '.spec.template.spec.containers[0].image' app.yaml
# 출력: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/web:1.0
```
 
| 선택지 | 출력 | 설명 |
|--------|------|------|
| `.metadata.name` | `web` | Deployment 이름 |
| `.spec.replicas` | `2` | 레플리카 수 |
| ✅ `.spec.template.spec.containers[0].image` | 이미지 경로 | **정답** |
| `.spec.template.spec.containers[0].ports[0].containerPort` | `8080` | 포트 번호 |
 
---
 
### Q2. HPA CPU 임계값 30% → 50% 변경
 
```bash
find . -name "*hpa*.yaml" -exec yq -i '(.spec.metrics[] | select(.resource.name == "cpu") | .resource.target.averageUtilization) = 50' {} \;
```
 
- `find ... -exec ... {} \;` : 찾은 모든 HPA yaml 파일에 일괄 적용
- `select(.resource.name == "cpu")` : metrics 배열 중 cpu 항목만 선택
- `= 50` : averageUtilization 값을 50으로 덮어씀
 
---
 
## 6. 핵심 개념 & 치트시트
 
### 파일 권한
 
```
777 = rwxrwxrwx  ⚠️ 보안 위험 (모두에게 전체 권한)
644 = rw-r--r--  ✅ 로그 파일 권장
755 = rwxr-xr-x  ✅ 실행 파일/디렉터리 권장
```
 
### `jq` vs `awk` 선택 기준
 
| 상황 | 추천 |
|------|------|
| JSON 형식 데이터 | `jq` |
| CSV/TSV 형식 데이터 | `awk` |
| 중첩 구조 탐색 | `jq` |
| 대용량 단순 집계 | `awk` (더 빠름) |
| 새 JSON 객체 생성 | `jq` |
 
### 빠른 참조 (Quick Reference)
 
```bash
# find
find [경로] -type f -perm 777
find [경로] -type f -perm 777 -name "*패턴*"
find [경로] -type f -perm 777 -exec chmod 644 {} \;
 
# awk — 집계
awk -F',' 'NR>1 && [조건] {arr[$KEY]++} END{for(k in arr) print k, arr[k]}' file.csv
 
# awk — 중복 제거 + 집계
awk -F',' 'NR>1 && [조건] && !seen[$0]++ {arr[$KEY]++} END{...}' file.csv
 
# jq — 필터링 + 필드 선택 + 배열 출력
jq '[.[] | select(.field=="val") | {key: .field}]' file.json
 
# jq → 언어별 집계
jq -r '.[].field' file.json | sort | uniq -c | sort -rn
 
# yq — 조건부 필드 수정
yq -i '(.array[] | select(.name=="target") | .value) = 새값' file.yaml
 
# find + yq — 여러 yaml 파일 일괄 수정
find . -name "*.yaml" -exec yq -i '[수정 표현식]' {} \;
```
 
