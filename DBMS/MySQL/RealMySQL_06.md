# 06.데이터 압축
- 디스크의데이터 파일이 크면 클수록 백업/복구 시간이 오래 걸리며, 그만큼의 저장 공간이 필요
- 이러한 문제점을 해결하기 위해 데이터 압축 기능을 제공

</br>

## 6.1 페이지 압축
- 버퍼 풀에 데이터 페이지가 한 번 적재되면 InnoDB 스토리지 엔진은 압축이 해제된 상태로만 데이터 페이지를 관리함
    - 압축: MySQL 서버가 디스크에 저장하는 시점에 데이터 페이지가 압축되어 저장
    - 해제: MySQL 서버가 디스크에서 데이터 페이지를 읽어올 때 압축이 해제
- 페이지 압축 작동 방식
    1. 16KB 페이지를 압축
    1. MySQL 서버는 디스크에 압축된 결과 7KB를 기록
        - 이때, MySQL 서버는 압축 데이터 7KB에 9KB의 빈 데이터를 기록
    1. 디스크에 데이터를 기록한 후, 7KB 이후의 공간 9KB에 대해 펀치 홀 생성
    1. 파일 시스템은 7KB만 남기고 나머지 디스크의 9KB 공간은 다시 운영체제로 반납
- 펀치 홀 기능을 사용하지 못하는 경우가 있어, **실제 페이지 압축은 많이 사용되지 않음**
    - 펀치 홀 기능은 OS뿐만 아니라 하드웨어 체제에서도 해당 기능을 지원해야 사용 가능
    - 아직 파일 시스템 관련 명령어가 펀치 홀을 지원하지 못함

</br>

## 6.2 테이블 압축
- 장점
    - OS나 하드웨어에 대한 제약 없이 사용할 수 있어, 일반적으로 활용도가 더 높음
    - 디스크의 데이터 파일 크기를 줄일 수 있음
- 단점
    - 버퍼 풀 공간 활용률이 낮음
    - 쿼리 처리 성능이 낮음
    - 빈번한 데이터 변경 시 압축률이 떨어짐

### 6.2.1 압축 테이블 생성
- 압축을 사용하기 위한 전제 조건
    - 압축 사용하려는 테이블이 별도의 테이블 스페이스를 사용해야 함
    - `innodb_file_per_table` 시스템 변수가 ON으로 설정된 상태에서 테이블이 생성돼야 함
    - 테이블 생성 시 `ROW_FORMAT=COMPRESSED`옵션을 명시해야 함
    - `KEY_BLOCK_SIZE`옵션으로 압축된 페이지의 목표 크기 명시 가능(2n으로만 설정 가능)
- 압축을 적용하는 방법
    1. 16KB의 데이터 페이지를 압축
    
        (1) 압축된 결과가 8KB 이하면 그대로 디스크에 저장
           
        (2) 압축된 결과가 8KB 초과하면 원본 페이지를 스플릿해서 2개의 페이지에 8KB씩 저장
    1. 나뉜 페이지 각각에 대해 '1'번 단계를 반복 실행

### 6.2.2 KEY_BLOCK_SIZE 설정
- 테이블 압축에서 가장 중요한 부분은 압축된 결과를 예측해서 `KEY_BLOCK_SIZE`를 결정하는 것
- `KEY_BLOCK_SIZE` 적용하기 전 4KB 또는 8KB로 테이블 생성해서 샘플 데이터를 저장해보고 적절한지 판단하는 것이 좋음
- 최소한 테이블의 데이터 페이지가 10개 정도는 생성되도록 테스트 데이터를 `INSERT`해보는 것이 좋음

### 6.2.3 압축된 페이지의 버퍼 풀 적재 및 사용
- 압축된 테이블의 데이터 페이지를 버퍼 풀에 적재하면 압축된 상태와 압축이 해제된 상태 2개 버전을 관리함
    - LRU 리스트는 디스크에서 읽은 상태 그대로의 데이터 페이지 목록이 관리됨
    - Unzip_LRU 리스트는 압축을 해제한 상태의 데이터 페이지 목록이 별도로 관리됨
- 버퍼 풀에서 압축 해제된 버전의 데이터 페이지를 적절한 수준으로 유지하기 위해 다음과 같은 어댑티브 알고리즘을 사용함
    - CPU 사용량이 높은 서버에서는 가능하면 압축과 압축 해제를 피하기 위해 Unzip_LRU의 비율을 높여서 유지함
    - Disk IO 사용량이 높은 서버에서는 가능하면 Unzip_LRU 리스트의 비율을 낮춰서 InnoDB 버퍼 풀의 공간을 더 확보하도록 작동함

### 6.2.4 테이블 압축 관련 설정
- 테이블 압축을 사용할 때 연관된 시스템 변수 소개
    - `innodb_cmp_per_index_enabled`
        - 테이블 압축이 사용된 테이블의 모든 인덱스별로 압축 성공 및 압축 실행 횟수를 수집하도록 설정
    - `innodb_compression_level`
        - 압축률을 설정하는 변수이고, 0~9 까지의 값 중에서 선택할 수 있음
        - 값이 작을수록 압축 속도(CPU자원 소모량)는 빨라지지만 저장 공간은 커질 수 있음
        - 값이 클수록 압축 속도(CPU자원 소모량)는 느려지지만 압축률은 높아짐
    - `innodb_compression_failure_threshold_pct`, `innodb_compression_pad_pct_max`        
    - `innodb_log_compressed_pages`
        