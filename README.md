# oracledb-embulk-bigQuery

`oracledb`에 붙어서 `bigQuery`로 이관하는 작업을 수행합니다. data migration은 `embulk`를 사용합니다.

## 시작

가상머신을 준비합니다.

`Google Cloud Platform`에서 `ubuntu 16.0.4 LTS` 버전으로 VM을 생성하여 Hands-On을 진행합니다.

## Oracledb 연결

`ubuntu vm` 에서 oracledb와 연동하기 위해 `Oracle Instant Client`를 설치합니다. 버전은 12.1을 이용합니다.

`oracle-instant-basic` : `gs//oracle-client-bucket/oracle-instantclient12.1-basic-12.1.0.2.0-1.x86_64.rpm`

`oracle-instant-devel` : `gs://oracle-client-bucket/oracle-instantclient12.1-basic-12.1.0.2.0-1.x86_64.rpm`

`oracle-instant-sqlplus` : `gs://oracle-sqlplus/ oracle-instantclient12.1-sqlplus-12.1.0.2.0-1.x86_64.rpm`

다음 환경 설정을 순서대로 진행합니다.

```
$ vi /etc/ld.so.conf.d/oracle.conf && sudo chmod o+r /etc/ld.so.conf.d/oracle.conf
/usr/lib/oracle/12.1/client64/lib/
```

```
$ vi /etc/profile.d/oracle.sh && sudo chmod o+r /etc/profile.d/oracle.sh
export ORACLE_HOME=/usr/lib/oracle/12.1/client64
export TNS_ADMIN=/etc/oracle //tnsnames.ora
```

```
$ vi ~/.bash_profile
PATH=$ORACLE_HOME/bin:$PATH
LD_LIBRARY_PATH=$ORACLE_HOME/lib
export LD_LIBRARY_PATH
export PATH

$ source ~/.bash_profile
```

환경설정 셋팅을 완료하였으면 `$ vi /etc/oracle/tnsnames.ora ` 열어서 oracledb에 접속하기 위해 tns 값을 설정해줍니다.

```
XE=
(DESCRIPTION=
(ADDRESS_LIST=
(ADDRESS = (PROTOCOL = TCP)(HOST=[your-hostName])(PORT=1521))
)
(CONNECT_DATA=
(SERVICE_NAME=[your-serviceName])
)
)
```

`$ sqlplus username/password@dbhost(Ip):port/SID` 로 oracledb에 접속합니다.

## embulk .yaml 파일 작성

embulk를 실행시키기 위한 관련 package들을 설치합니다.

```
$ sudo apt install default-jre
$ curl --create-dirs -o ~/.embulk/bin/embulk -L "https://dl.embulk.org/embulk-latest.jar"
$ chmod +x ~/.embulk/bin/embulk
$ echo 'export PATH="$HOME/.embulk/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc

$ embulk gem install embulk-input-oracle
$ embulk gem install embulk-output-bigquery
```

그리고, embulk를 실질적으로 수행하는데 필요한 `.yaml` 파일을 작성합니다.

```
in:
 type: oracle
 driver_path: /home/gscloud94/ojdbc8.jar
 host: 34.64.254.253
 port : "1521"
 user: h2
 password: "1234"
 database: XE
 query : | SELECT * FROM hr.departments

out:
 type: bigquery
 mode: replace
 auth_method : json_key
 json_keyfile : [ json_keyfile 경로를 적어줍니다. ]
 service_account_email : [bigQuery에 접근하기 위해 IAM>service account 에서 생성한 service_account 등록(단, bigQuery admin권한 설정이 등록 완료된 상태)]
 project: [현재 수행중인 프로젝트 위치]
 dataset: [your-bigQuery-dataSet]
 location : [your-location(region)] e.g)asia-northeast3
 table: [your-bigQuery-table]
 auto_create_table: true
 auto_create_dataset: true
 ignore_unknown_values: true
 allow_quoted_newlines: true
 open_timeout_sec : 7200
 send_timeout_sec : 7200
 read_timeout_sec : 7200
```

## 마이그레이션 수행 시 발생할 수 있는 이슈 모음

`sqlplus64: error while loading shared libraries: libsqlplus.so: cannot open shared object file: No such file or directory` : `$ sudo apt-get install libaio1 libaio-dev`

`ORA-01882: timezone region not found ` :

```
sudo java -jar ojdbc8.jar configure -Doracle.jdbc.timezoneAsRegion=false
sudo timedatectl set-timezone Asia/Seoul
```
