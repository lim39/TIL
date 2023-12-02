# 10. 실행 계획
- 대부분의 DBMS는 많은 데이터를 안전하게 저장 및 관리하고 사용자가 원하는 데이터를 빠르게 조회할 수 있게 해주는 것이 주목적
- 이러한 목적을 달성하려면 옵티마이저가 사용자의 쿼리를 최적으로 처리될 수 있게 하는 쿼리의 실행 계획을 수립할 수 있어야 함 
- 실행 계획 명령어: `EXPLAIN`

 </br>

## 10.1 통계 정보
- 5.7 버전까지 테이블과 인덱스에 대한 개괄적인 정보를 가지고 실행 계획을 수립해서 실행 계획의 정확도가 떨어짐
    - 테이블 컬럼의 값들이 실제 어떻게 분포돼 있는지에 대한 정보가 없기 때문
- 8.0 버전부터 인덱스되지 않은 컬럼들에 대해서도 데이터 분포도를 수집해서 저장하는 히스토그램 정보가 도입됨

### 10.1.1 테이블 및 인덱스 통계 정보
- **비용 기반 최적화에서 통계 정보가 가장 중요!**
- 통계 정보가 최신화 되어 있어야 정확한 최적화를 할 수 있음
    - 예시) 1억 건의 레코드가 저장된 테이블의 통계 정보가 갱신되지 않아서 레코드가 10건 미만인 것처럼 되어 있다면, 옵티마이저는 실제 쿼리를 실행할 때 인덱스 레인지 스캔이 아니라 테이블 풀 스캔으로 실행해 버릴 수도 있음
- MySQL 서버에서는 쿼리의 실행 계획을 수립할 때 실제 테이블에서 데이터를 일부 분석해서 통계 정보를 보완해서 사용함

#### 10.1.1.1 MySQL 서버의 통계 정보
- 버전별 통계 정보 히스토리
    - ~ 5.5 버전
        - 각 테이블의 통계 정보가 메모리에만 관리되기 때문에 서버 재시작 시 지금까지 수집된 통계 정보가 날아감
        - SHOW INDEX 명령으로만 테이블의 인덱스 컬럼의 분포도를 볼 수 있었음
        - 통계 정보를 수집할 때 샘플로 분석할 테이블 블록 수를 `innodb_stats_sample_pages` 시스템 설정 변수가 제공
    - 5.6 버전 ~
        - 각 테이블의 통계 정보를 mysql DB의 `innodb_index_stats` 테이블과 `innodb_table_stats` 테이블로 관리할 수 있게 개선됨
        - 통계 정보를 테이블로 관리함으로써 서버가 재시작돼도 기존의 통계 정보를 영구적으로 관리할 수 있게 개선됨
        - 테이블 단위로 통계 정보를 영구적으로 보관할지에 대한 여부를 결정할 수 있음
        - `innodb_stats_sample_pages` 시스템 설정 변수 옵션 없어지고, 2개의 시스템 변수로 분리됨
            - `innodb_stats_treansient_sample_pages`: 기본값은 8이고, 이는 통계 정보 수집이 실행될 때 8개 페이지만 임의로 샘플링해서 분석하고 그 결과를 통계 정보로 활용함을 의미함
            - `innodb_stats_persistent_sample_pages`: 기본값은 20이며, `ANALYZE TABLE` 명령이 실행되면 임의로 20개 페이지만 샘플링해서 분석하고 그 결과를 영구적인 통계 정보 테이블에 저장하고 활용함을 의미함
                - 더 정확한 통계 정보를 수집하고 싶다면 해당 시스템 변수 값을 높이면 됨
                - 높일수록 통계 정보 수집 시간이 길어지는 것은 주의
        
- 통계 정보 테이블 주요 컬럼별 설명
    - `innodb_index_stats.stat_name='n_diff_pfx%'`: 인덱스가 가진 유니크한 값의 개수
    - `innodb_index_stats.stat_name='n_leaf_pages'`: 인덱스의 리프 노드 페이지 개수
    - `innodb_index_stats.stat_name='size'`: 인덱스 트리의 전체 페이지 개수
    - `innodb_index_stats.n_rows`: 테이블의 전체 레코드 건수
    - `innodb_index_stats.clustered_index_size`: 프라이머리 키의 크기(InnoDB 페이지 개수)
    - `innodb_index_stats.sum_of_other_index-sizes`: 프라이머이 키를 제외한 인덱스의 크기(InnoDB 페이지 개수)
        - 테이블의 `STATS_AUTO_RECALC` 옵션에 따라 0으로 보일 수도 있는데, 이 경우 `ANALYZE TABLE [테이블명]` 을 실행하면 통계값이 저장됨

- 통계 정보가 자동으로 갱신되는 경우(이벤트 발생 시)
    - 테이블이 새로 오픈되는 경우 (오픈?)
    - 테이블의 레코드가 대량으로 변경되는 경우(테이블의 전체 레코드 중에서 1/16 정도의 `UPDATE`/`INSERT`/`DELETE` 가 실행되는 경우)
    - `ANALYZE TABLE` 명령이 실행되는 경우
    - `SHOW TABLE STATUS` 명령이나 `SHOW INDEX FROM` 명령이 실행되는 경우
    - `InnoDB` 모니터가 활성화되는 경우
    - `innodb_stats_on_metadata` 시스템 설정이 ON인 상태에서 `SHOW TABLE STATUS` 명령이 실행되는 경우

- `STATS_PERSISTENT` 옵션
    - 옵션 값별 통계 정보 관리 방식 달라짐
        - =0 : 5.5 이전의 방식대로 관리  
        - =1 : 5.6 이후의 방식대로 관리  
        - =DEFAULT : 해당 옵션을 설정하지 않은 것과 동일, 테이블의 통계를 영구적으로 관리할지에 대한 여부는 `innodb_stats_persistent` 시스템 변수의 값으로 결정함
            - 이 시스템 변수는 기본적으로 ON(1)로 설정돼 있음
    - 예시
        ```sql
        CREATE TABLE tab_test (fd1 INT, fd2 VARCHAR(20), PRIMARY KEY(fd1))
        ENGINE=InnoDB
        STATS_PERSISTENT={DEFAULT | 0 | 1};

        --추후 옵션 값 변경 가능!
        ALTER TABLE tab_test STATS_PERSISTENT=={DEFAULT | 0 | 1};
        ```
- `STATS_AUTO_RECALC` 옵션
    - 옵션 값별 통계 정보 자동 수집 여부가 달라짐
        - =1 : 테이블의 통계 정보를 5.5 이전의 방식대로 자동 수집함
        - =0 : 테이블의 통계 정보는 ANALYZE TABLE 명령을 수행할 때만 수집됨
        - =DEFAULT : 테이블을 생성할 때 별도로 `STATS_AUTO_RECALC` 옵션을 설정하지 않은 것과 동일, 테이블의 통계정보 수집을 `innodb_stats_auto_recalc` 시스템 설정 변수의 값으로 결정함
            - 이 시스템 변수는 기본적으로 ON이므로 영구적인 통계 정보를 이용하고자 한다면 OFF로 변경해야 함

- 

### 10.1.2 히스토그램
- 8.0 버전부터 도입
- 컬럼의 데이터 분포도를 참조할 수 있는 기능

#### 10.1.2.1 히스토그램 정보 수집 및 삭제
- 히스토그램 정보 수집 및 조회
    - 히스토그램 정보는 컬럼 단위로 관리됨
    - 자동으로 수집되지 않고 `ANALYZE TABLE ... UPDATE HISTOGRAM` 명령을 실행해 수동으로 수집 및 관리됨
    - 수집된 히스토그램 정보는 시스템 딕셔너리에 함께 저장
    - MySQL 서버가 시작될 때 딕셔너리의 히스토그램 정보를 `information_schema` 데이터베이스의 `column_statistics` 테이블을 `SELECT`해서 참조할 수있음
- 히스토그램 삭제
    - 삭제 작업은 테이블의 데이터를 참조하는 것이 아니라 딕셔너리의 내용만 삭제함
    - 따라서 대상 테이블 다른 작업들에 영향주지 않고 즉시 처리됨
    - 히스토그램이 삭제되면 실행 계획이 달라질 수 있으니 주의해야 함
    - 히스토그래믈 삭제하지 않고 옵티마이저가 히스토그램을 사용하지 않게 하려면 `optimizer_switch` 시스템 변수의 값을 변경하면 됨
        ```sql
        -- 서버 전체 히스토그램 사용하지 않게 설정
        SET GLOBAL optimizer_switch='condition_fanout_filter=off';

        -- 현재 커넥션에서 실행되는 쿼리만 히스토그램을 사용하지 않게 설정
        SET SESSION optimizer_switch='condition_fanout_filter=off';

        -- 현재 쿼리만 히스토그램 사용하지 않게 설정 (힌트에 명시)
        SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=off') */ 
            * 
        FROM ...
        ;
        ```
- 히스토그램 수집, 조회 및 삭제 예제 
    ```sql
    -- 히스토그램 수집
    ANALYZE TABLE employees.employees -- schema_name.table_name
    UPDATE HISTOGRAM ON gender, hire_date;

    --히스토그램 조회
    SELECT *
    FROM COLUMN_STATISTICS
    WHERE SCHEMA_NAME='employees'
        AND TABLE_NAME='employees';

    -- 히스토그램 삭제
    ANALYZE TABLE employees.employees
    DROP HISTOGRAM ON gender, hire_date;
    ```

- 히스토그램은 버킷(Bucket) 단위로 구분되어 레코드 건수나 컬럼값의 범위가 관리되고, 8.0 버전에선 2종류의 히스토그램 타입이 지원됨
    - 싱글톤 히스토그램(Singleton)
        - 컬럽값 개별로 레코드 건수를 관리하는 히스토그램으로, Value-Based 히스토그램 또는 도수 분포라고도 불림
        - 컬럼이 가지는 값별로 버킷이 할당됨
        - 각 버킷이 컬럼의 값과 발생 빈도의 비율의 2개의 값을 가짐
        - 주로 코드 값과 같이 유니크한 값의 개수가 상대적으로 적은 경우 사용됨 (=히스토그램의 버킷 수보다 적은 경우)
    - 높이 균형 히스토그램(Equi-Height)
        - 컬럼값의 범위를 균등한 개수로 구분해서 관리하는 히스토그램으로, Height-Balanced 히스토그램이라고도 불림
        - 개수가 균등한 컬럼값의 범위별로 하나의 버킷이 할당됨
        - 각 버킷이 범위 시작 값과 마지막 값, 그리고 발생 빈도율과 각 버킷에 포함된 유니크한 값의 개수 등 4개의 값을 가짐

- 히스토그램 테이블의 주요 컬럼
    - `sampling-rate`
        - 히스토그램 정보를 수집하기 위해 스캔한 페이지의 비율
        - 샘플링 비율이 0.35라면 전체 데이터 페이지의 35%를 스캔해서 이 정보를 수집했다는 것을 의미함
        - 샘플링 비율이 높아질수록 정확도는 높아지나, 스캔 부하가 높아져 시스템의 자원을 많이 소모함
        - `histogram_generation_max_mem_size` 시스템 변수에 설정된 메모리 크기에 맞게 적절히 샘플링 함 (20 MB로 초기화 되어 있음)
    - `histogram-type`
        - 히스토그램의 종류를 저장함
    - `number-of-buckets-specified`
        - 히스토그램을 생성할 때 설정했던 버킷의 개수를 저장함
        - DEFUALT 값은 100이라, 생성할 때 기본으로 100개의 버킷이 사용됨
        - 버킷은 최대 1024개 설정 가능하나, 100개면 충분함

#### 10.1.2.2 히스토그램의 용도
- 히스토그램이 도입되기 전에도 테이블과 인덱스에 대한 통계 정보가 존재했지만, 옵티마이저는 항상 균등하게 분포돼 있을 것으로 예측함 > 부정확함
- 히스토그램은 특정 컬럼이 가지는 모든 값에 대한 분포도 정보를 가지지는 않지만 각 범위(버킷)별로 레코드의 건수와 유니크한 값의 개수 정보를 가지기 때문에 훨씬 정확한 예측을 할 수 있음
- 히스토그램 정보 수집 유무에 따라 조인 성능을 10배 정도 차이 날 수 있음
    - 조인 시 건수가 적은 테이블을 드라이빙 테이블로 잡는 것이 성능적으로 매우 유리함 (NL조인 특성)
    - 히스토그램 정보를 모르면 옵티마이저는 컬럼들의 데이터 분포를 전혀 알지 못하고 드라이빙 테이블 선정을 하게 됨

#### 10.1.2.3 히스토그램과 인덱스
- 히스토그램은 주로 인덱스되지 않은 컬럼에 대한 데이터 분포도를 참조하는 용도로 사용됨
- 실행 계획을 수립할 때 사용 가능한 인덱스들로부터 조건절에 일치하는 레코드 건수를 대략 파악하고 최종적으로 비용이 가장 적은 실행 계획을 선택함
    - 인덱스는 부족한 통계 정보를 수집하기 위해 사용됨
    - 옵티마이저는 실제 인덱스의 B-Tree를 샘플링해서 살펴봄 (인덱스 다이브(Index Dive)라고도 함)
- **인덱스된 컬럼을 검색 조건으로 사용하는 경우 그 컬럼의 히스토그램은 사용하지 않고, 실제 인덱스 다이브를 통해 직접 수집한 정보를 활용함**
    - 인덱스 다이브 작업은 어느 정도의 비용이 필요하여
    - 떄로는 실행 계획 수립만으로도 상당한 인덱스 다이브를 실행하고 비용도 그만큼 커짐
    - 저자는 추후 버전에서 실제 인덱스 다이브를 실행하기보다 히스토그램을 활용하는 최적화 기능이 도입될 것으로 예상함

### 10.1.3 코스트 모델(Cost Model)
- 전체 쿼리의 비용을 계산하는 데 필요한 단위 작업들의 비용을 **코스트 모델(Cost Model)**이라고 함
    - MySQL 서버가 쿼리를 처리할 때 필요한 작업들
        - 디스크로부터 데이터 페이지 읽기
        - 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
        - 인덱스 키 비교
        - 레코드 평가
        - 메모리 임시 테이블 작업
        - 디스크 임시 테이블 작업
    - MySQL 서버는 사용자의 쿼리에 대해 이러한 다양한 작업이 얼마나 필요한지 예측하고 전체 작업 비용을 계산한 결과로 최적 실행 계획을 찾음
- MySQL 버전별 코스트 모델 계산 방식 변화
    - 5.7 이전 버전까지 이런 작업들의 비용을 서버 소스 코드에 상수화해서 사용했음
        - 이 작업들의 비용은 서버가 사용하는 하드웨어에 따라 달라질 수 있으나 고려되지 않음
        - 고정된 비용을 일률적으로 적용하는 것은 최적의 실행계획 수립에 있어서 방해 요소였음
    - 5.7 버전부터 각 단위 작업의 비용을 DBMS 관리자가 조정할 수 있게 개선됨
        - 5.7 버전에선 인덱스되지 않은 컬럼의 데이터 분포(히스토그램)나 메모리에 상주 중인 페이지의 비율 등 비용 계산과 연관된 부분의 정보가 부족했음
        - 8.0 버전부턴 히스토그램과 각 인덱스별 메모리에 적재된 페이지의 비율이 관리되고 옵티마이저의 실행 계획 수립에 사용되기 시작함
    - 8.0 버전 코스트 모델은 mysql DB에 있는 2개 테이블에 저장돼 있는 설정값 사용
        - `server_cost`: 인덱스를 찾고 레코들를 비교하고 임시 테이블 처리에 대한 비용 관리
            - `server_cost.cost_name`: 코스트 모델의 각 단위 작업
            - `server_cost.default_value`: 각 단위 작업의 비용(기본값이며, 이 값은 MySQL 서버 소스 코드에 설정된 값)
            - `server_cost.cost_value`: DBMS 관리자가 설정한 값(이 값이 NULL이면 MySQL 서버는 `default_value` 컬럼의 비용 사용
            - `server_cost.last_updated`: 단위 작업의 비용이 변경된 시점
            - `server_cost.comment`: 비용에 대한 추가 설명
        - `engine_cost`: 레코드를 가진 데이터 페이지를 가져오는 데 필요한 비용 관리
            - `engine_cost.engine_name`: 비용이 적용된 스토리지 엔진
                - 스토리지 엔진별로 각 단위 작업의 비용을 설정할 수 있음
                - 기본값은 'default'로 특정 스토리지 엔진의 비용이 설정되지 않았다면 해당 스토리지 엔진의 비용으로 이 값을 적용한다는 의미
            - `engine_cost.device_type`: 디스크 타입
                - 8.0 버전에선 아직 이 컬럼의 값을 활용하지 않음
                - 따라서 '0' 만 설정 가능
    - 8.0 버전 코스트 모델에서 지원하는 단위 작업
        |  | cost_name | default_value | description |
        | --- | --- | --- | --- |
        | engine_cost | `io_block_read_cost` | 1.00 | 디스크 데이터 페이지 읽기 |
        | engine_cost | `memory_block_read_cost` | 0.25 | 메모리 데이터 페이지 읽기 |
        | server_cost | `disk_temptable_create_cost` | 20.00 | 디스크 임시 테이블 생성 |
        | server_cost | `disk_temptable_row_cost` | 0.50 | 디스크 임시 테이블의 레코드 읽기 |
        | server_cost | `key_compare_cost` | 0.05 | 인덱스 키 비교 |
        | server_cost | `memory_temptable_create_cost` | 1.00 | 메모리 임시 테이블 생성 |
        | server_cost | `memory_temptable_row_cost` | 1.00 | 디스크 데이터 페이지 읽기 |
        | server_cost | `row_evaluate_cost` | 0.10 | 레코드 비교 |
    
- 코스트 모델에서 중요한 것은 각 단위 작업에 설정되는 비용 값이 커지면 어떤 실행 계획들이 저비용인지 고비용인지를 파악하는 것임
- 참고) 대표적으로 각 단위 작업의 비용이 변경되면 예상할 수 있는 결과
    - `key_compare_cost` 비용을 높이면 MySQL 서버 옵티마이저가 가능하면 정렬을 수행하지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
    - `row_evaluate_cost` 비용을 높이면 풀 스캔을 실행하는 쿼리들의 비용이 높아지고, MySQL 옵티마이저는 가능하면 인덱스 레인지 스캔을 사용하는 실행 계획을 선택할 가능성이 높아짐
    - `disk_temptable_create_cost`와 `disk_temptable_row_cost` 비용을 높이면 MySQL 옵티마이저는 디스크에 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
    - `memory_temptable_create_cost`와 `memory_temptable_row_cost` 비용을 높이면 MySQL 옵티마이저는 메모리 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
    - `io_block_read_cost` 비용이 높아지면 MySQL 옵티마이저는 가능하면 InnoDB 버퍼 풀에 데이터 페이지가 많이 적재돼 있는 인덱스를 사용하는 실행 계획을 선택할 가능성이 높아짐
    - `memory_block_read_cost` 비용이 높아지면 MySQL 서버는 InnoDB 버퍼 풀에 적재된 데이터 페이지가 상대적으로 적다고 하더라도 그 인덱스를 사용할 가능성이 높아짐

- 참고) 실행 계획별 계산된 비용(Cost) 조회
    ```sql
    mysql> EXPLAIN FORMAT=TREE
            SELECT *
            FROM employees WHERE first_name='Matt' \G
    ********************************* 1. row *********************************
    EXPLAIN:    -> Index lookup on employees using ix_firstname (first_name='Matt')
                    (cost=256.10 rows=233)
    ```

## 10.2 실행 계획 확인
- 실행 계획은 `DESC` 또는 `EXPLAIN` 명령으로 확인할 수 있음
- 8.0 버전부터 `EXPLAIN`에 사용할 수 있는 옵션인 '실행 계획의 출력 포맷'과 '실제 쿼리의 실행 결과 확인' 이 추가됨

### 10.2.1 실행 계획 출력 포맷
- 다른 DBMS와는 다르게 DEFAULT가 테이블 포맷인게 특징
- 테이블 포맷 표시
```sql
mysql> EXPLAIN
        SELECT *
        FROM employees e
        INNER JOIN salaries s ON s.emp_no=e.emp_no
        WHERE first_name='ABC';
+---+------------+------+-----------+-----+----------------------+--------------+---------+--------------------+-----+----------+
|id |select_type |table |partitions |type |possible_keys         | key          | key_len | ref                |rows | filtered |
+---+------------+------+-----------+-----+----------------------+--------------+---------+--------------------+-----+----------+
| 1 | SIMPLE     | e    | NULL      | ref | PRIMARY,ix_firstname | ix_firstname | 58      | const              | 1   | 100.00   |
| 1 | SIMPLE     | s    | NULL      | ref | PRIMARY              | PRIMARY      | 4       | employees.e.emp_no | 10  | 100.00   |
+---+------------+------+-----------+-----+----------------------+--------------+---------+--------------------+-----+----------+
```
- 트리 포맷 표시
```sql
mysql> EXPLAIN FORMAT=TREE
        SELECT *
        FROM employees e
        INNER JOIN salaries s ON s.emp_no=e.emp_no
        WHERE first_name='ABC';
********************************* 1. row *********************************
    EXPLAIN:    -> Nested loop inner join (cost=2.40 rows=10)
            -> Index lookup on e using ix_firstname (first_name='ABC') (cost=0.35 rows=1)
            -> Index lookup on s using PRIMARY (emp_no=e.emp_no) (cost=2.05 rows=10)
```
- JSON 포맷 표시
```sql
mysql> EXPLAIN FORMAT=JSON
        SELECT *
        FROM employees e
        INNER JOIN salaries s ON s.emp_no=e.emp_no
        WHERE first_name='ABC';
********************************* 1. row *********************************
EXPLAIN:  {
    "query_block":  {
        "select_id":1,
        "cost_info":  {
            "query_cost": "2.40"
        },
    "nested_loop":  [
    {   
        "table_name": "e",
        "access_type": "ref",
        "possible_keys":  [
            "PRIMARY",
            "ix_firstname"
        ],
        "key": "ix_firstname",
        "used_key_parts":  [
            "first_name"
        ],
    ...
```

### 10.2.2 쿼리의 실행 시간 확인
- 8.0.18 버전부터는 쿼리의 실행 계획과 단계별 소요된 시간 정보를 확인할 수 있는 `EXPLAIN ANALYZE` 기능이 추가됨
    - 해당 기능은 항상 TREE 포맷으로 보여줘서 FORMAT 옵션을 사용할 수 없음
- `EXPLAIN ANALYZE` 명령은 실제 쿼리를 실행하고 사용된 실행 계획과 소요된 시간을 보여줌
    - 실행 시간이 오래 걸리는 쿼리라면 `EXPLAIN` 명령으로 먼저 실행 계획만 확인해서 어느 정도 튜닝한 후 `EXPLAIN ANALYZE` 명령을 실행하는 것이 좋음
- 실행 계획을 실제 실행 순서에 따라 읽는 규칙
    - 들여쓰기가 같은 레벨에서는 상단에서 위치한 라인이 먼저 실행
    - 들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행
- `EXPLAIN ANALYZE` 예시
    ```sql
    mysql>  EXPLAIN ANALYZE
            SELECT e.emp_no, avg(s.salary)
            FROM employees e
              INNER JOIN salaries s ON s.emp_no=e.emp_no
                        AND s.salaries>50000
                        AND s.from_date<='1990-01-01'
                        AND s.to_date>'1990-01-01'
            WHERE e.first_name='Matt'
            GROUP BY e.hire_date \G
    A) ->Table scan on <temporary>    (actual time=0.001..0.004 rows=48 loops=1)
    B)      -> Aggregate using temporary table  (actual time=3.799..3.808 rows=48 loops=1)
    C)          -> Nested loop inner join   (cost=685.24 rows=135)
                                (actual time=0.367..3.602 rows=48 loops=1)
    D)              ->Index lookup on e using ix_firstname (first_name='Matt') (cost=215.08 rows=233)
                                (actual time=0.348..1.046 rows=233 loops=1)
    E)              -> Filter:  ((s.salary>50000) and (s.from_date <= DATE'1990-01-01') and (s.to_date > DATE'1990-01-01')) (cost=0.98 rows=1) 
                                (actual time=0.009..0.011 rows=0 loops=233)
    F)                  ->Index lookup on s using PRIMARY (emp_no=e.emp_no) (cost=0.98 rows=10)
                                (actual time=0.007..0.009 rows=10 loops=233)
    ```
    - 실행 순서: D -> F -> E -> C -> B -> A
        1. D) employees 테이블의 ix_firstname 인덱스를 통해 first_name='Matt' 조건에 일치하는 레코드를 찾고
        2. F) salaries 테이블의 PRIMARY 키를 통해 emp_no가 (1)번 결과의 emp_no와 동일한 레코드를 찾아서
        3. E) ((s.salary>50000) and (s.from_date <= DATE'1990-01-01') and (s.to_date > DATE'1990-01-01')) 조건에 일치하는 건만 가져와
        4. C) (1)번과 (3)번의 결과를 조인해서
        5. B) 임시 테이블에 결과를 저장하면서 GROUP BY 집계를 실행하고
        6. A) 임시 테이블의 결과를 읽어서 결과를 반환한다.
    - 실행 계획 결과에 표시된 필드
        - 실제 소요된 시간(actual time)
            - loops=1 이면 평균값이 표기됨
        - 처리한 레코드 건수(rows)
            - loops=1 이면 평균값이 표기됨
        - 반복 횟수(loops)

## 10.3 실행 계획 분석
- 옵션 없이 `EXPLAIN` 명령을 실행하면 쿼리 문장의 특성에 따라 표 형태로 된 1줄 이상의 결과가 표시됨
- 표의 각 라인은 쿼리 문장에서 사용된 테이블의 개수만큼 출력되고, 실행 순서는 위에서 아래로 순서대로 표시됨
    - UNION이나 상관 서브쿼리와 같은 경우 순서대로 표시되지 않을 수 있음
- 해당 목차에선 실행 계획에 표시되는 각 컬럼이 어떤 것을 의미하는 지 설명함

### 10.3.1 id 컬럼
- id 컬럼은 **단위 SELECT 쿼리별로 부여되는 식별자 값**임
- SELECT 단위로 id 구분
    - 하나의 SELECT 문장 내 여러 테이블을 조인하는 쿼리문의 경우
        - 조인되는 테이블의 개수만큼 실행 계획 레코드가 출력되지만 같은 id 값이 부여됨
    - 쿼리 문장이 3개의 단위 SELECT 쿼리로 구성돼 있는 경우
        - 실행 계획의 각 레코드가 각기 다른 id 값을 지님
        - 여기서 주의할 점은 실행 계획의 id 컬럼이 테이블의 접근 순서를 의미하지는 않음
- 테이블(표) 형태 실행 계획에서 접근 순서가 혼란스럽다면, `EXPLAIN FORMAT=TREE` 명령으로 확인 권장

### 10.3.2 select_type 컬럼
- select_type 컬럼은 **각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 컬럼**임
- 해당 컬럼에 표시할 수 있는 값은 다음과 같다
    - SIMPLE
    - PRIMARY
    - UNION
    - DEPENDENT UNION
    - UNION RESULT
    - SUBQUERY
    - DEPENDENT SUBQUERY
    - DERIVED
    - DEPENDENT DERIVED
    - UNCACHEABLE SUBQUERY
    - UNCACHEABLE UNION
    - MATERIALIZED

#### 10.3.2.1 SIMPLE
- UNION이나 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우 **'SIMPLE'**로 표시됨
- 실행 계획에서 SIMPLE인 단위 SELECT 쿼리는 하나만 존재함
    - 일반적으로 제일 바깥 SELECT 쿼리의 `select_type`이 'SIMPLE'로 표시됨

#### 10.3.2.2 PRIMARY
- UNION이나 서브쿼리를 가지는 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 SELECT 쿼리는 `select_type`이 **'PRIMARY'**로 표시됨
- 'SIMPLE'과 마찬가지로 하나만 존재

##### 10.3.2.3 UNION
- UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리들은 `select_type`이 **'UNION'**로 표시됨
- 'UNION'의 첫 번째 단위 SELECT은 `select_type`이 'DERIVED'로 표시됨
    - UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블(DERIVED)

#### 10.3.2.4 DEPENDENT UNION
- 'DEPENDENT UNION' 또한 'UNION' `select_type`과 같이 UNION이나 UNION ALL로 집합을 결합하는 쿼리에서 표시됨
- 여기서 'DEPENDENT'는 외부 쿼리에 의해 영향을 받는 것을 의미함
- 예제 쿼리문) 메인 쿼리 조건(WHERE)절 내 서브쿼리로 UNION이 들어간 경우
    ```sql
    SELECT *
    FROM employees e1 WHERE e1.emp_no IN (
        SELECT e2.emp_no FROM employees e2 WHERE e2.first_name='Matt'
        UNION
        SELECT e3.emp_no FROM employees e3 WHERE e3.last_name='Matt'
    );    
    ``` 
    - 쿼리 실행하는 순서에 의해 IN 내부의 서브쿼리를 먼저 처리하지 않고 외부의 employees 테이블을 먼저 읽은 다음 서브쿼리를 실행하는데,
    이때 employees 테이블의 컬럼값이 서브쿼리에 영향을 줌
    - 이렇게 내부 쿼리가 외부의 값을 참조해서 처리될 때 `select_type`에 'DEPENDENT' 키워드가 표시됨

#### 10.3.2.5 UNION RESULT
- 'UNION RESULT'는 UNION 결과를 담아두는 테이블을 의미함
- 8.0 버전부터 UNION ALL의 경우 임시 테이블을 사용하지 않도록 기능이 개선됨
    - 이전 버전에선 UNION ALL, UNION 쿼리는 모두 UNION 결과를 임시 테이블로 생성하게 돼 있었음
- 'UNION RESULT'는 실제 쿼리에서 단위 SELECT 쿼리가 아니기 때문에 별도의 id 값은 부여되지 않음
    - id 값은 NULL로 표기

#### 10.3.2.6 SUBQUERY
- **FROM 절 이외에서 사용되는 서브쿼리만을 의미함**
- MySQL 서버의 실행 계획에서 `FROM` 절에 사용된 서브쿼리는 `select_type`이 'DERIVED'로 표시됨
- *그 밖의 위치에서 사용된 서브쿼리는 전부 'SUBQUERY'라고 표시됨*
- 이 책이나 MySQL 메뉴얼에서 사용되는 용어인 "파생 테이블"은 'DERIVED'와 같은 의미로 이해하면 됨

#### 10.3.2.7 DEPEDENT SUBQUERY
- 서브쿼리가 바깥쪽 SELECT 쿼리에서 정의된 컬럼을 사용하는 경우, `select_type`에 **'DEPENDENT SUBQUERY'**라고 표기 됨
- 'DEPENDENT UNION'과 같이 'DEPENDENT SUBQUERY' 또한 외부 쿼리가 먼저 수행된 후 내부 쿼리(서브쿼리)가 실행되어야 하므로 일반 서브쿼리보다 처리 속도가 느릴 때가 많음
- 예제 쿼리문
    ```sql
    SELECT e.first_name,
            (SELECT COUNT(*)
            FROM dept_emp de, dept_manager dm
            WHERE dm.dept_no=de.dept_no AND de.emp_no=e.emp_no) AS cnt
    FROm employees e
    WHERE e.first_name='Matt';
    ```
#### 10.3.2.8 DERIVED
- **'DERIVED'는 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것**을 의미함
    - 5.5 버전까지 서브쿼리가 FROM 절에 사용된 경우 항상 `select_type`이 'DERIVED'인 실행 계획을 만들었음
    - 5.6 버전부터 옵티마이저 옵션(optimizer_switch)에 따라 FROM 절의 서브쿼리를 외부 쿼리와 통합하는 형태의 최적화가 수행되기도 함
- 'DERIVED'인 경우에 생성되는 임시 테이블을 파생 테이블이라고도 함
    - 5.5 버전까지 파생 테이블에는 인덱스가 전혀 없어서 다른 테이블과 조인할 때 성능상 불리함
    - 5.6 버전부터 옵티마이저 옵션에 따라 쿼리의 특성에 맞게 임시 테이블에도 인덱스를 추가할 수 있음

#### 10.3.2.9 DEPENDENT DERIVED
- 8.0 이전 버전에서는 FROM 절의 서브쿼리는 외부 컬럼을 사용할 수 없었음
- 8.0 버전부터 `LATERAL JOIN` 기능이 추가되면서 FROM 절의 서브쿼리에서도 외부 컬럼을 참조할 수 있게 됨
- 해당 테이블이 `LATERAL JOIN`로 사용된 경우 `select_type`이 **'DEPENDENT DERIVED'**임
    - `LATERAL JOIN`의 경우 `LATERAL` 키워드를 사용해야 하며, 키워드가 없는 서브쿼리에서 외부 컬럼을 참조하면 오류가 발생함
    - 예제 쿼리문) `LATERAL JOIN`
        ```sql
        SELECT *
        FROM employees e
        LEFT JOIN LATERAL
            (SELECT *
            FROM salaries s
            WHERE s.emp_no=e.emp_no
            ORDER BY s.from_date DESC LIMIT 2) AS s2 
        ON s2.emp_no=e.emp_no;
        ```

#### 10.3.2.10 UNCACHEABLE SUBQUERY
- 조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고, 이전의 실행 결과를 그대로 사용할 수 있게 서브쿼리 결과를 내부 캐시 공간에 담아둠
    - 파생 테이블(DERIVED)과는 무관한 기능
- 서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능하면 'UNCACHEABLE SUBQUERY'로 표시됨
    - 사용자 변수가 서브쿼리에 사용된 경우
    - NOT-DETERMINISTIC 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
    - UUID()나 RAND()와 같이 결과값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우
- `select_type`이 'SUBQUERY'인 경우와 'UNCACHEABLE SUBQUERY'은 이 캐시를 사용할 수 있냐 없냐의 차이
- SUBQUERY, DEPENDENT SUBQUERY 의 캐시를 사용하는 방법
    - SUBQUERY는 바깥쪽의 영향을 받지 않으므로 처음 한 번만 실행해서 그 결과를 캐시하고 필요할 때 캐시된 결과를 이용함
    - DEPENDENT SUBQUERY는 의존하는 바깥쪽 쿼리의 컬럼의 값 단위로 캐시해두고 사용함

#### 10.3.2.11 UNCACHEABLE UNION
- UNION과 UNCACHEABLE 두 개의 키워드의 속성이 혼합된 `select_type`임

#### 10.3.2.12 MATERIALIZED
- 주로 FROM 절이나 IN 형태의 쿼리에 사용된 서브쿼리의 최적화를 위해 사용됨
- 'DERIVED'와 비슷하게 쿼리의 내용을 임시 테이블로 생성하는 것을 의미한다는 정도만 알면 됨
- 예제 쿼리문)
    ```sql
    SELECT *
    FROM employees e
    WHERE e.emp_no IN (SELECT emp_no FROM salaries WHERE salary BETWEEN 100 AND 1000);
    ```
    - 5.6 버전까지 employees 테이블을 읽어서 레코드마다 salaries 테이블을 읽는 서브쿼리가 실행되는 형태로 처리됨
    - 5.7 버전부터 서브쿼리의 내용을 임시 테이블로 구체화(MATERIALIZED)한 후, 임시 테이블과 employees 테이블을 조인하는 형태로 최적화되어 처리됨


### 10.3.3 table 컬럼
- MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시됨
    - 테이블에 별칭이 부여된 경우 별칭으로 표시됨
- **table 컬럼에 '<>'로 둘러싸인 것은 임시 테이블을 의미하고, 안에 항상 표시되는 숫자는 단위 SELECT 쿼리의 id 값을 지칭함**
- `select_type`이 'MATERIALIZED'인 실행 계획에서는 '<subquery N>'과 같은 값이 table 컬럼에 표시됨
    - 이는 서브쿼리의 결과를 구체화해서 임시 테이블로 만들었다는 의미
    - 실제로는 <derived N>과 같은 방법으로 해석하면 됨

### 10.3.4 partitions 컬럼
- partitions 컬럼에선 접근한 파티션을 알 수 있음
- 용어 참고) 파티션 프루닝
    - 파티션이 여러 개인 테이블에서 불필요한 파티션을 빼고 쿼리를 수행하기 위해 접근해야 할 것으로 판단되는 테이블만 골라내는 과정

### 10.3.5 type 컬럼
- type 컬럼의 값은 각 테이블의 접근 방법(Access type)으로 해석하면 됨
- type 컬럼에 표시될 수 있는 값: 테이블 접근 방식 (일반적으로 처리 성능이 빠른 순서)
    - system
    - const
    - eq_ref
    - ref
    - fulltext
    - ref_or_null
    - unique_subquery
    - index_subquery
    - range
    - index_merge
    - index
    - ALL
- ALL을 제외하고 모두 인덱스를 사용하는 접근 방식임

#### 10.3.5.1 system
- 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 **system**이라고 함
- 이 접근 방식은 InnoDB에서 나타나지 않고, MyISAM이나 MEMORY 엔진 테이블에서만 사용됨
- 8.0 부터 InnoDB 엔진만 사용되니 이하 생략

#### 10.3.5.2 const
- 테이블의 레코드 건수와 관계없이 쿼리가 PK나 Unique Key 컬럼을 이용하는 WHERE 조건절을 가지고 있으며, 반드시 1건을 반환하는 쿼리의 처리 방식을 **const**라고 함
    - 다른 DBMS에선 **유니크 인덱스 스캔**이라고도 표현함
- 다중 컬럼으로 구성된 PK나 Unique Key 중 인덱스의 일부 컬럼만 조건으로 사용될 땐 const 접근 방식은 사용할 수 없음
    - 해당 조건이 1건일거라는 보장이 없음
    - PK의 일부만 조건으로 사용할 땐 type컬럼에 reg로 표시됨
- 참고)옵티마이저가 쿼리를 최적화하는 단계에서 쿼리를 먼저 실행해서 통째로 상수화하기 때문에 const(상수)라고 표기됨

#### 10.3.5.3 eq_ref
- **eq_ref** 접근 방식은 여러 테이블이 조인되는 쿼리의 실행 계획에서만 표시됨
- 조인 조건 컬럼들이 PK나 Unique Key 컬럼인 경우 두 번째 이후에 읽는 테이블의 type 컬럼에 eq_ref라고 표시됨
    - 조인에서 두 번째 이후에 읽는 테이블에서 반드시 1건만 존재한다는 보장이 있어야 함

#### 10.3.5.4 ref
- **ref** 접근 방식은 eq_ref와 달리 조인의 순서와 관계없고, PK나 UK 제약 조건도 없이 사용됨
- 인덱스의 종류와 관계없이 동등 조건으로 검색할 때는 ref 접근 방식이 사용됨
- 반환되는 레코드가 반드시 1건이라는 보장이 없어서, const와 eq_ref보단 느리지만, 동등 조건으로만 비교되므로 매우 빠른 레코드 조회 방법 중 하나

#### 10.3.5.5 fulltext
- **fulltext** 접근 방식은 MySQL 서버의 전문 검색 인덱스를 사용해 레코드를 읽는 접근 방식을 의미함
- 전문 검색 인덱스는 통계 정보가 관리되지 않으며, 전혀 다른 SQL 문법을 사용해야 함
- 전문 검색 조건은 우선숙위가 상당히 높고, 반드시 해당 테이블에 전문 검색용 인덱스가 준비돼 있어야만 함
    - 전문 검색용 인덱스가 없다면 오류가 발생하고 중지됨
    - MATCH (...) AGAINST(...) 구문으로 검색
- 예제 쿼리문) 전문 검색 조건
    ```sql
    SELECT *
    FROM employee_name
    WHERE emp_no=10001
        AND emp_no BETWEEN 10001 AND 10005
        AND MATCH(first_name, last_name) AGAINST ('Facello' IN BOOLEAN MODE);

#### 10.3.5.6 ref_or_null
- **ref_or_null**은 ref 접근 방식과 같은데, NULL 비교가 추가된 형태
- 실제 업무에서 많이 활용되지만, 만약 사용된다면 나쁘지 않은 접근 방법

#### 10.3.5.7 unique_subquery
- **unique_subquery**은 WHERE 조건절에서 사용될 수 있는 IN 형태의 쿼리를 위한 접근 방식임
- 의미 그래도 서브쿼리에서 중복되지 않는 유니크한 값만 반환할 때 이 접근 방법을 사용함

#### 10.3.5.8 index_subquery
- IN 서브쿼리가 중복된 값을 인덱스를 이용해서 제거할 수 있을 때 **index_subquery** 접근 방식이 사용됨
- index_subquery와 unique_subquery 차이
    - unique_subquery: IN 형태의 조건에서 subquery의 반환 값에는 중복이 없으므로 별도의 중복 제거 작업이 필요하지 않음
    - index_subquery: IN 형태의 조건에서 subquery의 반환 값에는 중복된 값이 있을 수 있지만 인덱스를 이용해 중복된 값을 제거할 수 있음

#### 10.3.5.9 range
- **range**는 인덱스 레인지 스캔 형태의 접근 방식
- 인덱스를 하나의 값이 아니라 범위 조건으로 검색하는 경우를 의미하는데, 주로 '<, >, IS NULL, BETWEEN, IN, LIKE' 등의 연산자를 이용해 인덱스를 검색할 때 사용됨
- 애플리케이션의 쿼리가 가장 많이 사용하는 접근 방식

#### 10.3.5.10 index_merge
- **index_merge** 접근 방식은 2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어낸 후, 그 결과를 병합해서 처리하는 방식
- **index_merge** 접근 방식 특징
    - 여러 인덱스를 읽어야 하므로 일반적으로 range 접근 방식보다 효율성이 떨어짐
    - 전문 검색 인덱스를 사용하는 쿼리에서는 index_merge가 적용되지 않음
    - index_merge 접근 방식으로 처리된 결과는 항상 2개 이상의 집합이 되기 때문에 그 두 집합의 교집합이나 합집합 또는 중복 제거와 같은 부가적인 작업이 더 필요함
- 예제 쿼리문)
    ```sql
    SELECT * 
    FROM employees
    WHERE emp_no BETWEEN 10001 AND 11000
        OR first_name='Smith';
    ```

#### 10.3.5.11 index
- **index** 접근 방식은 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔을 의미함
    - 인덱스 풀 스캔은 테이블 풀 스캔보다는 빠르고, 소량의 레코드 건수 조회 시 효과적이지만
    - 조회할 레코드 건수가 많아지면 상당히 느린 처리를 수행함
- **index** 접근 방식은 다음 조건 중 (1,2) 조건을 충족하거나 (1,3) 조건을 충족하는 쿼리에서 사용되는 읽기 방식
    1. range나 const ref 같은 접근 방식으로 인덱스를 사용하지 못하는 경우
    2. 인덱스에 포함된 컬럼만으로 처리할 수 있는 쿼리인 경우(즉, 데이터 파일을 읽지 않아도 되는 경우)
    3. 인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우(즉, 별도의 정렬 작업을 피할 수 있는 경우)

#### 10.3.5.12 ALL
- **ALL**은 테이블 풀 스캔을 의미함
- InnoDB도 다른 DBMS와 같이 테이블/인덱스 풀 스캔은 대량의 디스크 I/O를 유발하는 작업을 위해 한꺼번에 많은 페이지를 읽어 들이는 기능을 제공함
    - 이러한 기능을 리드 어헤드(Read Ahead)라고 함
- 빠른 응답을 사용자에게 보내야하는 OLTP 환경에는 적합하지 않음

### 10.3.6 possible_keys 컬럼
- possible_keys 컬럼 값은 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록임
- 실제 실행 계획을 보면 그 테이블의 모든 인덱스가 목록에 포함되어 출력되는 경우가 많아서 튜닝하는 데 크게 도움되지 않음
- 특별한 경우를 제외하고는 그냥 무시해도 되는 컬럼

### 10.3.7 key 컬럼
- 최종 선택된 실행 계획에서 사용하는 인덱스를 의미함
    - 인덱스 사용하지 않으면 NULL 값이 표시됨
- 쿼리 튜닝 시 key 컬럼에 의도했던 인덱스가 표시되는지 확인하는 것이 중요함
- index_merge가 아닌 경우에는 반드시 테이블 하나당 인덱스 하나만 이용 가능

### 10.3.8 key_len 컬럼
- key_len 컬럼은 많은 사용자가 쉽게 무시하는 정보지만 사실 매우 중요한 정보임
- key_len 컬럼 값은 쿼리를 처리하기 위해 다중 컬럼으로 구성된 인덱스에서 몇 개의 컬럼까지 사용했는지 표시됨
    - 더 정확하게는 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값임
    - 인덱스 컬럼 중 NULLABLE 컬럼이 있다면 null 유무 확인하기 위해 1바이트를 추가로 더 사용함
    - 예시) char(4) 타입인 컬럼이 인덱스를 탔다면 key_len 값은 16

### 10.3.9 ref 컬럼
- 접근 방식이 ref면 참조 조건(Equal 비교 조건)으로 어떤 값이 제공했는지 보여줌
- 상수값을 지정했다면 ref 컬럼의 값은 const로 표시되고, 다른 테이블의 컬럼값이면 그 테이블명과 컬럼명이 표시됨
- 이 컬럼에 출력되는 내용은 크게 신경 쓰지 않아도 무방한데, ref 컬럼값이 'func'인 경우 조금 주의해야 함
    - 콜레이션 변환이나 값 자체의 연산을 거쳐서 참조됐다는 것을 의미함
    - 사용자가 명시적으로 값을 변환할 때뿐만 아니라 MySQL 서버가 내부적으로 값을 반환해야 할 때도 ref 컬럼에는 'func'가 출력됨

### 10.3.10 rows 컬럼
- rows 컬럼값은 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여줌
- 반환하는 레코드의 예측치가 아니라 쿼리를 처리하기 위해 얼마나 많은 레코드를 읽고 체크해야 하는지를 의미함

### 10.3.11 filtered 컬럼
- filtered 컬럼값은 인덱스를 사용하지 못하는 조건에 일치하는 레코드 비율을 의미함
- 비율이 낮을수록 성능이 좋음
- 8.0 부터 filtered 컬럼의 값을 더 정확히 예측할 수 있도록 히스토그램 기능이 도입됐음

### 10.3.12 Extra 컬럼
- 쿼리의 실행 계획에서 성능에 관련된 중요한 내용이 자주 표시됨
- Extra 컬럼에는 고정된 몇 개의 문장이 표시되는데, 일반적으로 2~3개씩 함께 표시됨
- Extra 컬럼에는 주로 내부적인 처리 알고리즘에 대해 조금 더 깊이 있는 내용을 보여주는 경우가 많음
- Extra 컬럼에 표시될 수 있는 문장을 하나씩 살펴보자. 여기서 설명하는 순서는 성능과는 무관하다.

#### 10.3.12.1 const row not found
- const 접근 방식으로 테이블을 읽었지만 실제로 해당 테이블에 레코드가 1건도 존재하지 않는 경우 표시됨
- 이러너 메시지가 표시되는 경우엔 테이블에 적절히 테스트용 데이터를 저장하고 다시 한 번 쿼리의 실행 계획을 확인해 보는 것이 좋음

#### 10.3.12.2 Deleting all rows
- MyISAM 스토리지 엔진과 같이 엔진의 핸들러 차원에서 테이블의 모든 레코드를 삭제하는 기능을 제공하는 테이블인 경우 표시됨
- WHERE 조건절이 없는 DELETE 문장의 실행 계획에서 자주 표시됨
- 이 문구는 테이블의 모든 레코드를 삭제하는 핸들러 기능(API)을 한 번 호출함으로써 처리됐다는 것을 의미함
- 조건절 없이 DELETE 하는 것보단 TRUNCATE 하는 것을 권장

#### 10.3.12.3 Distinct
- 쿼리의 DISTINCT가 명시됐을 때 문구가 표시됨
- 쿼리의 DISTINCT를 처리하기 위해 조인하지 않아도 되는 항목은 모두 무시하고 꼭 필요한 것만 조인하고, 꼭 필요한 레코드만 읽음

#### 10.3.12.4 FirstMatch
- 세미 조인의 여러 최적화 중에 FirstMatch 전략이 사용되면 이 문구 출력
- FirstMatch 메시지에 함께 표시되는 테이블명은 기준 테이블을 의미함

#### 10.3.12.5 Full scan on NULL key
- 이 처리는 'col1 IN (SELECT col2 FROM ...)'과 같은 조건을 가진 쿼리에서 자주 발생할 수 있음
- col1 값이 NULL이 된다면 NULL에 대한 연산의 규칙이 적용됨
    - 서브쿼리가 1건이라도 결과 레코드를 가진다면 최종 비교 결과는 NULL
    - 서브쿼리가 1건도 결과 레코드를 가지지 않는다면 최종 비교 결과는 FALSE
- MySQL 서버가 쿼리를 실행하는 중 col1이 NULL을 만나면 차선책으로 서브쿼리 테이블에 대해서 풀 테이블 스캔을 사용할 것이라고 알려주는 키워드임

#### 10.3.12.6 Impossible HAVING
- 쿼리에 사용된 HAVING 절이 조건을 만족하는 레코드가 없을 때 표시됨
- 애플리케이션 쿼리 중 실행 계획의 이 메시지가 출력된다면 쿼리가 제대로 작성되지 못한 경우가 대부분임

#### 10.3.12.7 Impossible WHERE
- 'Impossible HAVING'과 비슷하며, WHERE 조건이 항상 FALSE가 될 수 밖에 없는 경우 이 메시지가 출력됨

#### 10.3.12.8 LooseScan
- 세미 조인 최적화 중에서 LooseScan 최적화 전략이 사용되면 해당 문구 표시됨

#### 10.3.12.9 No matching min/max row
- MIN() 또는 MAX()와 같은 집합 함수가 있는 쿼리의 조건절에 일치하는 레코드가 한 건도 없을 때 해당 문구 표시됨

#### 10.3.12.10 no matching row in const table
- const 방식으로 테이블에 접근할 때 일치하는 레코드가 없다면 해당 문구 출력됨

#### 10.3.12.11 No maching rows after partition pruning
- 파티션된 테이블에 대한 UPDATE 또는 DELETE 할 대상 레코드가 없는 경우 해당 문구 표시됨

#### 10.3.12.12 No tables used
- FROM 절이 없는 쿼리 문장이나 'FROM DUAL' 형태의 쿼리 실행 계획에서 해당 문구 표시됨

#### 10.3.12.13 Not exists
- 아우터 조인을 이용해 안티-조인을 수행하는 쿼리에서 해당 문구 표시됨
    - 안티-조인은 NOT IN , NOT EXISTS 연산자를 주로 이용
- 안티-조인은 일반 조인(INNER JOIN)을 했을 때 나오지 않는 결과만 가져오는 방식임

#### 10.3.12.14 Plan isn't ready yet
- `EXPLAIN FOR CONNECTION` 명령을 실행했을 때 해당 문구가 표시됨
- 이 경우는 대상 커넥션의 쿼리가 실행 계획을 수립할 여유 시간을 좀 더 주고 다시 `EXPLAIN FOR CONNECTION` 명령을 실행하면 됨

#### 10.3.12.15 Range checked for each record(index map:N)
- 레코드마다 인덱스 레인지 스캔을 체크할 때 해당 문구 표시됨
- Extra 컬럼의 출력 내용 중에서 '(index map: 0x1)'은 사용할지 말지를 판단하는 후보 인덱스의 순번을 나타냄
    - index_map은 16진수로 표시됨

#### 10.3.12.16 Recursive
- 재귀 쿼리의 실행 계획에 표시됨
- 8.0 버전부터 CTE(Common Table Expression)를 이용해 재귀 쿼리를 작성할 수 있게 됨
    - With as () 구문

#### 10.3.12.17 Rematerialize
- LATERAL JOIN 되는 테이블은 선행 테이블의 레코드별로 서브쿼리를 실행해서 그 결과를 임시 테이블에 저장할 때 해당 문구 표시됨

#### 10.3.12.18 Select tables optimized away
- MIN(), MAX()만 SELECT 절에 사용되거나 GROUP BY로 MIN(),MAX()를 조회하는 쿼리가 인덱스를 오름차순 또는 내림차순으로 1건만 읽는 형태의 최적화가 적용될 때, 해당 문구 표시됨

#### 10.3.12.19 Start temporary, End temporary
- 세미 조인 최적화 중에서 Duplicate Weed-out 최적화 전략이 사용되면 옵티마이저는 실행 계획의 Extra 컬럼에 'Start temporary', 'End temporary' 문구를 표시함

#### 10.3.12.20 unique row not found
- 두 개의 테이블이 각각 유니크(PK포함) 컬럼으로 아우터 조인을 수행하는 쿼리에서 아우터 테이블에 일치하는 레코드가 존재하지 않을 때 해당 문구 표시됨

#### 10.3.12.21 Using filesort
- ORDER BY 처리가 인덱스를 사용하지 못할 때만 해당 문구 표시됨
- 이는 조회된 레코드를 정렬용 메모리 버퍼에 복사해 퀵소트 또는 힙소트 알고리즘을 이용해 정렬을 수행하게 된다는 의미
- 해당 문구가 출력되면 많은 부하를 일으키므로 가능하다면 쿼리를 튜닝하거나 인덱스를 생성하는 것이 좋음

#### 10.3.12.22 Using index(커버링 인덱스)
- 데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을 때 해당 문구 표시됨
- 커버링 인덱스로 처리할 수 있을 때와 아닐 때 성능 차이는 수십 배에서 수백 배까지 날 수 있음
- 인덱스 레인지 스캔(eq_ref, ref, range, index_merge 등)을 사용할 때만 커버링 인덱스로 처리되는 것은 아님
- 인덱스 풀 스캔을 실행할 때도 커버링 인덱스로 처리될 수 있는데, 이때도 똑같은 인덱스 풀 스캔의 접근 방식이라면 커버링 인덱스가 아닌 경우보다 훨씬 빠르게 처리됨

#### 10.3.12.23 Using index condition
- 옵티마이저가 인덱스 컨디션 푸시다운 최적화를 사용하면 해당 문구 표시됨

#### 10.3.12.24 Using index for group-by
- GROUP BY 처리를 하기 위해 MySQL 서버는 그루핑 기준 컬럼을 이용해 정렬 작업을 수행하고 다시 정렬된 결과를 그루핑하는 형태의 고부하 작업을 핋요로 함
- 하지만 GROUP BY 처리가 인덱스를 이용해서 별도의 추가 정렬 없이 수행됄 때 해당 문구 출력

##### 10.3.12.24.1 타이트 인덱스 스캔을 통한 GROUP BY 처리

##### 10.3.12.24.2 루스 인덱스 스캔을 통한 GROUP BY 처리

#### 10.3.12.25 Using index for skip scan

#### 10.3.12.26 Using join buffer(Block Nested Loop), Using join buffer(Batched Key Access), Using join buffer(hash join)

#### 10.3.12.27 Using MRR

#### 10.3.12.28 Using sort_union(...), Using union(...), Using intersect(...)

#### 10.3.12.29 Using temporary

#### 10.3.12.30 Using where

#### 10.3.12.31 Zero limit
