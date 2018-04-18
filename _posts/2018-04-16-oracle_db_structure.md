---
layout: post
title:  "[ORACLE] DB 구조"
date:   2018-04-16 17:40:00
author: garimoo
categories: Database
---
<br/>
# 11g란? grid 개요

* OGF(Open Grid Forum): 그리드 컴퓨팅의 표준을 개발하는 표준단체
* 오라클은 그리드 컴퓨팅 Infrastructure 소프트웨어 만듬. 모든 요소가 클러스트화.

### ASM(자동 저장 영역 관리)

* 모든 디스크에 DB 데이터를 분배하고, 저장 영역 그리드를 생성 및 유지
* 디스크가 추가되거나 삭제되면 ASM이 데이터를 자동으로 재분배

### RAC(Real Application Clusters)

* 서버 클러스터에서 모든 응용 프로그램 작업 로드를 실행, 조정
    * 통합 클러스터웨어: 클러스터 연결, 메시징, lock, 클러스터 제어 및 recovery 기능
    * 자동 작업 로드 관리: 각 서비스에 프로세싱 리소스를 자동으로 할당
    * Automatic event notification to the mid-tier: 클러스터 구성이 변경되면 mid-tier가 즉시 instance failover 적용

### Oracle WebLogic Application Grid

* 예측 가능한 응용 프로그램 확장성 및 성능 제공
* 몇 개의 서버에서 수 천 개의 서버까지 미들웨어 인프라를 선형적으로 확장
* in-memory 데이터 그리드 솔루션을 사용해 자주 사용되는 데이터에 빠르게 액세스 가능하게 함

### Enterprise Manager Grid Control

* 전체 소프트웨어 스택 관리
* 유저 프로비저닝
* 데이터베이스 복제 및 패치 관리 등

# 오라클 데이터베이스 구조

![Image](/assets/20180416/1.png)
오라클 데이터베이스 시스템은 `오라클 데이터베이스`와 `데이터베이스 Instance`로 구성

## instance

instance가 시작될 때마다 **SGA(System Global Area)** 라는 공유 메모리 영역이 할당, 백그라운드 프로세스 시작.

* 데이터베이스 마운트: 데이터베이스 instance를 시작 후, 해당 instance를 특정 DB와 연결하는 과정.

![Image](/assets/20180416/2.png)

* connection: user process와 오라클 DB instance 사이의 통신 경로.
* session: 데이터베이스 instance에 대한 현재 유저의 로그인 상태. 유저가 연결하는 시점에서 연결을 끊거나 DB application을 종료할 때까지 지속.

## 메모리 구조

![Image](/assets/20180416/3.png)

### SGA(System Global Area)

하나의 instance의 데이터 및 제어 정보를 포함하는 공유 메모리 구조. 모든 서버 및 백그라운드 프로세스에서 공유.

#### Shared Pool

![Image](/assets/20180416/4.png)

* 데이터 딕셔너리: 데이터베이스, 해당 구조 및 유저에 대한 참조 정보를 포함하는 데이터베이스 테이블 및 뷰 모음. SQL 구문 분석 중에 데이터 딕셔너리에 자주 액세스.
    * 데이터 딕셔너리 캐시
    * 라이브러리 캐시
* 공유 SQL 영역: DB에서 실행하는 각 SQL 문을 공유 SQL으로 나타내고, 두 유저가 같은 SQL문을 사용하는 경우에 공유 SQL 영역 재사용.
    * 지정된 SQL문에 대한 구문 분석 트리 및 실행 계획이 포함.
* 고정영역: SGA에 대한 시작 오버헤드

#### 데이터베이스 버퍼 캐시

* 데이터 파일에서 읽은 데이터 블록 복사본을 보관.
* Instance에 동시에 연결하는 모든 유저는 데이터베이스 버퍼 캐시에 대한 액세스를 공유
* LRU 에 의해 관리

#### redo 로그 버퍼

* SGA의 순환 버퍼
* DB에 대한 변경 사항 관련 정보를 보관
* DML, DDL 또는 내부 작업에 의해 DB에 수행된 변경 사항을 redo하는데 필요한 정보 포함

#### Large Pool

* Shared Server및 Oracle XA 인터페이스용 세션 메모리(트랜잭션이 여러 DB와 상호 작용하는 경우)
* I/O 서버 프로세스
* 오라클 데이터베이스 백업 및 복원 작업
* Parallel Query

공유 SQL을 캐시에 저장하는 데에 **Shared Pool**을 사용할 수 있으며, 공유 SQL 캐시를 축소하면 발생하는 성능 오버헤드를 방지할 수 있다.

#### Java Pool

JVM의 모든 세션별 Java 코드 및 데이터를 저장하는 데 사용.

#### Streams Pool

**Oracle Streams** 전용으로 사용. 버퍼링된 큐 메시지를 저장하고, Oracle Streams 캡처 프로세스 및 적용 프로세스에 대해 메모리 제공.

### PGA(Program Global Areas)

서버 프로세스에 대한 데이터 및 제어 정보를 포함하는 전용(Private) 메모리 영역. 각 서버 프로세스에는 고유한 PGA가 포함. 액세스는 배타적이다. stack space와 UGA로 구성

#### UGA(User Global Area)

* 커서 영역
* 유저 세션 데이터 저장 영역
* SQL 작업 영역
    * 정렬 영역 - ORDER BY, GROUP BY 등
    * 해시 영역 - 해시 조인
    * 비트맵 생성 영역 - 비트맵 인덱스 생성
    * 비트맥 병합 영역 - 비트맵 인덱스 계획 실행 분석

## 프로세스 구조

![Image](/assets/20180416/5.png)

### 서버 프로세스

서버 프로세스를 생성하여 Instance에 연결된 User Process의 요청을 처리. User Process란 오라클 DB에 연결하는 어플리케이션을 뜻함.

* SQL문 구문 분석 및 실행
* 디스크의 데이터 파일에서 필요한 블록을 SGA의 공유 데이터베이스 버퍼로 읽기
* 응용 프로그램이 정보를 처리할 수 있는 방식으로 결과 반환

### 백그라운드 프로세스

![Image](/assets/20180416/6.png)

#### DBWn(Database Writer Process)

* 데이터베이스 버퍼 캐시의 수정된(더티) 버퍼를 SCN 순서로 디스크에 write
* redo는 SCN 순서로 되어 있으므로 해당 Instance의 Recovery가 필요한 경우 정확한 위치에서부터 redo 읽기를 시작하고, 불필요한 I/O를 하지 않아야 한다. → _Incremental 체크포인트_
* DBWn 프로세스 동작 경우
    * 서버 프로세스가 버퍼를 스캔한 후에 재사용 가능한 클린 버퍼를 찾지 못하면 DBWn에 write를 수행하라는 신호 보냄. DBWn은 다른 처리를 수행하면서 더티 버퍼를 디스크에 비동기 방식으로 write.
    * DBWn은 버퍼를 기록하여 Incremental 체크포인트를 전진시킴. 이 로그 위치는 버퍼 캐시의 가장 오래된 더티 버퍼에 의해 결정.

```
SCN (System Change Number)
: 트랜잭션이 커밋될 때 마다 순차적으로 증가. DB의 커밋 버전.
: 데이터베이스의 모든 컨트롤파일, 데이터파일, redo레코드의 헤더 정보에 동일하게 저장됨

```

#### LGWR(LoG WRiter)

* redo 로그 버퍼 항목을 디스크에 wrtie하여 redo 로그 버퍼를 관리. 마지막으로 write된 이후 버퍼로 복사된 모든 redo 항목을 write.
* LGWR 동작 경우
    * User Process가 트랜잭션을 커밋할 때
    * redo 로그 버퍼가 1/3 찼을 때
    * DBWn 프로세스가 수정된 버퍼를 디스크에 기록하기 전에
    * 3초마다

#### CKPT(ChecKPoinT)

* DB의 redo 스레드에 SCN을 정의하는 데이터 구조.
* 체크포인트가 발생할 경우 DB는 모든 데이터 파일의 헤더를 갱신하여 체크포인트의 세부 사항을 기록해야 함.
    * CKPT는 블록을 디스크에 기록하지 않지만, DBWn은 항상 블록을 디스크에 기록
* 체크포인트 정보를 기록하는 위치
    * control file
    * 각 데이터 파일 헤더

#### SMON(System MONitor)

* 필요한 경우 instance가 시작될 때 recovery를 수행.
* 더이상 사용하지 않는 임시 세그먼트 정리.

#### PMON(Process MONitor)

* user process가 실패할 경우 프로세스 recovey 수행
    * 데이터베이스 버퍼 캐시 정리
    * suer process에서 사용하는 리소스 해제
* 정기적으로 디스패처와 서버 프로세스 상태 확인하여 실행이 정지된 경우 이를 재시작.
* 리스너에 동적으로 데이터베이스 서비스 등록.

#### RECO(RECOvery)

* 분산 트랜잭션과 관련된 failure를 자동으로 해결.
* RECO는 관련 DB 서버간의 연결을 재설정할 때 모든 in-doubt 트랜잭션을 자동으로 해결하여 각 DB의 보유 트랜잭션 테이블에서 해결된 in-doubt 트랜잭션에 해당하는 모든 행을 제거.

```
In-Doubt 트랜잭션
: local → remote 순차 트랜잭션(커밋, 업데이트 등)이 있을 때 local에서 정상 실행, remote에서 실행되지 못했을 때 그 트랜잭션을 뜻함.

```

#### ARCn(ARChiver process)

* 로그 스위치가 발생한 후에 리두 로그 파일을 지정된 저장 장치로 복사.
* 트랜잭션 리두 데이터를 수집하여 대기 대상으로 전송

#### 프로세스 시작 시퀀스

* Oracle 그리드 Infrastructure는 OS init 데몬에 의해 시작.

### 데이터베이스 저장 영역 구조

* 컨트롤 파일: 데이터베이스 자체에 대한 데이터(물리적 구조), 백업과 관련된 메타 데이터 포함.
* 데이터 파일: 유저, 응용 프로그램 데이터, 메타 데이터, 데이터 딕셔너리
* 온라인 redo 로그 파일: 데이터베이스의 instance recovery 허용.

다음은 추가적인 파일.

* 파라미터 파일: instance 시작 시 어떻게 instance를 구성할지 정의
* .password 파일
* 백업 파일: 데이터베이스 recovery에 사용.
* 아카이브된 리두 로그 파일: instance에 의해 생성되는 데이터 변경에 대한 기록을 지속적으로 포함. 이 파일과 데이터베이스 백업을 사용하면 `손실된 데이터 파일 recovery` 가능.
* trace file: 내부 오류가 감지되면 프로세스가 오류 정보를 trace file에 덤프.
* alert log file: 특수 trace file. 메시지와 오류가 시간순으로 기록.

### 논리적/물리적 데이터베이스 구조

![Image](/assets/20180416/7.png)

#### 테이블스페이스

* 데이터베이스가 생성될 때 `SYSTEM` 테이블스페이스와 `SYSAUX` 테이블스페이스는 자동으로 생성. 여기에는 응용 프로그램의 데이터를 저장하는 데 사용하지 않는 것이 좋다.

##### SYSTEM 테이블스페이스

* 데이터베이스가 열려 있는 경우 항상 온라인 상태.
* 데이터 딕셔너리테이블 등 데이터베이스의 핵심 기능을 지원하는 테이블 저장

##### SYSAUX 테이블스페이스

* SYSTEM 테이블스페이스의 보조.
* 많은 데이터베이스의 구성 요소가 저장되어 있기 때문에 구성 요소가 올바르게 작동하려면 온라인 상태여야 함.

![Image](/assets/20180416/8.png)

```
- 세그먼트는 테이블스페이스 내에 존재
- 세그먼트는 Extent 모음
- Extent는 데이터 블록 모음
- 데이터 블록은 디스크 블록에 매핑

```

* 세그먼트
    * 데이터 세그먼트: 클러스터화 되지 않은 각각의 비인덱스 구성 테이블에는 데이터 세그먼트가 있다.
    * 인덱스 세그먼트: 각 인덱스는 해당 데이터를 모두 저장하는 데이터 세그먼트를 가짐.
    * undo 세그먼트: 각 instance당 하나의 undo 테이블스페이스 생성.
    * 임시 세그먼트: SQL문에서 실행을 완료할 임시 작업 영역이 필요할 때 생성.
* Extent: 단일 할당으로 얻은 일정 수의 연속적인 데이터 블록으로, `특정 유형의 정보를 저장`할 때 사용. 논리적으로는 연속된 데이터지만 RAID 등에 의해 물리적으로 분산되어 있을 수 있다.
* 데이터 블록: 가장 작은 레벨에서, 데이터는 데이터 블록에 저장. 하나의 데이터 블록은 디스크에서 특정 바이트 수의 물리 공간에 해당.

![Image](/assets/20180416/9.png)

### ASM(Automatic Storage Management)

![Image](/assets/20180416/10.png)

* 이식 가능한 고성능 클러스터 파일 시스템
* 오라클 DB 환경 관리
* ACFS으로 응용 프로그램 파일 관리
* 로드가 균형을 이루도록 여러 디스크에 데이터 분산
* Failure 시 데이터 미러링
* 저장 영역 관리 문제 해결

### ASM 저장 영역 구성 요소

![Image](/assets/20180416/11.png)
(잘 모르겠음)

### 상호 작용 과정

```
1. 호스트/데이터베이스 서버에서 Instance 시작
2. 유저가 User Process 생성하는 어플리케이션 시작. 어플리케이션이 서버에 대한 연결 설정 시도.
3. 서버는 해당하는 Oracle Net 서비스 처리기를 포함한 리스너 실행. 리스너는 어플리케이션의 연결 요청 감지, Dedicated Server 프로세스 생성
4. 유저가 DML SQL 실행, 커밋 (ex) 테이블에서 고객 주소 변경, 커밋
5. 서버 프로세스에서 Shared Pool에서 동일한 SQL 문이 포함된 공유 SQL 영역이 있는지 검사. 발견되면 유저의 액세스 권한 확인하고 기존 SQL영역이 명령문 영역에 사용. 없다면 명령문이 구문 분석 및 처리되도록 새 공유 SQL 영역이 명령문에 할당.
6. 서버 프로세스는 실제 데이터 파일 또는 버퍼 캐시에 저장된 값에서 필요한 데이터 값 검색
7. 서버 프로세스가 SGA 데이터 수정. 트랜잭션이 커밋되었으므로 LGWR 프로세스가 즉시 트랜잭션을 리두 로그 파일에 기록. DBWn 프로세스는 가능한 경우 수정된 블록을 디스크에 영구적으로 기록.
8. 트랜잭션이 성공하면 서버 프로세스가 네트워크를 통해 메시지를 어플리케이션으로 보냄. 실패하면 오류 메시지 전송.
9. 전체 과정에서 개입이 필요한 상호아이 되면 다른 백그라운드 프로세스 실행.

```