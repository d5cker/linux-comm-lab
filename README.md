# 🔍 CI/CD 파이프라인 로그 분석 실전 가이드
 
> `find` · `awk` · `grep` · `jq` · `yq` 를 활용한 DevOps 로그 분석 종합 정리
 
---
 
## 📚 목차
 
1. [파일 시스템 보안 — `find` + `chmod`](#1-파일-시스템-보안--find--chmod)
2. [CSV 로그 분석 — `awk`](#2-csv-로그-분석--awk)
3. [JSON 로그 분석 — `jq`](#3-json-로그-분석--jq)
4. [복합 파이프라인 — `grep` + `awk`](#4-복합-파이프라인--grep--awk)
5. [YAML 설정 분석 — `yq`](#5-yaml-설정-분석--yq)
6. [핵심 개념 & 치트시트](#6-핵심-개념--치트시트)
 
---
 
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
 
