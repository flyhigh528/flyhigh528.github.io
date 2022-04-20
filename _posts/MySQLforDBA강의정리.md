# MySQL for DBA 강의정리
https://www.youtube.com/watch?v=wfdgI5mM8Ew&t=2819s
#
## MySQL 소개
### 세계적으로 가장 많이 보급된 오픈소스 데이터베이스  
### web 애플리케이션 개발의 디폴트 표준
- 가볍고 성능좋고 높은 신뢰성 제공
- 대규모 OLTP 처리 부하에 최적화
- 복제로 인한 쉬운 확장성
### 개발/운영의 사용 용의성
- 심플한 아키텍처 및 관리툴 제공
- 다양한 플랫폼 및 개발언어 지원
- SQL + NoSQL 지원
- 성숙된 오픈소스 커뮤니티 생태계
- 듀얼 라이센스
### MySQL 기타 제품
#### MySQL Connectors
- 여러 개발 언어로 MySQL Server로 접속하기 위한 연결 제공
#### MySQL Router
- 애플리케이션과 MySQL서버 사이의 투명한 라우팅을 제공하는 경량의 미들웨어
- 부하 분산 및 고가용성 목적으로 사용됨
- MySQL InnoDB Cluster의 구성 요소
#### MySQL Shell
- JavaScript, Python, SQL 모드를 지원하는 개발 관리 툴
- X-Protocol/Standard Protocol 지원
- MySQL InnoDB Cluster 관리 API
#### MySQL on Windows(Installer & Tools)
- Windows용 설치 프로그램을 사용하면 MySQL Server외에 MySQL for Excel등 Windows환경에 특화된 툴도 포함하여 설치할 수 있음
#
## MySQL 아키텍처
### InnoDB Storage Engine
#### ANSI/ISO표준 ACID 트랜젝션 완전히 지원
- 4가지 격리 레벨 지원(RR 디폴트)
- Commit/rollback/save point
- 자동 장애 복구
- Row 레벨 잠금
- 데이터 일관성 보장
#### 인-메모리 구성
- Innodb Buffer Pool - LRU 알고리즘
- Change Buffer - Secodnary Index
- Redo Log Buffer
#### 온-디스크 구성
- 여러 테이블스페이스 파일
- 여러 로그 파일
#
### InnoDB Tablespaces
#### Data Tablespaces
- System tablespace 파일 : ibdata1:12M; /ext/ibdata2:20M:autoextend
  - 각종 메타 데이터, 롤백 시그멘트 등 저장
  - innodb_file_per_table=OFF인 경우 실제 데이터도 저장
- **File-per-table tablespace: tablename.ibd 파일**
  - 한 테이블의 데이터, 인덱스 및 메타데이타가 싱글 파일에 포함됨
  - innodb_file_per_table=ON인 경우, 테이블 단위로 파일이 작성됨(8.0 디폴트 설정)
  - 압축 및 공간 회수에 유리
  - 데이터가 많은 테이블이 삭제되는 경우 자원 소모가 많다.
- **General tablespace파일 : 테이블 명.ibd 파일**
  - 여러 테이블이 하나의 테이블 스페이스 파일을 공유, 공간 재활용에 유리
  - 데이터가 많은 테이블이 삭제되는 경우에도 파일은 삭제가 되지않아 자원소모가 적다.
```
    CREATE TABLESPACE tablespace_name
    [ADD DATAFILE 'file_name']
    [FILE_BLOCK_SIZE = value]
    [ENGINE [=] engine_name]
```
- Undo Tablespaces : innodb_undo_001, ...002, ...
  - 변경 전 데이터를 저장, 롤백 및 읽기 일관성 보장
  - 독립된 tablespace 파일에 저장(MySQL 8.0)
  - 자동 축소됨
    - 디폴트로 128번 Purge thread 실행될 때마다 사이즈를 줄임
- Temporary tablespaces
  - Session(temp_1.ibt, ..._2.ibt, ...) : 사용자 및 내부 프로세스
    - innodb_temp_data_file_path = ibtmp1:12M:autoextend:max:5G
  - Global(ibtemp1) : undo log의 롤백시그멘트 저장
```
    -- temp tablespace status 조회
    show status like '%tmp%';
```
#### InnoDB 트랜젝션 로그 파일
- InnoDB Redo 로그 : ib_logfile0, ...1, .....
  - InnoDB 변경 작업을 기록하여 크래시 복구를 위한 로그 파일
  - 충분히 크게 설정(사용율 70%)
    - innodb_log_file_in_group
    - innodb_log_file_size
    - innodb_log_writer_threads(MySQL 8.0.22)
```
    -- 로그 설정 조회
    show variables where variable_name like '%log%';
```
- InnoDB Undo 로그
  - 트랜젝션 커밋하기 전 데이터 저장 - rollback segments
  - 두개의 디폴트 undo tablespace에 저장

#### 바이너리 로그 파일
실행한 쿼리 혹은 데이터 변경 처리 내용을 기록하여 **MySQL Replication**에 사용
- 바이너리 포맷, mysqlbinlog를 사용하여 텍스트로 전환가능
```
  $> mysqlbinlog --no-defaults -v binlog.000024 > t.log
```
```
    -- 바이너리 로그 파일 조회
    SHOW BINARY LOGS;
    -- 바이너리 이벤트 출력
    SHOW BINLOG EVENTS
    -- 특정 바이너리 로그 파일 이벤트 조회
    show binlog events in 'binlog.000007' limit 100;

    Rotate/flush logs - FLUSH BINARY LOGS❔❔❔❔❔❔❔❔❔❔❔
```
- binlog_format으로 기록 포맷 설정
  - ROW
  - STATEMENT
  - MIXED
- 필터링 가능(--binlog-do-db/--binlog-ignore-db) /etc/my.cnf 옵션파일 설정
- 데이터 Point-in-time Recovery에 사용
- MySQL 8.0에서 디폴트로 활성화, 시스템 변수 log_bin으로 제어
- MySQL 8.0에서 디폴트 자동삭제 기능 활성화(binlog-expire-logs-seconds)
  - 기존 사용하던 expire_logs_days는 권장하지 않고 폐지 예정
- 수동 삭제 가능
```
    PURGE  BINARY LOGS TO/BEFORE '파일명'
```

#### InnoDB Work Flow
- Crash safe를 보장할 수 있는 옵션들
  ```
  innodb_flush_log_at_trx_commit = 1 : 커밋할때마다 로그에 저장
  sync_binlog = 1
  gtid_mode = on
  enforce_gtid_consistency = 1
  binlog_gtid_simple_recovery = 1
  relay_log_recovery = ON
  master_info_repository = TABLE
  relay_log_info_repository = TABLE
  ```
- 추가 복제 관련 옵션
  ```
  -- Replication 환경에서 master가 문제가 생긴 경우, Slave를 승격하여 사용하는데 복제 과정에서 데이터 loss를 방지하기 위한 옵션
  rpl-semi-sync-master-enabled = 1
  rpl-semi-sync-slave-enabled = 1
  rpl_semi_sync_master_wait_no_slave = 1
  ```
#### 기타 로그 파일
- 에러로그
  - 시작/중지 및 서버측 오류 혹은 경고 메시지에 관련된 로그 파일
  - 디폴트 출력 위치는 플랫폼에 따라 다름
    - log_error
    - log_error_verbosity
  - JSON 포맷 지원(8.0버전)
    - https://dev.mysql.com/doc/refman/8.0/en/error-log-json.html
  - 테이블에 기록(MySQL 8.0.22)
    ```
    SELECT * FROM performance_schema.error_log\G
    ```
  - 일반 쿼리 로그
    - 클라이언트로 부터의 연결 및 실행한 전체 SQL문을 기록
    - log_output /general_log로 위치 설정
  - **Slow 쿼리 로그**
    - 지정된 시간 이상으로 실행된 쿼리를 기록
    - log_output /slow_query_log로 위치 설정
    - log_query_time : 초단위로 지정(0.5로 지정하는 경우 500ms)
    - log_queries_not_using_indexes : 인덱스를 사용하지 않은 쿼리를 기록
  - Enterprise Audit Log
    - EE에서 지원, 데이터베이스에 대한 액티비티를 기록
    - 정책 기반 필터링 및 암호화 지원
#
## MySQL 관리 및 모니터링
### 서버 상태 변수
```
    SHOW GLOBAL|SESSION STATUS;
```
#### **모니터링 주요 상태 변수**
- Innodb_buffer_pool_wait_free
  - 일반적으로 InnoDB 버퍼 풀에 대한 쓰기는 백그라운드에서 발생합니다.
  - InnoDB가 페이지를 읽거나 작성해야 하고 사용 가능한 클린 페이지가 없는 경우 InnoDB는 더티 페이지를 먼저 플러시하고 해당 작업이 완료되기를 기다립니다.
  - 이 카운터는 이러한 대기 인스턴스를 계산합니다. innodb_buffer_pool_size가 올바르게 설정된 경우 값은 작아야 합니다.
- innodb_log_waits
  - 로그 버퍼가 너무 작아서 계속하기 전에 플러시하기 위해 대기 해야하는 횟수
- threads_connected
  - 현재 열려있는 연결 수
- threads_created
  - 연결을 처리하기 위해 작성된 thread 수 입니다.
  - threads_created가 큰 경우 threads_cache_size 값을 증가 시킬 수 있습니다.
  - **cache miss rate는 threads_created/Connections로 계산할 수 있습니다.**
- threads_running
  - Sleep 상태가 아닌 thread 수 입니다.
- threads_cached
  - thread_cache의 thread 수
  - 이 변수는 임베디드 서버(libmysqld)에서 의미가 없으며, MySQL 5.7.2부터 임베디드 서버에서 볼 수 없습니다.
- connections
  - MySQL 서버에 대한 연결 시도 횟수(성공 여부 포함)
- **max_used_connections**
  - 서버가 시작된 이후 동시 사용중인 최대 연결 수입니다.
```
    -- 설정 확인
    show variables like '%connect%';
    -- 성능 분석 계산
    show status like '%connect%';
    Cache Miss Rate(%) =  Threads_created / Connections * 100
    Connection Miss Rate(%) = Aborted_connects / Connections * 100
    Connection Usage(%) = Threads_connected / max_connections * 100
```
- created_tmp_disk_tables
  - 명령문을 실행하는 동안 서버가 작성한 내부 온 디스크 임시 테이블 수
- handler_read_first
  - 인덱스의 첫번째 항목을 읽은 횟수입니다.
  - 이 값이 크면 테이블이 쿼리에 대해 올바르게 인덱싱 된 것입니다.
- open_table_%
  - definitions : 캐시된 .frm파일 수
- opened_table_%
  - definitions : 캐시 되었던 .frm파일 수
- **select_full_join**
  - 인덱스를 사용하지 않고 테이블 스캔을 수행하는 조인 수입니다.
  - **이 값이 0이 아닌 경우 테이블 인덱스를 신중하게 확인해야 합니다.**
- **slow_queries**
  - long_query_time 초 이상 걸린 쿼리 수입니다.
  - 이 카운터는 slow_query_log 사용여부와 관계없이 증가합니다.
  - slow_query_log 상태가 ON인 경우 로그를 통해 Time, User@Host, Query_time, Lock_time, Query를 확인할 수 있다.
- com_xxx_%(commit|rollback)
  - 각 xxx문이 실행 된 횟수를 나타냅니다. 각 유형의 명령문마다 하나의 상태변수가 있습니다. 
  - ex) com_delete, com_update ...
- uptime
  - 서버가 작동한 시간(초)
#
### MySQL Metadata 정보
INFORMATION_SCHEMA를 통해 MySQL서버의 metadata 정보(view로 제공) 확인
```
    USE INFORMATION_SCHEMA;
    SHOW TABLES;
    SELECT TABLE_NAME, FROM INFORMATION_SCHEMA.TABLES
    WHERE TABLE_SCHEMA = 'information_schema'
    ORDER BY TABLE_NAME\G

    DESC TABLES;
    SHOW CHARACTER SET;
    SHOW COLLATION;
```
#
### MySQL 성능 지표
PERFORMANCE_SCHEMA를 통해 MySQL서버의 성능 지표 수집, 메모리에 저장
```
    USE PERFORMANCE_SCHEMA;
    SHOW TABLES;
```
#
### Table Maintenance
- SQL 명령문
```
  ANALYZE TABLE [테이블명] UPDATE HISTOGRAM ON c1, c2, c3 WITH 10 BUCKETS; (8.0)
  CHECK TABLE [테이블명] : 테이블 및 뷰 오류 체크
  REPAIR TABLE : MYISAM, ARCHIVE, CSV, **InnoDB 제외**
  OPTIMIZE TABLE [테이블명] : 테이블 데이터 및 인덱스 공간 정리, 통계 갱신, **InnoDB 제외**
```
- InnoDB table
  - 자동 장애 복구
  - innochecksum : 오프라인 체크섬 툴
  - mysqldump + mysqlbinlog로 복구
  - innodb_force_recovery = 1 옵션으로 재시작
  - ALTER TABLE [테이블명] FORCE; : 공간 정리 및 통계 갱신
#
### MySQL 보안 기능
- 연결 암호화 지원
  - TLS 프로토콜
  - SSL/RSA 인증
- 패스워드 인증 관리
  - 강화된 인증 알고리즘
    - caching_sha2_password(MySQL 8.0)
  - 강화된 암호 정책
    - mysql_secure_installation
- 데이터 암호화
  - https://dev.mysql.com/doc/refman/8.0/en/encryption-functions.html
#
## MySQL 상태 모니터링
- Who?
```
    show processlist;
    kill <thread id>;
```
- What configuration do i have?
```
    show variables like '%옵션명%';
```
- What status do i have?
```
    -- mysql 상태
    show status like '%변수명%';
    -- 시스템 상태 확인
    top|free|vmstat|mpstat|netstat|iostat
    -- 프로시저를 통한 서버 상태 수집(파일생성)
    mysql>tee diag.out;
    mysql>CALL sys.diagnostics(120, 30, 'current');
    mysql>notee;
    >vi diag.out 확인
```
- How does Server/InnoDB/Performance_Schema process the jobs?
```
  -- 쿼리의 스텝 별 진행 시간까지 EXPLAIN 확인 가능
  EXPLAIN ANALYZE(MySQL 8.0.20)
  SHOW engine INNODB status
  SHOW engine INNODB mutex
  SHOW engine PERFOMANCE_SCHEMA status\G
```
- **Performance_Schema : 성능 모니터링 메모리 데이터**
  - Error
  - statments
  - transactions
  - waits
  - file
  - memory
  - Replication
  - table_id
  - table_lock

- MySQL Enterprise Monitor
  - MySQL보안 위기 및 방지
    - 보안 취약점 발견 및 해결
    - Firewall 및 Audit 모니터링
    - 유저 액세스 및 권한 모니터링
  - 시스템 성능 관리 및 개선
    - 시각적인 쿼리 분석기 및 쿼리 성능 모니터링
    - 데이터베이스 가용성 모니터링
    - CPU 사용량, RAM 사용량, swap 사용량
  - 복제 토폴로지 및 성능
    - Replication
    - InnoDB Cluster
    - MySQL Cluster
  - 개발자 최적화 가이드 Advisor
    - 임계 값 기반 경고 및 알람 발송

  |성능 문제 발생 원인|
  |------------------|
  |테이블 스캔을 수행하는 Query|
  |디스크 임시 테이블 과다|
  |CPU 스파이크|
  |디스크 I/O 포화|
  |내부 잠금|
  |하드웨어 문제|
  |데이터베이스 및 스키마 변경사항|
  |새로운 Query 도입|
  |MySQL 설정 및 구성 문제|

#
## MySQL Best Practice
### 초기 설정 파일 : my.cnf
서버 디폴트 설정 권장
파일 경로 설정 : 로그파일, IO분산 목적, SSD활용

### 추가 모니터링 옵션
- innodb_monitor_enable
```
    set global innodb_monitor_enable = all;
```
- SHOW VARIABLES LIKE 'perf%';

### 용량 설정
- innodb_buffer_pool_size : 70~80% of memory
- innodb_log_file_size : 70% usage
- max_connections : 151
- table_definition_cache > 테이블 개수
- table_open_cache : 4000
- table_open_cache_instances = 16

### 여러 버퍼 값은 디폴트로 시작, 세션별 동적 설정 권장
- join_buffer_size
- sort_buffer_size
- tmp_table_size

### 데이터 일관성 (default)
- innodb_flush_log_at_trx_commit = 1 
- sync_binlog=1

### **확인 및 변경 방법**
```
    SHOW GLOBAL/SESSION VARIABLES LIKE '%변수명%';
    SELECT @@global/session.변수명;
    SET @@sql_mode = 'TRADITIONAL';
    SET PERSIST max_connections = 1000;(MySQL 8.0)
```
#
## MySQL Databases 설계
데이터 타입을 선택할 떄 **공간, 효율성, 운영** 문제 고려
- PK를 AUTO_INCREMENT로 정의할 경우 INT 혹은 BIGINT 선택?
  - 데이터가 많이쌓일 것으로 예상되는 경우 BIGINT로 설계
- Unsigned 대신에 Signed 
  - 월별 매출 건수를 계산 시, Unsigned인 경우 마이너스(-) 처리가 안됨
- DECIMAL(n,m) 대신에 INT
  - 금액 계산부하가 많은 경우 INT가 효율이 좋음
- INET_NTOA(), INET_ATON()
  - IP의 경우 STRING으로 사용
- String타입에 대하여 UTF8MB4 권장
  - 테이블과 포함된 데이터 캐릭터셋 변경
```
    ALTER TABLE [테이블명] CONVERT TO CHARSET utf8mb4;
```
- 상태 값은 CHAR(1) + CHECK (8.0.16)
```
    'S','T','D','U' 이외의 값은 체크해서 오류를 반환(무효데이터 방지)
    CONSTRAINT 'status_chk' CHECK (status in ('S','T','D','U'))
```
- JSON 타입(8.0.17) - 사용자 등록 테이블, 사용자 태그
  - JSON 함수 지원
  - Functional 인덱스 지원
```
    -- 예를 들어 사용자 테이블에 운영 태그가 늘어날 경우 JSON을 이용하여 추가가 용이
    CREATE TABLE UserLogin(
        userId BIGINT NOT NULL,
        loginInfo JSON,
        PRIMARY KEY(userid));
    )
```

#
## **인덱스 설계**
조회 성능을 향상하기 위한 데이터 구조
## InnoDB_Clustered_Index
<img src="./images/InnoDB_Clustered_Index.jpg"  title="InnoDB_Clustered_Index" alt="InnoDB_Clustered_Index"></img>
Primary Key에 의해 데이터가 구성
Primary Key의 값으로 Non-leaf 페이지를 구성
모든 row값은 leaf 페이지에 구성
Primary Key를 통한 데이터 조회 시에는 바로 메모리에서 Row를 조회하여 디스크 I/O가 발생하지 않음

Secondary index는 Secondary index column값을 Non-leaf 페이지를 구성
leaf 페이지는 Primary Key Value
Primary Key Value를 이용하여 Primary Key를 조회하여 전체 Row에 접근

## 인덱스는 많으면 많을수록 좋을까요?
- 조회에 꼭 필요한 인덱스만 만들고, 사용하지 않는 인덱스는 삭제할 것을 권장
```
    -- 사용하지않는 인덱스 조회
    select * from sys.schema_unused_indexes where object_schema != 'performance_schema';
    -- 
    alter index [idx_name] invisible/visible;(MySQL 8.0)
```
- 복합 인덱스 활용할 것을 권장 (a,b)
  - 인덱스 개수를 줄임
  - Using filesort를 피할 수 있음(order by 사용 시)
  - Convering Index로 Back Table을 피할 수 있음 : (Extra: Using Index)
- 인덱스 컬럼 사이즈는 영향이 있을까요?
  - 공간 절약, 인덱스 스캔 효율성 향상
    - 16byte 혹은 이하로 권장, 32byte이상 피해야 함
    - Prefix 인덱스 활용
    - MD5로 해시키 생성