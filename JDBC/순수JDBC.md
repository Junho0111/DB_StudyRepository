# JDBC (Java Database Connectivity) 이해

### 1. JDBC의 개념 및 등장 배경

JDBC는 자바에서 데이터베이스에 접속할 수 있도록 하는 **자바 표준 API**임. 애플리케이션 서버가 데이터베이스를 사용하는 일반적인 과정은 커넥션 연결, SQL 전달, 결과 응답의 3단계로 이루어짐.

* **문제점**: 과거에는 DB마다 커넥션 연결 방식, SQL 전달 방법, 결과 응답 방식이 모두 달랐음. 이로 인해 DB 변경 시 관련 코드를 모두 수정해야 했으며, 개발자는 각 DB의 사용법을 새로 학습해야 하는 부담이 있었음.


* **해결**: JDBC라는 자바 표준 인터페이스가 등장하면서 개발자는 표준 인터페이스 사용법만 학습하면 수십 개의 DB에 동일하게 적용할 수 있게 됨. DB 변경 시에도 JDBC 구현 라이브러리인 **드라이버**만 교체하면 코드를 그대로 유지할 수 있음.

<br>

### 2. JDBC 표준 인터페이스와 드라이버

JDBC는 대표적으로 다음 3가지 기능을 표준 인터페이스로 정의함.

| 인터페이스 | 역할 |
| --- | --- |
| **`java.sql.Connection`** | 데이터베이스 연결 |
| **`java.sql.Statement`** | SQL을 담은 내용 전달 |
| **`java.sql.ResultSet`** | SQL 요청에 대한 결과 응답 |

* **JDBC 드라이버**: 각 DB 벤더에서 JDBC 인터페이스를 자신의 DB에 맞도록 구현하여 라이브러리로 제공하는 것임. 
(추가로 DriverManager가 각DB라이브러리를 뒤져 해당 Driver를 찾아냄)

<br>

### 3. JDBC와 최신 데이터 접근 기술

최근에는 JDBC를 직접 사용하기보다 이를 편리하게 도와주는 기술들을 활용함.

* **SQL Mapper (예: JdbcTemplate, MyBatis)**: SQL 응답 결과를 객체로 편리하게 변환하고 반복 코드를 제거해주지만, 개발자가 직접 SQL을 작성해야 함.


* **ORM 기술 (예: JPA)**: 객체를 DB 테이블과 매핑하여 SQL을 동적으로 생성해주므로 개발 생산성이 높음


* **중요**: 이러한 기술들도 내부적으로는 모두 **JDBC를 사용**

<br>

### 4. 데이터베이스 연결 실무

`DriverManager.getConnection()`을 통해 DB 커넥션을 획득함.

* **동작 원리**: `DriverManager`는 라이브러리에 등록된 드라이버 목록을 자동으로 인식하고, 접속 정보를 드라이버들에게 순서대로 넘겨 본인이 처리할 수 있는 요청인지 확인한 후 커넥션을 반환함.

<br>

### 5. JDBC - DriverManager (CRUD)

* **회원 등록 (Create)**: "`PreparedStatement`를 이용한 파라미터 바인딩 및 데이터 저장" 


```java
public Member save(Member member) throws SQLException {
    String sql = "insert into member(member_id, money) values(?,?)";
    Connection con = null;
    PreparedStatement pstmt = null;

    try {
        con = getConnection(); // 커넥션 획득 
        pstmt = con.prepareStatement(sql); // SQL 준비 
        pstmt.setString(1, member.getMemberId()); // 파라미터 바인딩 
        pstmt.setInt(2, member.getMoney());
        pstmt.executeUpdate(); // DB에 전달 

        return member;
    } catch (SQLException e) {
        log.error("db error", e);
        throw e;
    } finally {
        close(con, pstmt, null); // 리소스 정리 
    }
}

```
<br>

* **회원 조회 (Read)**: "`ResultSet` 커서 이동을 통한 데이터 추출" 


```java
public Member findById(String memberId) throws SQLException {
    String sql = "select * from member where member_id = ?";
    Connection con = null;
    PreparedStatement pstmt = null;
    ResultSet rs = null;

    try {
        con = getConnection();
        pstmt = con.prepareStatement(sql);
        pstmt.setString(1, memberId);

        rs = pstmt.executeQuery(); // 조회 결과 반환 
        if (rs.next()) { // rs.next() 처음 실행하면 그제서야 첫번째 데이터 가르킴 
            Member member = new Member();
            member.setMemberId(rs.getString("member_id")); // 결과 매핑
            member.setMoney(rs.getInt("money"));
            return member;
        } else {
            throw new NoSuchElementException("member not found memberId=" + memberId);
        }
    } catch (SQLException e) {
        log.error("db error", e);
        throw e;
    } finally {
        close(con, pstmt, rs); // ResultSet 포함 리소스 정리 
    }
}

```
<br>

* **회원 수정 (Update)**: "`executeUpdate()`의 반환값으로 영향받은 로우 수 확인" 


```java
public void update(String memberId, int money) throws SQLException {
    String sql = "update member set money = ? where member_id = ?";
    Connection con = null;
    PreparedStatement pstmt = null;

    try {
        con = getConnection();
        pstmt = con.prepareStatement(sql);
        pstmt.setInt(1, money);
        pstmt.setString(2, memberId);

        int resultSize = pstmt.executeUpdate();// 영향받은 row 수 반환 
        log.info("resultSize={}", resultSize);
    } catch (SQLException e) {
        log.error("db error", e);
        throw e;
    } finally {
        close(con, pstmt, null);
    }
}

```
<br>

* **회원 삭제 (Delete)**: "특정 PK를 지정하여 행 삭제" 


```java
public void delete(String memberId) throws SQLException {
    String sql = "delete from member where member_id = ?";
    Connection con = null;
    PreparedStatement pstmt = null;

    try {
        con = getConnection();
        pstmt = con.prepareStatement(sql);
        pstmt.setString(1, memberId);
        pstmt.executeUpdate();
    } catch (SQLException e) {
        log.error("db error", e);
        throw e;
    } finally {
        close(con, pstmt, null);
    }
}

```
<br>

---

**리소스 정리를 위한 `close()` 메서드 구현 예시** 

* **주의**: 리소스를 반환할 때는 항상 **획득한 역순**으로 종료해야 함 (ResultSet → Statement → Connection).



```java
private static void close(Connection con, Statement stmt, ResultSet rs) {
    if (rs != null) {
        try { 
            rs.close(); 
        } catch (SQLException e) {
             log.error("error", e); 
        }
    }

    if (stmt != null) {
        try { 
            stmt.close(); 
        } catch (SQLException e) {
            log.error("error", e); 
        }
    }

    if (con != null) {
        try { 
            con.close(); 
        } catch (SQLException e) {
            log.error("error", e); 
        }
    }
}

```
<br>

**커넥션 연결을 위한 `getConnection()` 메서드 구현 예시** 

* 이런식으로 설정용을 따로 생성하며

```java
public class DBConnectionUtil {

    public static Connection getConnection(){
        try {
            Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
            log.info("get connection={}, class={}", connection,connection.getClass());
            return connection;
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
}

```

* 위의 crud 구현시 아래 메서드를 구현하여 사용가능

```java
 private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }

```

<br>

### 6. 최종 요약 및 주의사항

1. **리소스 정리 필수**: `Connection`, `PreparedStatement` 등의 리소스는 사용 후 반드시 역순으로 `close()` 해야 함. 이를 누락하면 **리소스 누수**가 발생하여 장애의 원인이 될 수 있으므로 `finally` 블록에서 처리해야 함.


2. **SQL Injection 예방**: 파라미터를 직접 SQL 문자열에 더하지 말고, 반드시 `PreparedStatement`를 통한 **파라미터 바인딩 방식**을 사용해야 함.
