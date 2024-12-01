## Prometheus

## : https://github.com/prometheus/prometheus

3장부터 교재를 읽다가, 옵저버빌리티의 대표적인 오픈소스인 프로메테우스를 뜯어보며 공부하는 것이 좋을것이라 생각해서,

우선 코드부터 뜯어보았습니다. ( 오픈소스 최고! )

---

### 프로메테우스를 직접 사용해보기 전에 self-study를 진행해보자

프메 : 프로메테우스는 Cloud Native Computing Foundation(CNCF)의 주요 프로젝트로서, 시스템 및 서비스 모니터링을 위한 오픈소스 도구


---

특징, 개념 생략. 구글링을 하면 공식문서에 너무 잘 나와있다.

그보다, 프로메테우스 프로젝트의 코드 및 파일구성부터 살펴보자.

<img width="1368" alt="image" src="https://github.com/user-attachments/assets/dc520025-c760-429c-a9fc-90d6e303425e">

우선, 많은 부분이 Go를 사용해서 작성되었음을 알 수 있다. 모니터링 시각화 화면은 TypeScript로 구현한것으로 보인다.

### 프로메테우스의 핵심 디렉토리들은 무엇이 있을까?

## 1. /promql - 최우선 핵심 디렉토리

> 프로메테우스의 핵심 기능인 PromQL(Prometheus Query Language) 구현
메트릭 데이터를 쿼리하고 분석하는 모든 로직이 포함
시계열 데이터 처리를 위한 연산자와 함수들의 구현체
최근 커밋에서도 보이듯 지속적인 성능 최적화와 기능 개선이 이루어지는 중요 컴포넌트이다..!


정말 코드가 많지만, 프로메테우스의 다양한 기능이 집약 구현되어있는 함수파일을 들여다보자

### 히스토그램 관련 핵심 함수들


~~~go
// === histogram_sum(Vector parser.ValueTypeVector) (Vector, Annotations) ===
func funcHistogramSum(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper) (Vector, annotations.Annotations) {
	inVec := vals[0].(Vector)

	for _, sample := range inVec {
		// Skip non-histogram samples.
		if sample.H == nil {
			continue
		}
		if !enh.enableDelayedNameRemoval {
			sample.Metric = sample.Metric.DropMetricName()
		}
		enh.Out = append(enh.Out, Sample{
			Metric:   sample.Metric,
			F:        sample.H.Sum,
			DropName: true,
		})
	}
	return enh.Out, nil
}

// === histogram_avg(Vector parser.ValueTypeVector) (Vector, Annotations) ===
func funcHistogramAvg(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper) (Vector, annotations.Annotations) {
	inVec := vals[0].(Vector)

	for _, sample := range inVec {
		// Skip non-histogram samples.
		if sample.H == nil {
			continue
		}
		if !enh.enableDelayedNameRemoval {
			sample.Metric = sample.Metric.DropMetricName()
		}
		enh.Out = append(enh.Out, Sample{
			Metric:   sample.Metric,
			F:        sample.H.Sum / sample.H.Count,
			DropName: true,
		})
	}
	return enh.Out, nil
}

// === histogram_stddev(Vector parser.ValueTypeVector) (Vector, Annotations)  ===
func funcHistogramStdDev(vals []parser.Value, args parser.Expressions, enh *EvalNodeHelper) (Vector, annotations.Annotations) {
	inVec := vals[0].(Vector)

	for _, sample := range inVec {
		// Skip non-histogram samples.
		if sample.H == nil {
			continue
		}
		mean := sample.H.Sum / sample.H.Count
		var variance, cVariance float64
		it := sample.H.AllBucketIterator()
		for it.Next() {
			bucket := it.At()
			if bucket.Count == 0 {
				continue
			}
			var val float64
			if bucket.Lower <= 0 && 0 <= bucket.Upper {
				val = 0
			} else {
				val = math.Sqrt(bucket.Upper * bucket.Lower)
				if bucket.Upper < 0 {
					val = -val
				}
			}
			delta := val - mean
			variance, cVariance = kahanSumInc(bucket.Count*delta*delta, variance, cVariance)
		}
		variance += cVariance
		variance /= sample.H.Count
		if !enh.enableDelayedNameRemoval {
			sample.Metric = sample.Metric.DropMetricName()
		}
		enh.Out = append(enh.Out, Sample{
			Metric:   sample.Metric,
			F:        math.Sqrt(variance),
			DropName: true,
		})
	}
	return enh.Out, nil
}
~~~

## 코드 해설?? (feat. claude )

# 프로메테우스 히스토그램 함수 분석 문서

## 1. 히스토그램 합계 함수 (HistogramSum)

### 함수 시그니처
```go
func funcHistogramSum(
    vals []parser.Value, 
    args parser.Expressions, 
    enh *EvalNodeHelper
) (Vector, annotations.Annotations)
```

### 기능 개요
* **주요 목적**: 히스토그램의 전체 누적값 계산
* **사용 사례**: 전체 시스템 부하량, 총 요청 수 등 집계 데이터 산출

### 처리 프로세스
1. 입력 벡터 순회
2. 히스토그램 데이터 검증
3. 메트릭 이름 처리
4. 합계값 계산 및 반환

## 2. 히스토그램 평균 함수 (HistogramAvg)

### 함수 시그니처
```go
func funcHistogramAvg(
    vals []parser.Value, 
    args parser.Expressions, 
    enh *EvalNodeHelper
) (Vector, annotations.Annotations)
```

### 기능 개요
* **주요 목적**: 히스토그램 데이터의 평균값 산출
* **활용 분야**: 시스템 성능의 중심 경향성 파악

### 계산 프로세스
1. 입력 데이터 순회 및 검증
2. 평균값 계산: `Sum / Count`
3. 결과값 벡터화

## 3. 히스토그램 표준편차 함수 (HistogramStdDev)

### 함수 시그니처
```go
func funcHistogramStdDev(
    vals []parser.Value, 
    args parser.Expressions, 
    enh *EvalNodeHelper
) (Vector, annotations.Annotations)
```

### 기능 개요
* **주요 목적**: 데이터 분산도 측정
* **활용 분야**: 시스템 성능의 안정성 및 변동성 분석

### 핵심 알고리즘
1. **평균값 계산**
   ```go
   mean := sample.H.Sum / sample.H.Count
   ```

2. **버킷 값 처리**
   ```go
   if bucket.Lower <= 0 && 0 <= bucket.Upper {
       val = 0
   } else {
       val = math.Sqrt(bucket.Upper * bucket.Lower)
       if bucket.Upper < 0 {
           val = -val
       }
   }
   ```

3. **Kahan 합산 최적화**
   - 부동소수점 연산의 정밀도 보존
   - 누적 오차 최소화

## 공통 구현 패턴

### 1. 데이터 검증
```go
if sample.H == nil {
    continue
}
```
- 히스토그램 데이터 타입 확인
- 무효 데이터 필터링

### 2. 메트릭 처리
```go
if !enh.enableDelayedNameRemoval {
    sample.Metric = sample.Metric.DropMetricName()
}
```
- 조건부 메트릭명 제거
- 네이밍 일관성 유지

### 3. 결과 포맷팅
```go
Sample{
    Metric:   sample.Metric,
    F:        value,
    DropName: true
}
```
- 표준화된 출력 형식
- 메타데이터 보존

## 최적화 고려사항

### 1. 성능 최적화
- 불필요한 메모리 할당 최소화
- 효율적인 반복문 처리

### 2. 정확도 최적화
- Kahan 합산 알고리즘 활용
- 부동소수점 연산 정밀도 관리

### 3. 메모리 관리
- 벡터 재사용
- 효율적인 메모리 할당

---
*Note: 이 문서는 프로메테우스 v2.x 버전을 기준으로 작성되었습니다.*

## 2. /tsdb - 두 번째 핵심 디렉토리


> Time Series Database 구현체가 포함된 디렉토리
프로메테우스의 데이터 저장소 엔진 역할
시계열 데이터의 효율적인 저장과 검색을 담당 (내부 로직이 궁금하니 찾아볼 것. 구현을 어떻게 한건지..?)
히스토그램 처리, 샘플링 등 성능에 직접적인 영향을 미치는 핵심 기능 포함되어 있음. 


## 3, /storage - 세 번째 핵심 디렉토리


> 원격 저장소 연동을 포함한 스토리지 계층 구현
Remote Write/Read 기능 구현 (최근 DNS resolver 개선 등 활발한 개발)
데이터 보존 정책과 장기 저장소 연동 관련 핵심 기능 포함
확장성 있는 모니터링 시스템 구축을 위한 필수 컴포넌트

이 세 디렉토리는 프로메테우스의 핵심 가치인 '메트릭 수집', '저장', '쿼리' 기능을 각각 담당하기에 Go를 빠르게 배워보고

모니터링 오픈소스는 어떻게 만드는지??? 에 대한 궁금증..을 계속해서 파헤쳐보자..!
