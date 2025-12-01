# MoneyMong Crawler (머니몽 크롤러)
> 증권사 리포트 및 금융 데이터 자동 수집 엔진

**MoneyMong Crawler**는 네이버 증권 등 주요 금융 정보 사이트에서 애널리스트 리포트와 금융 데이터를 자동으로 수집하는 데이터 파이프라인입니다. AWS Lambda 기반 서버리스 아키텍처로 구축되어 정기적으로 최신 리포트를 수집하고 PDF Parser로 전달합니다.

## 목차

- [주요 기능](#주요-기능)
- [기술 스택](#기술-스택)
- [시스템 아키텍처](#시스템-아키텍처)
- [설치 및 실행](#설치-및-실행)
- [크롤링 전략](#크롤링-전략)
- [데이터 파이프라인](#데이터-파이프라인)

## 주요 기능

### 1. 증권사 리포트 자동 수집
- **네이버 증권 크롤링**: 주요 증권사 애널리스트 리포트 수집
- **메타데이터 추출**: 제목, 저자, 발행일, 증권사, 종목 정보
- **PDF 다운로드**: 원본 리포트 파일 자동 다운로드 및 저장
- **중복 제거**: 이미 수집된 리포트 필터링

### 2. 금융 데이터 수집
- **종목 정보**: KRX 상장 종목 코드 및 기본 정보
- **시장 데이터**: 주가, 거래량 등 시계열 데이터
- **뉴스 데이터**: 종목 관련 주요 뉴스 수집

### 3. 서버리스 자동화
- **정기 실행**: AWS EventBridge를 통한 스케줄링 (일 1회)
- **확장성**: Lambda 함수 기반 자동 스케일링
- **모니터링**: CloudWatch를 통한 실행 로그 및 알림

## 기술 스택

### Core
- **Python** 3.11+ - 크롤링 로직 구현
- **BeautifulSoup4** - HTML 파싱
- **Selenium** - 동적 페이지 크롤링
- **Requests** - HTTP 요청 처리

### AWS Services
- **Lambda** - 서버리스 함수 실행
- **EventBridge** - 정기 실행 스케줄링
- **S3** - PDF 파일 저장소
- **CloudWatch** - 로그 및 모니터링

### Data Processing
- **Pandas** - 데이터 정제 및 변환
- **PyPDF2** - PDF 메타데이터 검증

## 설치 및 실행

### 사전 요구사항
- Python 3.11+
- AWS CLI 설정 완료
- AWS Lambda 및 S3 접근 권한
- Chrome/Chromium (Selenium용)

### 환경 변수 설정
```bash
cp .env.example .env
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, S3_BUCKET_NAME 등 설정
```

### 로컬 실행
```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python main.py
```

### AWS Lambda 배포
```bash
# Lambda 레이어 생성 (의존성 패키징)
pip install -r requirements.txt -t python/
zip -r lambda_layer.zip python/

# Lambda 함수 코드 패키징
zip -r function.zip app/ main.py

# AWS CLI를 통한 배포
aws lambda update-function-code \
  --function-name moneymong-crawler \
  --zip-file fileb://function.zip
```

### Docker 실행
```bash
docker build -t moneymong-crawler .
docker run --env-file .env moneymong-crawler
```

## 크롤링 전략

### 네이버 증권 리포트 크롤링

#### 1. 리포트 목록 수집
- **URL**: `https://finance.naver.com/research/company_list.naver`
- **페이징**: 10페이지씩 처리 (약 200개 리포트/일)
- **필터링**: 최근 24시간 이내 발행 리포트

#### 2. 상세 페이지 파싱
```python
# 주요 추출 정보
{
    "title": "리포트 제목",
    "author": "애널리스트 이름",
    "published_date": "발행일시",
    "securities_firm": "증권사명",
    "stock_code": "종목코드",
    "stock_name": "종목명",
    "pdf_url": "PDF 다운로드 링크"
}
```

#### 3. PDF 다운로드 및 검증
- **다운로드**: 원본 PDF 파일 저장
- **검증**: 파일 크기, PDF 유효성 확인
- **명명 규칙**: `{stock_code}_{securities_firm}_{YYYYMMDD}.pdf`

### 크롤링 제한 및 예외 처리
- **Rate Limiting**: 요청 간 1초 대기 (서버 부하 방지)
- **Retry Logic**: 실패 시 최대 3회 재시도 (지수 백오프)
- **User-Agent**: 정상적인 브라우저로 위장
- **에러 핸들링**: 네트워크 오류, 파싱 오류 별도 처리

## 데이터 파이프라인

### 전체 흐름
```
1. 크롤링 → 2. 메타데이터 저장 → 3. S3 업로드 → 4. PDF Parser 트리거
```

### 1. 크롤링 단계
- 네이버 증권에서 신규 리포트 목록 수집
- 각 리포트의 상세 정보 및 PDF 다운로드

### 2. 메타데이터 저장
- PostgreSQL `documents` 테이블에 리포트 정보 저장
- 수집 이력 기록 (`crawl_history` 테이블)

### 3. S3 업로드
- PDF 파일을 S3 버킷에 업로드
- 버킷 경로: `s3://moneymong-reports/{YYYY}/{MM}/{DD}/{filename}.pdf`

### 4. PDF Parser 트리거
- RunPod API 호출을 통한 트리거
