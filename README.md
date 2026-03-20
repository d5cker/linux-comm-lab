# 🔍 DevOps 상황에서 Linux 명령어 문제 가이드

## 📌 개요
### Linux 기반 로그 분석 실전 가이드

Linux 명령어를 활용해 로그파일과 설정파일을 분석하는 데이터 처리 및 검증 실습 프로젝트입니다.

 -  `find`, `grep`, `awk`, 정규표현식을 활용해 파일 탐색과 로그 필터링을 수행

 -  JSON·YAML에서 `jq`와 `yq`를 이용해 형태의 데이터를 가공 및 검증 구성

---
 
## 📚 목차
1. [linux 명령어 개념 설명](#-개요)
2. [CSV 로그 데이터 이해](#2-csv-로그-분석--awk)
3. [로그 데이터 기반 문제 구성](#3-json-로그-분석--jq)
4. [JSON, YAML 설정 분석 및 문제 구성](#4-yaml-설정-분석--yq)
5. [핵심 개념](#5-핵심-개념--치트시트)
 
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

## 📊 데이터 구조

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

상단에서 학습한 Linux 명령어(grep, awk, find)와 로그를 기반으로,<br>
실제 CI/CD 운영 환경에서 발생할 수 있는 상황을 가정한 분석 문제 5개를 구성하였습니다.

---

# 🔴 장애 분석

## 🛑 [Mission] CRITICAL 장애 로그 식별

### 1. 상황

CI/CD 파이프라인 실행 중 발생한 장애 로그가 `/home/ubuntu/cicd_logs/cicd_logs.csv`에 저장되어 있다.
운영팀은 서비스 영향도가 높은 **CRITICAL 장애**를 우선적으로 분석하려고 한다.

---

### 2. 요구 사항 

1. severity가 `CRITICAL`인 로그만 추출
2. 다음 필드를 출력

* pipeline_id
* failure_stage
* failure_type

---

### 💻 모범 답안

```bash
awk -F',' 'NR>1 && $14=="CRITICAL" {print $1, $12, $13}' cicd_logs.csv
```

---

## 🛑 [Mission] CRITICAL 장애 단계별 집계

### 1. 상황

CRITICAL 장애 중 일부는 롤백이 발생한 경우이며, 중복 로그도 포함되어 있다.
운영팀은 **실제 장애 발생 건수**를 기준으로 단계별 통계를 보고자 한다.

---

### 2. 요구 사항 

1. severity가 `CRITICAL`이고 rollback_triggered가 `TRUE`인 로그만 대상
2. 중복된 전체 행은 제거
3. failure_stage 기준으로 발생 건수 집계

---

### 💻 모범 답안

```bash
awk -F',' 'NR>1 && $14=="CRITICAL" && $16=="TRUE" && !seen[$0]++ {stage[$12]++} END {for (s in stage) print s, stage[s]}' cicd_logs.csv
```

---

# 🟡 성능 분석

## ⚡ [Mission] 평균 테스트 시간 분석

### 1. 상황 

전체 파이프라인의 테스트 성능을 파악하기 위해 평균 테스트 시간을 계산한다.

---

### 2. 요구 사항 

* test_duration_sec(10번째 필드)의 평균값 계산

---

### 💻 모범 답안

```bash
awk -F',' 'NR>1 {sum+=$10; cnt++} END {print sum/cnt}' cicd_logs.csv
```

---

# 🟢 품질 분석

## 🧪 [Mission] flaky 테스트 성능 분석

### 1. 상황 

불안정한 테스트(flaky test)는 품질 저하의 주요 원인이다.
flaky 테스트만 따로 분석하여 평균 실행 시간을 확인한다.

---

### 2. 요구 사항 

* is_flaky_test가 `True`인 경우만 필터링
* 해당 테스트들의 평균 실행 시간 계산

---

### 💻 모범 답안

```bash
awk -F',' 'NR>1 && $15=="True" {sum+=$10; cnt++} END {print sum/cnt}' cicd_logs.csv
```

---

# 🔵 특정 조건 필터링

## 🕒 [Mission] 특정 시간대 로그 분석

### 1. 상황 

2026-01-12 새벽 01시대에 장애가 집중적으로 발생했다는 보고가 있다.
해당 시간대 로그를 확인한다.

---

### 2. 요구 사항 

* timestamp가 `2026-01-12T01`인 로그 필터링
* 사용자(author)와 빌드 시간 출력

---

### 💻 모범 답안

```bash
grep "2026-01-12T01" cicd_logs.csv | awk -F',' '{print $6, $9}'
```

---

# ⚫ 파일 보안 점검

## 🔐 [Mission] 위험 권한 파일 탐지

### 1. 상황 

서버 내 일부 로그 파일이 잘못된 권한(777)으로 설정되어 보안 위험이 발생하고 있다.

---

### 2. 요구 사항 

1. `/home/ubuntu/cicd_logs/` 이하에서 권한이 777인 파일 탐색
2. 파일 이름에 `failure` 포함된 파일만 필터링 (find만 사용)
3. 해당 파일들의 권한을 644로 변경

---

### 💻 모범 답안

```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777
```

```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777 -name "*failure*"
```

```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777 -exec chmod 644 {} \;
```

---

# 🔵 추가 심화 문제 

## ☁️ [Mission] AWS 환경 성능 비교

### 1. 상황 

AWS 환경에서 CI/CD 툴별 성능 차이를 분석하려고 한다.

---

### 2. 요구 사항 

* cloud_provider가 AWS인 로그만 필터링
* ci_tool별 평균 build_duration_sec 계산
* 가장 빠른 순으로 정렬

---

### 💻 모범 답안

```bash
grep ',AWS,' cicd_logs.csv | awk -F',' '{sum[$4]+=$9; cnt[$4]++} END{for(i in sum) print i, sum[i]/cnt[i]}' | sort -k2,2n
```




# 4. JSON · YAML 설정 검증 및 문제 구성

이 실습은 DevOps 환경에서 실제로 발생할 수 있는 로그 데이터를 기반으로,
Linux 명령어와 `jq`, `yq`를 활용해 데이터를 필터링하고 가공하는 과정을 다룹니다.


---

# 1. 🔴 CSV 로그 분석 (awk, grep, find)

## 🛑 [Mission] CRITICAL 장애 로그 식별

### 상황 

CI/CD 파이프라인 실행 로그가 `cicd_logs.csv` 파일에 저장되어 있다.
운영팀은 서비스 영향도가 높은 CRITICAL 장애만 우선적으로 확인하려고 한다.

### 요구 사항 

* severity가 `CRITICAL`인 로그만 추출
* 다음 필드를 출력

  * pipeline_id
  * failure_stage
  * failure_type

### 💻 모범 답안

```bash
awk -F',' 'NR>1 && $14=="CRITICAL" {print $1, $12, $13}' cicd_logs.csv
```

---

## 🛑 [Mission] 장애 단계별 집계

### 상황 

CRITICAL 장애 중 일부는 롤백이 발생했으며, 동일한 로그가 중복으로 기록되어 있다.
실제 장애 발생 건수를 기준으로 단계별 통계를 확인하려고 한다.

### 요구 사항 

* severity가 `CRITICAL`이고 rollback_triggered가 `TRUE`인 로그만 대상
* 중복된 전체 행은 제거
* failure_stage 기준으로 발생 건수 집계

### 💻 모범 답안

```bash
awk -F',' 'NR>1 && $14=="CRITICAL" && $16=="TRUE" && !seen[$0]++ {stage[$12]++} END {for (s in stage) print s, stage[s]}' cicd_logs.csv
```

---

## ⚡ [Mission] 평균 테스트 시간 계산

### 상황 

전체 파이프라인의 테스트 성능을 파악하기 위해 평균 테스트 시간을 계산한다.

### 요구 사항 

* test_duration_sec 필드의 평균값 계산

### 💻 모범 답안

```bash
awk -F',' 'NR>1 {sum+=$10; cnt++} END {print sum/cnt}' cicd_logs.csv
```

---

## 🧪 [Mission] flaky 테스트 성능 분석

### 상황 

불안정한 테스트(flaky test)는 품질 저하의 주요 원인이다.
flaky 테스트만 따로 분석하여 실행 시간을 확인한다.

### 요구 사항 
* is_flaky_test가 `True`인 로그만 필터링
* 해당 로그들의 평균 테스트 시간 계산

### 💻 모범 답안

```bash
awk -F',' 'NR>1 && $15=="True" {sum+=$10; cnt++} END {print sum/cnt}' cicd_logs.csv
```

---

## 🕒 [Mission] 특정 시간대 로그 분석

### 상황 

특정 시간대에 장애가 집중 발생했다는 보고가 있다.

### 요구 사항 

* timestamp가 `2026-01-12T01`인 로그만 필터링
* 사용자(author)와 빌드 시간 출력

### 💻 모범 답안

```bash
grep "2026-01-12T01" cicd_logs.csv | awk -F',' '{print $6, $9}'
```

---

## 🔐 [Mission] 로그 파일 권한 점검

### 상황 

로그 디렉터리 내 일부 파일이 777 권한으로 설정되어 보안 문제가 발생하고 있다.

### 요구 사항 

1. `/home/ubuntu/cicd_logs/` 이하에서 권한이 777인 파일 찾기
2. 파일 이름에 `failure`가 포함된 파일만 추가 필터링 (find만 사용)
3. 해당 파일들의 권한을 644로 변경

### 💻 모범 답안

```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777
```

```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777 -name "*failure*"
```

```bash
find /home/ubuntu/cicd_logs/ -type f -perm 777 -exec chmod 644 {} \;
```

---

# 2. 🟠 JSON 로그 분석 (jq)

## 📌 [Mission] 인시던트 로그 리포트 생성

### 상황 

파이프라인 로그가 `pipeline_logs.json` 파일에 저장되어 있으며,
인시던트가 생성된 로그만 별도로 정리하려고 한다.

### 요구 사항 

* incident_created가 true인 로그만 추출
* 다음 필드만 포함한 JSON 배열로 변환

  * run_id
  * os
  * cloud_provider

### 💻 모범 답안

```bash
jq '[.[] | select(.incident_created == true) | {run_id: .run_id, os: .os, cloud_provider: .cloud_provider}]' pipeline_logs.json
```

---

## 📌 [Mission] CRITICAL 로그 필터링

### 요구 사항 

* severity가 "CRITICAL"인 로그만 출력

### 💻 모범 답안

```bash
jq '.[] | select(.severity == "CRITICAL")' pipeline_logs.json
```

---

## 📌 [Mission] 핵심 필드 추출

### 요구 사항 

* CRITICAL 로그만 대상
* pipeline_id, error_code만 출력

### 💻 모범 답안

```bash
jq '.[] | select(.severity == "CRITICAL") | {pipeline_id: .pipeline_id, error_code: .error_code}' pipeline_logs.json
```

---

## 📊 [Mission] 언어별 장애 통계

### 요구 사항 

* language 필드 추출
* 언어별 발생 횟수 집계

### 💻 모범 답안

```bash
jq -r '.[].language' pipeline_logs.json | sort | uniq -c
```

---

# 3. 🟢 YAML 설정 분석 (yq)

## 📌 [Mission] Kubernetes 이미지 확인

### 상황 

배포 전 YAML 설정 파일에서 실제 사용되는 컨테이너 이미지를 확인하려고 한다.

### 요구 사항 

* 첫 번째 컨테이너의 image 값 출력

### 💻 모범 답안

```bash
yq '.spec.template.spec.containers[0].image' app.yaml
```

---

## 📌 [Mission] HPA CPU 기준 수정

### 상황 

HPA 설정에서 CPU 기준 사용률을 30%에서 50%로 변경해야 한다.

### 요구 사항 

* hpa가 포함된 yaml 파일 찾기
* CPU averageUtilization 값을 50으로 변경

### 💻 모범 답안

```bash
find . -name "*hpa*.yaml" -exec yq -i '(.spec.metrics[] | select(.resource.name == "cpu") | .resource.target.averageUtilization) = 50' {} \;
```

---

## 📌 [Mission] 서버 필터링

### 상황 

VM 배포 전에 리소스가 충분한 서버를 선별하려고 한다.

### 요구 사항 (Requirements)

* ram_gb > 10
* status == "running"
* 조건을 만족하는 서버 객체 전체 출력

### 💻 모범 답안

```bash
yq '.servers[] | select(.ram_gb > 10 and .status == "running")' my_servers.yaml
```


---
## 5. 핵심 개념 & 치트시트
 
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
 
