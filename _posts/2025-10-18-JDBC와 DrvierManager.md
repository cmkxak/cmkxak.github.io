### 데이터베이스 연결

s
- jdbc가 제공하는 DriverManager는 **라이브러리에 등록된 db 드라이버들을 관리**하고, **커넥션을 획득하는 기능**을 제공한다.
- 커넥션이 필요하면 애플리케이션 로직에서 `DriverManager.getConnection()`을 호출한다.
- DriverManager는 라이브러리에 등록된 드라이버 목록을 자동으로 인식한다. 이 드라이버들에게 순차적으로 정보를 넘겨 커넥션을 획득할 수 있는지 확인한다.

### JDBC 개발 - 등록

```java
@Slf4j
public class MemberRepositoryV0 {

    public Member save(Member member) throws SQLException {
        String sql = "insert into member(member_id, money) values(?, ?)";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            pstmt.setInt(2, member.getMoney());
            pstmt.executeUpdate();

            return member;
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }
    }

    private void close(Connection con, Statement stmt, ResultSet rs) {
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

    private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }
}
```

- prepareStatement)
    - db에 전달할 sql과 파라미터로 전달할 데이터들을 준비한다.
- 쿼리를 실행한 뒤에는 리소스를 반납해야 한다.
    - Connection, PrepareStatement 등 자원을 반납한다.
    - 리소스를 반납할 경우에는 항상 역순으로 진행한다.
- sql injection 방지를 위해 PreparedStatement를 통한 파라미터 바인딩 방식을 사용한다.

### JDBC 개발 - 조회

```java
@Slf4j
public class MemberRepositoryV0 {

    public Member findById(String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, memberId);
            rs = pstmt.executeQuery();

            if (rs.next()) {
                Member member = new Member();
                member.setMemberId(rs.getString("member_id"));
                member.setMoney(rs.getInt("money"));

                return member;
            } else {
                throw new NoSuchElementException("member not found memberId=" + memberId);
            }
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, rs);
        }
    }

    private void close(Connection con, Statement stmt, ResultSet rs) {
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

    private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }
}
```

### ResultSet

![image.png](attachment:3475b464-a23c-498f-82c8-6f3f4c2fbbdc:image.png)

- ResultSet은 위와 같이 생긴 데이터 구조
- 보통 select 쿼리의 결과가 순서대로 들어갑니다.
- 예를 들어 select member_id, money라고 지정하면, member_id와 money라는 이름으로 데이터가 저장됩니다.
- ResultSet 내부에 있는 커서를 이동해서 다음 데이터를 조회합니다.
- `rs.next()`
    - 호출 시 커서가 다음으로 이동합니다.
    - 최초의 커서는 데이터를 가리키고 있지 않으므로, `rs.next()` 최초 한번은 호출해야 데이터를 정상적으로 조회할 수 있습니다.
    - rs.next()의 결과가 true이면, 이동된 커서 위치에 데이터가 있다는 것을 의미합니다.
    - false이면, 커서가 가리키는 위치에 데이터가 없는 것을 의미합니다.
    - `rs.getString("member_id")` : 현재 커서가 가리키고 있는 위치의 `member_id` 데이터를 `String` 타입으로 반환한다.
    - `rs.getInt("money")` : 현재 커서가 가리키고 있는 위치의 `money` 데이터를 `int` 타입으로 반환한다.

### JDBC 개발 - 수정, 삭제

```java
@Slf4j
public class MemberRepositoryV0 {

    public void update(String memberId, int money) throws SQLException {
        String sql = "update member set money=? where member_id=?";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setInt(1, money);
            pstmt.setString(2, memberId);
            int resultSize = pstmt.executeUpdate();
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }
    }

    public void delete(String memberId) throws SQLException {
        String sql = "delete from member where member_id=?";

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

    private void close(Connection con, Statement stmt, ResultSet rs) {
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

    private Connection getConnection() {
        return DBConnectionUtil.getConnection();
    }
}
```

###