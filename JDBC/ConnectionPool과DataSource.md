# 커넥션 풀과 데이터소스 이해

1. 커넥션 풀(Connection Pool)

데이터베이스 커넥션을 매번 획득하는 방식은 TCP/IP 연결을 위한 네트워크 동작(3 way handshake), ID/PW 인증, DB 세션 생성 등 복잡하고 리소스를 많이 소모하는 과정을 거침.

* **문제점**: SQL 실행 시간 외에 커넥션 생성 시간이 추가되어 응답 속도에 영향을 주며, 매번 새로운 리소스를  할당해야 함.

* **해결 (커넥션 풀)**: 애플리케이션 시작 시점에 필요한 만큼 커넥션을 미리 확보하여 풀(Pool)에 보관해두고 재사용하는 방식임
* (만약 미리 확보한 풀을 넘어서 필요로 하는 커넥션이 더 있으면 내부적인(히카리 기준)설정에의해 일정시간(기본 30초)뒤 끊어짐).


* **사용 방식**: 애플리케이션 로직은 이제 DB 드라이버 대신 커넥션 풀에서 이미 생성된 커넥션을 객체 참조로 가져다 쓰고, 사용 완료 후 종료가 아닌 **풀에 반환**함 (히카리 경우 close를 하면 커넥션을 정리하는게 아니라 풀에 반환함).

<br>

2. DataSource

커넥션을 획득하는 방법은 `DriverManager`를 직접 사용하거나 여러 종류의 커넥션 풀을 사용하는 등 다양함.

* **문제점**: 커넥션 획득 방식을 변경하면(예: `DriverManager` → `HikariCP`) 해당 방식을 사용하는 모든 애플리케이션 코드도 함께 수정해야 함.


* **해결 (DataSource 인터페이스)**: 자바는 커넥션을 획득하는 방법을 추상화한 `javax.sql.DataSource` 인터페이스를 제공함.


* **추상화의 장점**: 애플리케이션 로직은 `DataSource` 인터페이스에만 의존하므로, 구현 기술을 변경해도 로직 코드를 수정할 필요가 없음 (DI와 OCP 원칙 준수).

<br>

3. 설정과 사용의 분리 

`DataSource`를 사용하면 설정 부분과 사용 부분을 명확히 분리할 수 있음.

* **설정**: `URL`, `USERNAME`, `PASSWORD` 등 접속 정보는 `DataSource` 생성 시점에만 입력함.


* **사용**: 리포지토리 등 실제 사용처에서는 `dataSource.getConnection()`만 호출하면 되며, 상세한 설정 정보를 몰라도 됨.


* **효과**: 향후 DB 접속 정보나 커넥션 풀 종류가 바뀌어도 설정 코드만 수정하면 되므로 유연한 대처가 가능함.

<br>

### 4. DataSource 기반 개발 (MemberRepositoryV1)

스프링이 제공하는 `JdbcUtils`를 활용하여 리소스를 더 편리하게 관리함.

```java
@Slf4j
public class MemberRepositoryV1 {
    private final DataSource dataSource; // 인터페이스에 의존

    public MemberRepositoryV1(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    // CRUD 로직은 V0와 유사하나 getConnection() 방식이 변경됨
    private Connection getConnection() throws SQLException {
        Connection con = dataSource.getConnection(); // DataSource를 통한 획득 
        return con;
    }

    private void close(Connection con, Statement stmt, ResultSet rs) {
        // JdbcUtils 편의 메서드 활용하여 리소스 정리
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(stmt);
        JdbcUtils.closeConnection(con); //커넥션 정리가 아닌 풀에 반환
    }
}

```
<br>

### 5. HikariDataSource 사용

* **커넥션 풀링(HikariCP) 설정**: "스프링 부트 기본 커넥션 풀 적용(10개)" 


```java
@Test
void dataSourceConnectionPool() throws SQLException, InterruptedException {
    // HikariCP 커넥션 풀링 설정 
    HikariDataSource dataSource = new HikariDataSource();
    dataSource.setJdbcUrl(URL);
    dataSource.setUsername(USERNAME);
    dataSource.setPassword(PASSWORD);
    dataSource.setMaximumPoolSize(10); // 최대 풀 사이즈 + setMaximumPoolSize설정안하면 자동으로 10개
    dataSource.setPoolName("MyPool");

    useDataSource(dataSource);
    Thread.sleep(1000); // 별도 쓰레드에서 커넥션 생성하는 시간 대기 
}

```
<br>

* **사용 예시**: "추상화된 Datasource로 받아 사용 " 
* 매번 새로운 커넥션이 아닌 만들어져있는 풀에있는걸 사용함

```java
    private void useDataSource(DataSource dataSource) throws SQLException {
        Connection con1 = dataSource.getConnection();
        Connection con2 = dataSource.getConnection();

        log.info("connection={}, class={}", con1, con1.getClass());
        log.info("connection={}, class={}", con2, con2.getClass());
    }
```

<br>

* **구현체 교체 테스트**: "로직 변경 없이 구현체만 갈아 끼우기" 


```java
    MemberRepositoryV1 repository;// DI용


    //기본 DriverManager - 항상 새로운 커넥션을 획득
    //DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);

    //커넥션 풀링
    HikariDataSource dataSource = new HikariDataSource();
    dataSource.setJdbcUrl(URL);
    dataSource.setUsername(USERNAME);
    dataSource.setPassword(PASSWORD);

    repository = new MemberRepositoryV1(dataSource);

```
<br>

### 6. 요약

1. **HikariCP 사용**: 성능, 편리함, 안전성이 검증되어 스프링 부트 2.0부터 기본으로 사용되는 표준 커넥션 풀임.


2. **리소스 반환 유의**: 커넥션 풀을 사용할 때 `close()`를 호출하면 커넥션이 물리적으로 닫히는 것이 아니라 풀에 **살아있는 상태로 반환**됨.


3. **적절한 풀 사이즈**: 서비스 특징과 서버 스펙에 따라 다르지만 기본값은 보통 10개이며, 반드시 성능 테스트를 거쳐 결정해야 함.


4. **JdbcUtils 활용**: 스프링의 `JdbcUtils`를 사용하면 리소스를 닫을 때 발생하는 예외 처리가 내재되어 있어 코드가 깔끔해짐.


5. **커넥션 생성의 비동기성**: HikariCP는 앱 실행 시간을 늦추지 않기 위해 별도 쓰레드에서 따로 커넥션을 채우므로, 테스트 시에는 `Thread.sleep` 등을 통해 생성 완료를 대기해야 로그를 확인할 수 있음 (안그러면 쓰레드가 풀에 커넥션을 채워 넣기도 전에 테스트가 끝나버림).