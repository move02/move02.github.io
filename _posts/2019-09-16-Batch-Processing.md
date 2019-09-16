---
layout: post
title: Batch Processing
excerpt: "Batch 프로세싱에 대한 정리와 Java(JDBC)에서의 응용"
categories: [TIL]
comments: true
---

Batch 정리
=========

Batch란 무엇인지, DB에서 사용된 Batch는 어떻게 사용된건지 정리해보았다.

## Batch?

Batch는 특정 프로그램의 기능이 아닌 일괄 처리(batch processing)를 뜻한다.

하나의 데이터당 하나의 처리를 하는 것이 아니라, 대량의 데이터를 모아서 한 번의 처리로 끝낼 때 사용되는 용어이다.

### batch processing 예시
- Transaction
  - 하나의 쿼리에 하나의 transaction이 실행된다면 너무 많은 오버헤드가 발생한다. 이를 위해 다량의 쿼리를 batch에 추가해두고, 일정 기간 혹은 일정 량을 기준으로 batch를 실행시키고 commit을 하면 효율이 늘어난다.
- Reporting
  - 어떠한 시스템에서 일어난 일들을 정리된 형태로 Report를 할 때, 모든 변경사항들을 반영하여 한 번에 처리한다.
- Researching
  - 높은 사양의 성능을 요구하는 연구 작업에서 batch processing을 이용하여 연관된 변수나 계산을 한 번에 처리한다.
- 등등

## Using Batch processing in JDBC

위의 batch processing 예시에서 설명했듯, 많은 양의 데이터를 조작해야 한다면(특히, insert)

다량의 쿼리를 batch에 추가해두고 일정 기간 혹은 일정 량을 기준으로 batch를 실행시켜 commit 하는 방법을 사용하는 것이 좋다.

PreparedStatement를 이용해 batch processing을 하는 예시
```java
public class BatchTest{
    private final static Connection connection = DriverManager.getConnection(
						this.url + this.dbName + "?characterEncoding=UTF-8&serverTimezone=UTC&useSSL=false",
						this.username, this.password);

    public static void main(String[] args){
		String query = "INSERT INTO DOC VALUES(?, ?, ?)";
		
        // 데이터 파일에서 documents를 읽어와 List<String[]> 형태로 리턴하는 가상의 함수
        List<String[]> documents = getDocumentsFromFile();

		PreparedStatement pstmt = null;
		try {
            // AutoCommit을 지원하는 DB일 경우 default가 true인 경우가 있기에 꼭 이 과정을 거쳐야함.
			this.connection.setAutoCommit(false);
			pstmt = this.connection.prepareStatement(query);
			
			int size = 0;
            
			for(String[] doc : documents) {
				int i = 1;

			    // documents의 field를 순회하면서 pstmt에 파라미터 추가
				for(String col : doc) {
					pstmt.setObject(i++, col);
				}

				// batch에 현재 루프에 파라미터가 추가된 pstmt를 추가
				pstmt.addBatch();
				
				size++;
				
                // 일정 갯수(1000개)를 넘기면 batch 실행 후 commit
				if(size > 1000) {
					pstmt.executeBatch();
					connection.commit();
					pstmt.clearBatch();
					size = 0;
				}
			}
            // 일정 갯수가 채워지지 않고 끝난 나머지 batch를 실행 후 commit()
			pstmt.executeBatch();
            connection.commit();
            pstmt.clearBatch();
		} catch (SQLException e) {
			e.printStackTrace();
			return;
		}
    }
}
```

### 쿼리 이어붙이기

나의 경우 Batch를 이용한 커밋보다 쿼리를 이어붙여 커밋하는 것이 더 빨랐다.

```java
public class BatchTest{
    public static void main(String[] args){
		long start = System.currentTimeMillis();

		// Data File 경로
		String path = "C:\\training\\DBnFile\\data\\doc.tsv";

		Connector mysqlConnector = new Connector("jdbc:mysql://localhost:3306/", "root", "daummove02");
		mysqlConnector.setDbName("training1");
		mysqlConnector.setTableName("DOC4");

		DAO<Document> dao = new DocumentDAO(mysqlConnector.connect(), mysqlConnector.getTableName());
		dao.setColNames();
		
		fileToDB(path, dao);
		long end = System.currentTimeMillis();

		System.out.println("걸린 시간 : " + (end - start) / 1000 + "sec");

		mysqlConnector.close();
    }

    // TsvReader를 이용해 파일을 읽어 bundle에 추가하는 함수.
    public static void fileToDB(String path, DAO<Document> dao) {
		TsvReader tsvReader = new TsvReader();

		List<String[]> lines = tsvReader.readFile(path);
		List<String[]> bundle = new ArrayList<>();

		for (String[] line : lines) {
			bundle.add(line);
            // bundle의 사이즈가 1000이면 한 번에 insert
			if (bundle.size() > 1000) {
				dao.insert(bundle);
				bundle.clear();
			}
		}

		dao.insert(bundle);
	}
}

public class DocumentDAO implements DAO<Document>{
    // 4. insert 1000 documents at once using multiple insert query
	@Override
	public void insert(List<String[]> bundle) {
		StringBuilder paramsBuilder = new StringBuilder("?");

		for (int i = 0; i < this.colNames.size() - 1; i++)
			paramsBuilder.append(", ?");

		String params = paramsBuilder.toString();
		String query = "INSERT INTO " + this.tabName + " VALUES (" + params + ")";
		PreparedStatement pstmt = null;
		try {
			int i = 1;
			for (int j = 1; j < bundle.size(); j++) {
				query += ",\n(" + params + ")";
			}

			pstmt = this.connection.prepareStatement(query);
			
			for (String[] doc : bundle) {
				for (String col : doc) {
					pstmt.setObject(i++, col);
				}
			}
			try {
				pstmt.execute();
			} catch (SQLIntegrityConstraintViolationException e) {
				e.printStackTrace();
			}
			pstmt.close();
		} catch (SQLException e) {
			e.printStackTrace();
			return;
		}
	}
}
```

위 방법처럼 `INSERT INTO TABLE VALUES (?, ?, ?), (?, ?, ?) ....` 이런 식으로 일정 숫자(1000개)만큼 쿼리를 이어붙여 커밋을 했을 때 가장 빨랐다. (3~5초)

**위 방법의 단점**
1. 최적의 해(한 쿼리에 들어가는 VALUE 의 수)를 찾기 위해서는 Trial하게 해봐야 한다.
2. 한 쿼리에 들어가는 Parameter의 갯수가 많아지면 아래와 같은 Exception이 발생한다.
```
com.mysql.cj.jdbc.exceptions.PacketTooBigException: Packet for query is too large (12,452,826 > 4,194,304).
```

## 참고자료
https://www.idug.org/p/bl/et/blogid=2&blogaid=602

IBM DB2 유저 커뮤니티인데, INSERT 성능을 높이는 여러가지 방법이 소개되어있다.
비단, DB2에만 해당되는 이야기는 아니라는 생각이 들어 나중에 성능 개선을 하게된다면 꼭 위 글을 참고하여 다시 개발해봐야겠다.