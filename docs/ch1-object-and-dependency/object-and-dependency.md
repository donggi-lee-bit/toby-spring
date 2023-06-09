# 1장 오브젝트와 의존관계

1장에서는 스프링이 관심을 갖는 대상인 오브젝트의 설계와 구현, 동작 원리에 대해 집중적으로 알아본다. 이걸 바탕으로 스프링을 이해해보자.

## 사용자 정보를 DB에 넣고 관리하는 DAO

```java
import java.sql.Connection;
import java.sql.PreparedStatement;

public class UserDao {

    public void add(User user) {
        Connection c = ...

        PreparedStatement ps = ...
        ps.setString(1, user.getId());
        ps.setString(2, user.getName);
        ps.setString(1, user.getPassword);

        ps.executeUpdate();

        ps.close();
        c.close();
    }
    
    public void get(String id) {
        Connection c = ...
        
        // SQL 쿼리 실행 결과를 ResultSet으로 받아서 저장할 오브젝트에 옮겨주고
        // add() 메서드처럼 생성된 Connection, PreparedStatement 등 리소스 작업이 끝나면 닫아주는 로직
    } 
}
```

사용자 정보를 DB에 넣고 관리하는 UserDao 클래스가 있다. 이 클래스가 제대로 동작하는지 확인해보고 싶다. 가장 간단한 방법으로 오브젝트 스스로 자신을 검증하도록 만들어준다.

### 테스트용 main() 메서드

```java
public class UserDao {
    
    public void add(User user) {
        // ...
    }
    
    public void get(String id) {
        // ...
    }

    public static void main(String[] args) {
        UserDao userDao = new UserDao();
        
        User user = new User();
        user.setId("hello");
        user.setName("toby");
        user.setPassword("password1234");
        
        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");
        
        // ...
    }
}
```
위처럼 오브젝트 내부에 main() 메서드를 만들어서 실행하면 성공 여부를 확인할 수 있다. 로직을 실행하는데 문제가 없다면 등록 성공이 출력될 것이다. 
<br><br>

`UserDao` 클래스 코드에는 여러 문제가 있다. 어떤 문제일까
- 잘 동작하는 코드를 개선해야 하는 이유는 뭘까
- 코드를 개선했을 때 얻는 장점은 뭘까
- 장점들이 지금, 그리고 미래에 주는 유익은 뭘까
- 객체지향 설계의 원칙과는 무슨 상관이 있을까
- 코드를 개선했을때와 그냥 사용했을때 스프링을 사용하는 개발에서 무슨 차이가 있을까

## DAO의 분리

세상에는 변하는 것과 변하지 않는 것이 있다. 하지만 객체지향 세계에서는 모든 것이 변한다고 한다. 변수나 오브젝트 필드 값이 변한다는 게 아니라 오브젝트에 대한 설계와 이를 구현한 코드가 변한다는 뜻이다.
<br><br>

개발자는 객체를 설계할 때 미래의 변화를 어떻게 대비할지 염두해야한다. 미래는 먼 미래만 있는 게 아닌, 몇 시간, 며칠 후가 될 수 있다. 객체지향이 절차적 패러다임에 비해 초기에 번거로운 작업이 많은 이유는 객체지향 기술이 변화에 효과적으로 대처할 수 있다는 특징 때문이다. <br>
<br>
미래를 어떻게 대비할 것인가
- 변화의 폭을 최소한으로 줄인다
  - 변경이 일어날 때 필요한 작업을 최소화하고, 그 변경이 다른 곳에 문제를 일으키지 않을 수 있던 건, `분리와 확장` 을 고려한 설계가 있었기 때문

분리에 대해
- 모든 변경과 발전은 한 번에 한 가지 관심사항에 집중해서 일어남
- **변화는 한 가지 관심사항에 대해 일어나지만 그에 따른 작업은 한곳에서 집중되지 않음**

변화가 한 번에 한 가지 관심에 집중되어 일어난다면
- 우리는 한 가지 관심이 한 군데에서 일어나게끔 해야한다
- 관심이 같은 것끼리 모으고, 관심이 다른 것은 따로 떨어져있게 한다

관심사를 뭉뚱그려서 한데 모으는 건 쉽다
- 뭉뚱그려진 관심사를 적절하게 분리해야할 때가 온다
  - 관심사가 같은 것끼리 모으고, 다른 것은 분리해줌으로써 같은 관심에 효과적으로 집중할 수 있게 만들어주는 것

### 관심사 분리 (커넥션 만드는 로직 분리)

DAO 클래스에서 add() 메서드는 
- DB와 연결을 위한 커넥션 로직
- DB를 동작시킬 query 문을 담는 Statement를 만들고 실행하는 로직
- Statement, Connection 오브젝트를 닫는 로직

세 가지 관심 사항이 존재한다. 먼저 DB와 연결을 위한 커넥션을 만드는 로직을 분리해본다. 
<br><br>
커넥션 로직은 add() 메서드와 get() 메서드에서 사용되고 있지만 추후 더 많은 메서드에서 커넥션을 연결이 필요할 수 있다. 그렇게 됐을 때 커넥션 로직에 변경이 생겼을 때 변경하는 게 골치 아플 것이다.

### 중복 코드 메서드 추출

```java
public class UserDao {
    
    public void add() {
        Connection c = getConnection();
        // ...
    }
    
    public void get() {
        Connection c = getConnection();
        // ...
    }
    
    private Connection getConnection() {
        // ...
        return new Connection();
    }
}
```

중복된 DB 연결 코드를 `getConnection()` 메서드로 추출해주었다. 분리한 `getConnection()` 메서드를 호출하여 DB 커넥션을 가져올 것이다. <br>
<br>
이렇게 되면 나중에 DB 커넥션이 필요한 메서드가 많아졌을 때 DB 연결과 관련된 수정사항이 생겨도 `getConnection()` 메서드의 코드만 수정해주면 된다. 독립적으로 관심을 집중해주었기 때문에 수정이 간단해졌다.

### 다른 종류의 DB를 연결할 수 있는 getConnection() 메서드

책에서 재밌는 예시로 UserDao 클래스가 유명해져서 외부에 납품할 수 있게 된다고 한다. 1번 회사와 2번 회사는 각기 다른 DB를 사용 중이라 `getConnection()` 메서드의 소스 코드를 변경해야하는 상황이다. 하지만 유명해진 UserDao 클래스의 소스코드를 직접 공개하고 싶지 않다고 한다. 이럴 때는
- UserDao 클래스를 `abstract class` 로 만든다
- `getConnection()` 메서드의 구현 코드를 제거하여 `abstract` 메서드로 만든다
  - 추상 메서드는 메서드 코드는 없지만 메서드 자체는 존재한다
- `getConnection()` 이 외에 `add()`, `get()` 메서드의 소스코드는 그대로 유지한다
- DB 연결을 하는 사용자 측에서 UserDao 를 상속 받아서 `getConnection()` 을 연결하고자 하는 DB에 맞게 구현한다

이제 UserDao 를 수정하지 않고, UserDao 를 상속 받는 서브 클래스에서 `getConnection()` 메서드를 구현하게 된다. 
- 이렇게 `super class` 에서 기본적인 흐름을 유지하고 `서브 클래스` 에서 필요한 메서드를 직접 구현해서 사용하도록 하는 방법이 디자인 패턴 중 `템플릿 메서드 패턴` 이라고 한다  
  - `템플릿 메서드 패턴` 은 스프링에서 애용되는 디자인 패턴
- UserDao 의 `getConnection()` 메서드는 Connection 타입의 오브젝를 생성한다는 기능을 정의한 추상 메서드
  - UserDao 의 서브 클래스의 `getConnection()` 메서드는 어떤 Connection 클래스의 오브젝트를 어떻게 생성할건 결정하는 방법
  - 이렇게 `서브 클래스` 에서 구체적인 오브젝트 생성 방법을 결정하는 것을 `팩토리 메서드 패턴` 이라고 부르기도 함
    - `템플릿 메서드 패턴` 과 `팩토리 메서드 패턴` 은 의미가 다르므로 혼동에 주의할 것

#### 상속 구조의 문제

템플릿 메서드 패턴 혹은 팩토리 메서드 패턴으로 관심이 다른 코드를 분리하고, 서로 독립적으로 변경, 확장할 수 있도록 만들었다. 하지만 상속 구조로 만들 경우 여러 문제가 발생하게 된다.
- 다른 목적으로 UserDao 에 상속을 적용하기 힘듦
  - java 는 다중 상속을 허용하고 있지 않기 때문
- 슈퍼 클래스에 수정 사항이 생기면 서브 클래스도 따라 수정해야함
- 확장된 `getConnection()` 메서드를 다른 DAO 클래스에 적용할 수 없다 
  - DAO 클래스마다 `getConnection()` 메서드를 중복으로 생성하게 해야한리 

## 관심사에 따른 클래스 분리

앞에서 관심사 분리를 다음과 같은 방법으로 진행했다.
- 독립된 메서드로 분리
- 상하위 클래스로 분리

이번에는 상속관계도 아닌 완전한 독립적인 클래스로 분리해보고자 한다. DB 커넥션 부분을 서브 클래스가 아닌, 별도의 클래스로 분리한다. 그걸 UserDao가 이용하게 하는 방식.

### SimpleConnectionMaker

`SimpleConnectionMaker` 라는 클래스를 만들고 DB 생성 기능을 넣어준다. 
- `add()`, `get()` 메서드에서 사용한다.
- 각각의 메서드에서 `SimpleConnectionMaker` 오브젝트를 만들수도 있지만
  - 한 번만 `SimpleConnectionMaker` 오브젝트를 만들고 저장, 사용하게 한다

```java
public class UserDao {

  private SimpleConnectionMaker simpleConnectionMaker;

  public UserDao() {
    simpleConnectionMaker = new SimpleConnectionMaker();
  }
  
  public void add() {
      Connection c = simpleConnectionMaker.makeNewConnection();
      // ...
  }

  public User get() {
    Connection c = simpleConnectionMaker.makeNewConnection();
    // ...
  }
}
```

```java
public class SimpleConnectionMaker {
    public Connection makeNewConnection() {
        Connection c;
        
        // DB 커넥션 생성 로직
      
        return c;
    }
}
```

위와 같이 DB 연결과 관심사를 클래스로 분리하여 UserDao 에서 `SimpleConnectionMaker` 의 오브젝트를 생성하여 사용하고 있다.
- `UserDao` 는 `SimpleConnectionMaker` 에 종속적이게 됐다
  - 상속 관계로 이뤄졌을 때처럼 `UserDao` 코드의 수정 없이 DB 커넥션 생성 기능을 변경할 수 없다
  - `UserDao` 의 소스 코드를 함께 제공하지 않으면 DB 연결 방법을 바꿀 수 없게 됐다
    - 다른 DB를 연결하고자 하면 `UserDao` 에서 아래의 소스 코드를 수정해야한다 
    - `simpleConnectionMaker = new SimpleConnectionMaker()` 
  - 마찬가지로 `add()` `get()` 메서드에서 DB 커넥션 코드를 일일히 변경해야하고, 메서드가 많아지면 작업의 양도 그만큼 많아지게 된다

또한 DB 커넥션을 제공하는 클래스가 어떤 것인지 `UserDao` 가 구체적으로 알고있어야 한다.

## 인터페이스를 이용하여 느슨한 연결고리 만들기

클래스를 분리하면서 생긴 문제를 해결할 방법은
- 두 개의 클래스가 직접적으로 연결되지 않고, 추상적인 느슨한 연결고리를 통해 연결하도록 한다

`추상화` 란
- 어떤 것들의 공통적인 성격을 뽑아내 이를 따로 분리해내는 작업
- `Java` 는 추상화를 위해 `인터페이스`를 제공

`인터페이스` 는
- 자신을 구현한 클래스에 대한 구체적인 정보를 감춘다
- 인터페이스로 추상화해놓은 통로를 통해 접근하게 되면 오브젝트를 만들 때 사용할 클래스가 뭔지 몰라도 된다
- 인터페이스를 통해 접근하면 실제 구현 클래스를 바꿔도 신경쓰지 않아도 됨

![img.png](img.png)

```java
public interface ConnectionMaker {
    
    Connection makeConnection();
    
}
```

- `ConnectionMaker` 라는 인터페이스를 정의
  - 인터페이스 내부에는 DB 커넥션을 가져오는 메서드 `Connection` 타입을 반환하는 `makeConnection()` 메서드 정의
- `UserDao` 는 `ConnectionMaker` 인터페이스 타입만 알고 있다
  - 어떤 구현체(클래스)인지 상관 없이 `makeConnection()` 메서드를 호출하면 `Connection` 타입의 오브젝트를 만들어서 돌려줄 것이라고 기대하게 됨 

```java
public class UserDao { 
    
    private ConnectionMaker connectionMaker;
    
    public UserDao() {
        connectionMaker = new AConnectionMaker();
    }
    
    public void add() {
        Connection c = connectionMaker.makeConnection();
        // ...
    }

    public User get() {
        Connection c = connectionMaker.makeConnection();
        // ...
    }
}
```

`UserDao` 의 `add()` `get()` 메서드와 필드에는 `ConnectionMaker` 라는 인터페이스와 인터페이스 메서드인 `makeConnection()` 만 사용하고 있다. <br>
하지만 여전히 `AConnectionMaker` 클래스를 생성자에서 호출해서 오브젝트를 생성하고 있다.
- 인터페이스를 이용해서 DB 연결하는 구체적인 정보는 제거했지만, 어떠한 오브젝트를 사용할지를 결정하는 생성자 코드는 남아있다
- 필요할 때마다 `UserDao` 의 생성자 메서드를 직접 수정하지 않고는 자유로운 DB 커넥션 확장은 불가능한 상태
