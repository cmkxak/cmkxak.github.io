---
title:  "Spring db - Connection Pool과 DataSource"
excerpt: "Connection Pool과 DataSource는 무엇일까?"

toc: true
toc_sticky: true
 
date: 2025-10-18
last_modified_at: 2025-10-18
---


### 문제점 : 데이터베이스 커넥션을 매번 획득하는 Case.

- 애플리케이션 로직은 DB 드라이버를 통해 커넥션을 조회한다.
- DB 드라이버는 DB와 TCP/IP 커넥션을 연결한다. 이 과정에서 3-way handshake와 같은 TCP/IP 연결을 위한 네트워크 동작이 발생한다.
- DB 드라이버는 TCP/IP 커넥션이 연결되면, ID, PW 부가정보를 DB에 전달한다.
- DB는 ID, PW를 통해 내부 인증을 완료하고, 내부에 DB 세션을 생성한다.
- DB는 커넥션 생성이 완료되었다는 응답을 전달한다.
- DB 드라이버는 커넥션 객체를 생성하여 클라이언트에 반환한다.

### 해결점 - Connection Pool

- 애플리케이션 시작 시점에 커넥션 풀에 필요한 커넥션을 미리 확보하여 담아둔다.
    - 서비스의 특징과 서버 스펙에 따라 커넥션의 개수는 다르지만, 기본값은 10개이다.
- 매번 요청시마다 커넥션을 생성하는 것이 아니라 커넥션 풀에 있는 커넥션들을 가져와서 커넥션을 활용한다.
- 커넥션을 사용한 뒤, 커넥션을 종료하는 것이 아니라 재활용이 가능하도록 커넥션을 그대로 커넥션 풀에 반환한다. 이 때, 커넥션을 종료하는 것이 아니라 살아 있는 커넥션을 그대로 커넥션 풀에 반환해야 한다.
- 성능과 사용의 편리함 측면에서 최근에는 Connection Pool로 hikariCP를 주로 사용한다.

### DataSource 이해

- 애플리케이션 로직에서 DriverManager를 활용해 커넥션을 획득하다가 HikariCP와 같은 커넥션 풀을 사용하여 커넥션을 획득하도록 바꾸려면, 커넥션을 획득하는 애플리케이션 코드도 함께 변경해야 한다. 의존관계가 DriverManager에서 HikariCP로 변경되기 때문이다.
- 자바에서는 구현체에 의존하여 OCP 원칙을 지키지 못하고 있는 문제를 해결하기 위해 javax.sql.**DataSource 인터페이스를 제공**한다.
- DataSource는 **커넥션을 획득하는 방법을 추상화**하는 인터페이스이다.
- 대부분의 커넥션 풀은 DataSource 인터페이스를 이미 구현해 두었다. 따라서, 개발자는 DBCP2 커넥션 풀, HikariCP 커넥션 풀의 코드를 직접 의존하는 것이 아닌, Datasource 인터페이스에만 의존하도록 애플리케이션 로직을 작성하면 된다.
- DriverManager는 Datasource 인터페이스를 구현하지 않는 문제가 있어 스프링이 이러한 문제를 해결하기 위해 **DriverManagerDataSource**라는 DataSource를 구현한 클래스를 제공한다.

### DataSource 예제 1 - DriverManager

```java
@Slf4j
public class ConnectionTest {

    @Test
    public void driverManager() throws Exception {
        Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD);
        log.info("connection={}, class={}", con1, con1.getClass());
        log.info("connection={}, class={}", con2, con2.getClass());
    }

    @Test
    public void dataSourceDriverManager() throws Exception {
        // 항상 새로운 커넥션 획득
        DataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
        userDataSource(dataSource);
    }

    private void userDataSource(DataSource dataSource) throws SQLException {
        Connection con1 = dataSource.getConnection();
        Connection con2 = dataSource.getConnection();
        log.info("connection={}, class={}", con1, con1.getClass());
        log.info("connection={}, class={}", con2, con2.getClass());
    }
}
```

- DriverManager는 커넥션을 획득할 때 마다 URL, USERNAME, PASSWORD와 같은 파라미터를 계속 전달해야 한다. 하지만, DataSource를 사용하는 방식은 객체 생성 시에만 필요한 파라미터를 넘겨주고, 커넥션 획득 시에는 단순히 getConnection() 메소드만 호출하면 된다.

### 설정과 사용의 분리

- 설정 : Datasource 생성 시 필요한 속성들을 활용해 해당 속성이 설정된Datasource 인스턴스를 생성합니다.
- 사용 : 설정은 신경 쓰지 않고, 필요한 기능이 담긴 메소드만 호출하여 사용합니다.
- 설정과 사용이 분리되지 않으면, 설정 속성이 변경될 때마다 해당 객체를 사용하는 모든 곳을 수정해야 한다. 따라서, 설정하는 곳은 따로 분리해두고, 사용하는 곳에서는 설정 내용을 모른채 단순히 필요한 기능만 활용하면 된다.
- 이 부분이 작아 보이지만 실제 운영 시 설정과 사용이 분리되지 않아 설정값이 변경되어 사용되는 부분을 모두 수정해야 한다고 생각하면 설정과 사용의 분리가 큰 차이를 만든다는 것이 이해될 것이다.

### DataSource 예제 2 - Connection Pool

```java
@Slf4j
public class ConnectionTest {

    @Test
    public void dataSourceConnectionPool() throws Exception {
        // 커넥션 풀링
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(URL);
        dataSource.setUsername(USERNAME);
        dataSource.setPassword(PASSWORD);
        dataSource.setMaximumPoolSize(10);
        dataSource.setPoolName("yhw");

        userDataSource(dataSource);
        Thread.sleep(1000);
    }

    private void userDataSource(DataSource dataSource) throws SQLException {
        Connection con1 = dataSource.getConnection();
        Connection con2 = dataSource.getConnection();
        log.info("connection={}, class={}", con1, con1.getClass());
        log.info("connection={}, class={}", con2, con2.getClass());
    }
}
```

- 커넥션 풀에서 `커넥션을 생성하는 작업`은 애플리케이션 실행 속도에 영향을 주지 않기 위해 **별도의 쓰레드에서 작동**한다.

### DataSource 적용

```java
/**
 * @apiNote Version 1
 * DataSource는 커넥션을 획득하는 방법을 추상화하는 인터페이스이다.
 * 커넥션 획득 시, 인터페이스에 의존하여 커넥션 획득 방식이 바뀌어도
 * 애플리케이션 로직은 영향이 받지 않도록 합니다. (OCP Principle)
 */
@Slf4j
@RequiredArgsConstructor
public class MemberRepositoryV1 {

    private final DataSource datasource;

    public Member findById(String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        Connection conn = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try {
            //1. Connection 획득
            Connection connection = getConnection();

            //2. 획득한 커넥션으로 전달할 sql문 세팅
            pstmt = connection.prepareStatement(sql);
            pstmt.setString(1, memberId);

            //3. 획득한 커넥션을 통해 sql 실행
            rs = pstmt.executeQuery();

            //4. 결과 반환 -> 조회 한 결과 값을 받아서 새로운 객체를 생성 후 반환
            if (rs.next()) { //커서를 이동해서 다음 데이터를 조회
                Member member = new Member();
                member.setMemberId(rs.getString("member_id")); //커서가 가리키고 있는 위치의 member_id를 String으로 반환
                member.setMoney(rs.getInt("money")); //커서가 가리키고 있는 위치의 money를 int로 반환
                return member;
            } else {
                throw new NoSuchElementException("member not found memberId=" + memberId);
            }
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(conn, pstmt, rs);
        }
    }

    public void update(String memberId, int money) throws SQLException {
        String sql = "update member set money = ? where member_id = ?";

        Connection conn = null;
        PreparedStatement pstmt = null;

        try {
            conn = getConnection();

            // DB에 전달할 SQL을 정의한다. //
            pstmt = conn.prepareStatement(sql);
            pstmt.setInt(1, money);
            pstmt.setString(2, memberId);

            // Statement를 통해 준비된 sql을 커넥션을 통해 실제 데이터베이스에 전달한다.
            pstmt.executeUpdate();
            int resultSize = pstmt.executeUpdate();
            log.info("resultSize={}", resultSize);
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(conn, pstmt, null);
        }
    }

    public Member save(Member member) throws SQLException {
        String sql = "insert into member(member_id, money) values(?, ?)";
        Connection conn = null; //커넥션 생성 담당
        PreparedStatement pstmt = null; //sql 전달
        try {
            conn = getConnection();

            // DB에 전달할 SQL을 정의한다. //
            pstmt = conn.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            pstmt.setInt(2, member.getMoney());

            // Statement를 통해 준비된 sql을 커넥션을 통해 실제 데이터베이스에 전달한다.
            pstmt.executeUpdate();
            return member;

        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(conn, pstmt, null);
        }
    }

    public void delete(String memberId) throws SQLException {
        String sql = "delete from member where member_id = ?";
        Connection conn = null;
        PreparedStatement pstmt = null;

        try {
            conn = getConnection();

            pstmt = conn.prepareStatement(sql);
            pstmt.setString(1, memberId);
            pstmt.executeUpdate();
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(conn, pstmt, null);
        }
    }

    /**
     * 자원 반납 시 반드시 역슌으로 진행 !!
     **/
    private void close(Connection c, Statement stmt, ResultSet rs) {
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(stmt);
        JdbcUtils.closeConnection(c);
    }

    private Connection getConnection() throws SQLException {
        Connection con = datasource.getConnection();
        log.info("get connection = {}, class={}", con, con.getClass());
        return con;
    }

}

```

```java
@Slf4j
class MemberRepositoryV1Test {

    MemberRepositoryV1 repository;

    @BeforeEach
    public void beforeEach() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
//        DriverManagerDataSource dataSource = new HikariDataSource();
//        dataSource.setUrl(URL);
//        dataSource.setUsername(USERNAME);
//        dataSource.setPassword(PASSWORD);

        repository = new MemberRepositoryV1(dataSource);
    }

    @Test
    public void crud() throws Exception {
        Member member = new Member("memberV100", 10000);
        repository.save(member);

        Member findMember = repository.findById(member.getMemberId());
        log.info("findMember={}", findMember);
        assertThat(findMember).isEqualTo(member); // @Data

        repository.update(member.getMemberId(), 20000);
        Member updatedMember = repository.findById(member.getMemberId());
        assertThat(updatedMember.getMoney()).isEqualTo(20000); // @Data

        repository.delete(member.getMemberId());
        assertThatThrownBy(() -> repository.findById(member.getMemberId())).isInstanceOf(NoSuchElementException.class);
    }
}
```

- `DataSource` 의존관계 주입
    - 외부에서 `DataSource` 를 주입 받아서 사용한다. 이제 직접 만든 `DBConnectionUtil` 을 사용하지 않아도 된다.
    - `DataSource` 는 표준 인터페이스 이기 때문에 `DriverManagerDataSource` 에서 `HikariDataSource` 로 변경되어도 해당 코드를 변경하지 않아도 된다.
- `JdbcUtils` 편의 메서드
    - 스프링은 JDBC를 편리하게 다룰 수 있는 `JdbcUtils` 라는 편의 메서드를 제공한다.

    - `JdbcUtils` 을 사용하면 커넥션을 좀 더 편리하게 닫을 수 있다.
