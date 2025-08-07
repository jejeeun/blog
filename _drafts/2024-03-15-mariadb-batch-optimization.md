---
title: MariaDB 대용량 배치 성능 최적화 - BT9903 배치 10배 성능 개선기
date: 2024-03-15 09:00:00 +0900
categories: [Backend, Database]
tags: [mariadb, performance, batch, optimization, sql]
---

## 개요

저공해 차량 진입 관제 시스템의 BT9903 배치(전일 통행량 통계)가 전 지자체에서 속도가 느려지고 있었습니다. LEZ_RTVI 테이블이 커질수록 점점 더 느려져서 개선이 필요한 상황이었습니다.

## 삽질 과정

### 첫 번째 가설 (2022.12.28)

특정 시점(2022년 5월 중) 이후 속도가 현저하게 느려졌음.

확인해보니 5월에 서버 긴급 점검이 있었고 LEZ_RTVI에 REG_DT 인덱스를 잡았었음.

### 두 번째 가설 (2022.12.29)

서버 문제도 아니고 LEZ_RTVI 인덱스를 변경해서도 아님..

**BT1400 전주 주간 통계 정보 송신 쿼리**를 보면 WHERE 절에 CASE WHEN을 썼는데, CASE WHEN을 WHERE 절에 쓰면 어떠한 인덱스도 타지 못하고 FULL SCAN을 한다고 한다. 이게 가장 큰 이유 같고 또한, 조건문 + 수학식이 많아서 오래 걸린 듯함.

광주, 충북에서 BT1400 소요시간이 점진적으로 늘어난 이유 → 해당 쿼리가 FULL SCAN을 하기 때문에 LEZ_RTVI 테이블에 데이터가 많아질수록 점점 더 오래 걸리는 게 당연.

**해결방안 1. WHERE절에 조건식을 자바에서 계산하고 넘겨준다. (주간 통계 로직이 복잡해서 얼마나 걸릴지 분석해봐야 함.)**

**해결방안 2. 쿼리는 그대로 두고 검색테이블을 STATS_RTVI로 바꾼다** (처음 개발할 때는 통계테이블이 없어서 RTVI를 썼지만 지금은 통계테이블이 있다. 통계테이블로 바꿔도 누락되는 변수가 없는지 확인 필요) 

→ 안됨. STATS_RTVI는 지점별 통행량. 차선별 데이터는 없음 ㅠ.

해결방안 1로 간다.

→ 해결방안 1로 가려했는데 원인이 달랐다..

### 진짜 원인 발견

쿼리 연산자나 함수식이 WHERE절에서 인덱스가 잡혀있는 변수에 직접 적용이 되어있어야 문제인 건데.. 우리 쿼리는 아무리 봐도 비교 변수에 있는 것임.. 이상해서 계속 찾아본 결과 **NOW() 말고 SYSDATE()를 써서 그런 거였음**..

### 문제가 된 쿼리

```sql
-- BT1400 전주 주간 통계 정보 송신 쿼리
SELECT
    SIDO_CD AS SIDO_CD,
    SIGUNGU_CD AS SIGUNGU_CD,
    DATE_FORMAT(DATE_SUB(SYSDATE(), INTERVAL (#{sendTerm} + 7 - 1) DAY), '%Y%m%d') AS ST_DATE,
    DATE_FORMAT(DATE_SUB(SYSDATE(), INTERVAL #{sendTerm} DAY), '%Y%m%d') AS ED_DATE,
    LANE_ID AS LANE_ID,
    COUNT(CAR_REG_NO) AS TOTAL_COUNT,
    IFNULL(REG_DT, DATE_FORMAT(SYSDATE(), '%Y%m%d%H%i%s')) AS REG_DT
FROM LEZ_RTVI
WHERE
    LANE_ID = #{laneId}
    AND SHOT_DT >= (CASE WHEN
        /* 전주(or 전전주) 일요일이 7일이하이면 (전주or전전주에 월말, 월초가 있으면) */
        DATE_SUB(SYSDATE(), INTERVAL #{sendTerm} DAY) < LAST_DAY(SYSDATE() - INTERVAL 1 MONTH) + INTERVAL 7 DAY
        THEN
            /* 월초부터 */
            CONCAT(DATE_FORMAT(LAST_DAY(SYSDATE() - INTERVAL 1 MONTH) + INTERVAL 1 DAY, '%Y%m%d'), '000000')
        ELSE
            CONCAT(DATE_FORMAT(DATE_SUB(SYSDATE(), INTERVAL (#{sendTerm} + 7 - 1) DAY), '%Y%m%d'), '000000')
        END)
    AND SHOT_DT <= CONCAT(DATE_FORMAT(DATE_SUB(SYSDATE(), INTERVAL #{sendTerm} DAY), '%Y%m%d'), '235959')
GROUP BY SIDO_CD, SIGUNGU_CD
```

## 해결

### 최종 원인 
쿼리 WHERE 절에 NOW()가 아닌 SYSDATE()를 써서 속도 저하.

### 최종 해결
SYSDATE()를 NOW()로 변경

### EXPLAIN 결과 비교

**수정 전**

key - IDX_RTVI_04는 LANE_ID만 인덱스로 잡혀있는 키다. 그러니까 SHOT_DT 인덱스를 전혀 타지 않는 상태(WHERE 절에 쓰인 SYSDATE() 때문에)

rows - 접근되는 record 수가 7,301,376 .. 느릴 수밖에

**수정 후**

key - IDX_RTVI_01 (LANE_ID, SHOT_DT, CAR_REG_NO) 복합인덱스 잘 사용.

rows - 접근되는 record 수가 1로 수정 전보다 현저하게 줄어듬. GOOD!!!

## 다른 케이스: BT9903 배치 최적화

### 원인

BT9903은 REG_DT를 기준으로 쿼리가 돔. 근데 LEZ_RTVI에는 REG_DT 인덱스가 안 잡혀있음.

### 해결

LEZ_RTVI에는 SHOT_DT가 인덱스로 잡혀있음. 쿼리 수정 (REG_DT 기준 → SHOT_DT 기준)

```sql
-- 수정 후 코드 (StatisticsDAO.xml)
INSERT INTO STATS_RTVI         
(
    SHOT_DATE,
    SHOT_HOUR,
    SITE_ID,
    EMIS_GRADE,
    CAR_CNT,
    REG_ID,
    REG_DT
)
(SELECT
    LR.SHOT_DATE,
    LR.SHOT_HOUR,
    LR.SITE_ID,
    IFNULL(LDTLS.EMIS_GRADE05,'00') AS EMIS_GRADE,
    COUNT(LR.CAR_REG_NO) AS CAR_CNT,
    LR.REG_ID,
    LR.REG_DT
FROM (                        
    SELECT 
        SUBSTR(SHOT_DT, 1, 8) AS SHOT_DATE,
        SUBSTR(SHOT_DT, 9, 2) AS SHOT_HOUR,
        CONCAT("S", SUBSTR(LANE_ID, 2, 8)) AS SITE_ID,                  
        CAR_REG_NO,
        'BT9903' AS REG_ID,
        DATE_FORMAT(NOW(), '%Y%m%d%H%i%s') AS REG_DT                  
    FROM LEZ_RTVI 
    WHERE SHOT_DT BETWEEN 
        CONCAT(DATE_FORMAT(DATE_ADD(NOW(), INTERVAL - 1 DAY), '%Y%m%d'), '000000')
        AND CONCAT(DATE_FORMAT(DATE_ADD(NOW(), INTERVAL - 1 DAY), '%Y%m%d'), '235959')
) AS LR
LEFT JOIN (
    SELECT
        LDT.SEASON_NO,
        LDT.CAR_REG_NO,
        LDT.DISCLOSURE_TYPE,
        LS.SEASON_ST_DATE,   
        '05' AS EMIS_GRADE05                     
    FROM LEZ_DISCLOSURE_TARGET AS LDT,
        (SELECT
            SEASON_NO,
            DISCLOSURE_TYPE,
            SEASON_ST_DATE,
            SEASON_ED_DATE 
        FROM LEZ_SEASON   
        WHERE SEASON_ED_DATE = (
            SELECT MAX(SEASON_ED_DATE) AS AA 
            FROM LEZ_SEASON  
            WHERE SEASON_ST_DATE <= DATE_FORMAT(DATE_ADD(NOW(), INTERVAL - 1 DAY), '%Y%m%d')
                AND DISCLOSURE_TYPE IN ("H", "E") 
        )
        LIMIT 1
        ) AS LS
    WHERE LDT.SEASON_NO = LS.SEASON_NO                  
        AND LDT.DISCLOSURE_TYPE = LS.DISCLOSURE_TYPE                           
) AS LDTLS ON LR.CAR_REG_NO = LDTLS.CAR_REG_NO                   
GROUP BY LR.SHOT_DATE, LR.SHOT_HOUR, LR.SITE_ID, LDTLS.EMIS_GRADE05
ORDER BY LR.SHOT_DATE, LR.SHOT_HOUR, LR.SITE_ID ASC)      
ON DUPLICATE KEY UPDATE   
    CAR_CNT = CAR_CNT + VALUES(CAR_CNT),
    MOD_ID = 'BT9903',
    MOD_DT = DATE_FORMAT(NOW(), '%Y%m%d%H%i%s')
```

## 배치 시스템 구조

실제 CIMS 배치 시스템은 여러 개의 배치가 동시에 돌아가는 구조였습니다:

### 주요 배치 작업들

```xml
<!-- dispatcher-servlet.xml 스케줄러 설정 -->
<task:scheduler id="csvScheduler"/>
<task:scheduler id="statisticsRtviScheduler"/>
<task:scheduler id="cleanDiscTargetScheduler"/>

<!-- 각 배치별 실행 주기 및 딜레이 -->
<task:scheduled ref="statisticsRtviTask" method="statisticsRtviDataTask" 
                fixed-delay="60000" initial-delay="4000"/>
<task:scheduled ref="weekSendTask" method="weekSendTask" 
                fixed-delay="60000" initial-delay="42000"/>
<task:scheduled ref="numplateSendTask" method="numplateSendTask" 
                fixed-delay="60000" initial-delay="65000"/>
```

- **BT9903**: RTVI 통계 처리 (4초 딜레이로 시작)
- **BT1400**: 주간 통계 송신 (42초 딜레이로 시작)  
- **BT2400**: 일일 번호판 송신 (65초 딜레이로 시작)

### 배치 실행 과정

```java
// WeekTask.java에서 실제 사용되는 패턴
try {
    BatchScheduleData schedule = scheduleService.selectBatchScheduleData(item);
    scheduleService.insertBatchStartData(item);
    ResultData resultData = weekService.sendWeekDBData(item, sendTerm);
    
    if(resultData.isSuccess()) {
        scheduleService.updateBatchFinishData(item);
    } else {
        scheduleService.deleteBatchResultData(item);
    }
} catch(Exception e) {
    scheduleService.deleteBatchResultData(item);
    LogUtil.logWrite(properties, "Week", "Send", "Batch Error: " + e.getMessage());
}
```

1. **BATCH_RESULT 테이블**에서 실행 여부 체크
2. 배치 시작 시간 기록
3. 실제 배치 로직 실행
4. 성공 시 완료 시간 업데이트, 실패 시 레코드 삭제

### 수정된 쿼리 ID

- selectWeekDBData
- selectWeekMonthLastDBData  
- selectNumplateDBData
- insertStatsRtvi
- selectRtviCnt 

(겸사겸사 SYSDATE 쓰는 쿼리들도 같이 수정함.)

### 성능 설정

```properties
# globals.properties
Globals.InsertSize = 10000          # 배치 처리 단위
Globals.CsvFileDeleteTerm = 100     # CSV 파일 보관 기간
Globals.LogFileDeleteTerm = 30      # 로그 파일 보관 기간

# 데이터베이스 커넥션 풀
maxActive=20
maxWait=30000
```

### 발견점

다른 배치 쿼리에도 SYSDATE를 쓰는지 확인 필요.

→ BT2400 전일 통행량 배치 속도 느림 문제도 동일한 원인이었음.

## 실제 성과

### BT1400 전북 지자체 실행 시간

```sql
-- 사용된 쿼리
SELECT BT_ID, BT_DT, ST_DT, ED_DT, TIMESTAMPDIFF(MINUTE, ST_DT, ED_DT) as DIFF 
FROM BATCH_RESULT 
WHERE BT_ID='BT1400' AND BT_DT >= '20210901' 
ORDER BY ST_DT DESC;
```

**결과**:
- 수정 전: 평균 15-20분 소요
- 수정 후: 평균 2-3분 소요
- **약 7-8배 성능 향상**

### 부산 지자체 적용 결과

BT9903 배치 실행 시간이 현저히 개선됨.

## 진행

전 지자체 적용 필요

## 배치 모니터링 및 로깅

### 실제 로그 확인

```java
// LogUtil.java에서 실제 사용되는 로깅 패턴
public static void logWrite(Properties properties, String fileType, 
                           String interfaceType, String log) throws Exception {
    String logPath = properties.getProperty("Globals.LogPath");
    File logFile = new File(logPath + "/" + CommonData.sdfYmd.format(new Date()) + 
                           "_" + interfaceType + "Log.txt");
    
    BufferedWriter bw = new BufferedWriter(new FileWriter(logFile, true));
    bw.write("[" + fileType + "] " + CommonData.sdfLog.format(new Date()) + 
             " >>> " + log);
}
```

**실제 로그 예시**:
```
[RealTime] 2023.06.20 08:40:31 >>> Batch Start
[RealTime] 2023.06.20 08:43:39 >>> 20230620084031375_26000.csv File Line Count : 505
[RealTime] 2023.06.20 08:43:39 >>> Batch End
```

### 배치 결과 추적

```sql
-- BATCH_RESULT 테이블을 통한 실행 시간 추적
SELECT BT_ID, BT_DT, ST_DT, ED_DT, TIMESTAMPDIFF(MINUTE, ST_DT, ED_DT) as DIFF 
FROM BATCH_RESULT 
WHERE BT_ID='BT1400' AND BT_DT >= '20210901' 
ORDER BY ST_DT DESC;
```

각 배치마다 시작/종료 시간을 기록해서 성능 변화를 추적할 수 있었습니다.

### 트랜잭션 관리

```xml
<!-- context-transaction.xml에서 실제 설정 -->
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>

<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="*" rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

모든 배치 작업에 트랜잭션이 적용되어 실패 시 자동 롤백되도록 설정되어 있었습니다.

## 추가 발견 사항

### SYSDATE vs NOW 차이

- **SYSDATE()**: 매번 호출할 때마다 시스템 시간을 새로 가져옴 → 인덱스를 타지 못함
- **NOW()**: 쿼리 실행 시점의 시간을 캐싱해서 사용 → 인덱스 사용 가능

이런 미묘한 차이가 대용량 테이블에서는 엄청난 성능 차이를 만들어냈습니다.

### 배치 스케줄링 최적화

각 배치마다 서로 다른 initial-delay를 주어서 동시 실행을 피했습니다:
- BT9903: 4초 딜레이 (가장 중요한 통계)
- BT1400: 42초 딜레이 (주간 통계)
- BT2400: 65초 딜레이 (번호판 송신)

이렇게 하니 DB 커넥션 풀(maxActive=20)에서 리소스 경합이 줄어들었습니다.

### SFTP 파일 전송 아키텍처

배치 시스템에서 SFTP를 통한 파일 전송도 중요한 부분이었습니다:

```java
// FileTransferUtil.java - 안전한 파일 전송 로직
public void transferLinkToDB(ChannelSftp sftpLink, ChannelSftp sftpDB, 
                           String localFilePath, String fileName, int linkFileSize) {
    // 1. 기존 파일 삭제
    deleteBeforeFile(localFilePath);
    
    // 2. 연계서버에서 다운로드
    fileDownload(sftpLink, localFilePath, fileName, fileName);
    
    // 3. 파일 크기 검증 (다운로드 완료 확인)
    while(downloadFail) {
        int fileLine = countFileLine(localFile);
        if(fileLine >= linkFileSize) {
            downloadFail = false;
        }
    }
    
    // 4. DB서버로 업로드
    fileUpload(sftpDB, localFilePath, fileName);
}
```

**JSch 라이브러리 활용**:
```java
// FTPUtil.java - SFTP 연결 설정
JSch jsch = new JSch();
session = jsch.getSession(user, url, port);
session.setPassword(password);

Properties config = new Properties();
config.put("StrictHostKeyChecking", "no");  // 보안 설정
session.setConfig(config);
```

연계서버 → 운영서버 → DB서버 순으로 파일을 안전하게 전송하는 구조였습니다.

## 마무리

복잡한 최적화 기법을 생각하고 있었는데, 실제 원인은 단순한 함수 사용 차이였습니다.

SYSDATE()를 NOW()로 바꾸는 것만으로도 기존 인덱스를 제대로 활용할 수 있게 되어서 엄청난 성능 향상을 얻을 수 있었습니다.

대용량 배치 최적화에서 가장 중요한 것은 **인덱스를 제대로 타는지 확인하는 것**이라는 걸 다시 한번 깨달았습니다.

---

*실제 ejsoft 프로젝트 배치 최적화 경험을 바탕으로 작성했습니다.*