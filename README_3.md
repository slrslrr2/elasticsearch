## Logstash 설치 및 실행

- Logstash 소개
  - logstash 파이프라인 구조 설명
  - Support Matrix 페이지에서 설치 가능한 환경 확인
- Logstash 설치
  - Logstash 다운로드
  - logstash 실행 가능 옵션 설명 : -e | -f
- Logstash 실행
  - Input {} output {} 입/출력 파이프라인 설정
  - 입력 : stdin 표준 입력, tcp 네트워크 입력
  - 출력 : stdout 표준 출력, elasticsearch 출력
  - Elasticsearch 에 데이터 입력된 것 확인



## Logstash란?

아래 그림처럼 **Beats**와 함께 로그 수집부분을 담당하고있다.

Logstash와 Beats의 차이점은 **Logstash**의 경우 파이프라인 기능을 통해 들어오는 **데이터를 다양하게 가공**할 수 있다.<br>**Beats**의 경우 **Logstash 보다 데이터를 가볍게 수집**한다고 생각하면 된다.

<img width="1014" alt="image-20220731161303005" src="https://user-images.githubusercontent.com/58017318/182023562-5d2f3885-5cca-4735-9a51-3e463b731c6f.png">


**Logstash의 파이프라인 구조**는 아래와 같다.

<img width="1251" alt="image-20220731161625196" src="https://user-images.githubusercontent.com/58017318/182023566-0c4ee0e4-9313-427b-8992-cf84abfc1ade.png">

1. Inputs : Logstash가 수집할 경로를 명시
2. Filter : 들어온 데이터를 어떻게 바꿀것인지 명시
3. Output: Logstash가 보낼 경로를 명시

-----

## Logstash 실행

Logstash의 경우 실행할 때, 꼭 파이프라인을 명시해주어야한다.

- Mac, Unix & Linux

  ```
  bin/logstash [options]
  ```

- Windows

  ```
  bin/logstash.bat [options]
  ```

- options

  ```
  input {}
  filter {}
  output {}
  ```



1. 실행하면서 파이프라인 적는방법
   - -e : Set Configurations in command line

```console
# Mac, Unix & Linux
bin/logstash -e 'input { stdin {} } output { stdout {} }'

# windows
bin/logstash.bat -e 'input { stdin {} } output { stdout {} }'
```



2. pipeline.conf 파일을 실행하는 방법

   - -f : If configurations are set in a file (ex. Pipeline.conf)

   - 

     ```
     bin/logstash -f pipeline.conf
     ```

   - pipeline.conf

     ```
     input {
     	tcp {
     		port => 9900 #9900 포트로 오는 데이터들을 수집한다.
     	}
     }
     
     output {
     	# stdout {}
     	elasticsearch {
     		hosts => ["localhost:9200"] # 9200 elasticsearch로 output한다.
     	}
     }
     ```

   - 결과

     - 9200 elasticsearch로 받을 수 있으므로 아래과 같이 명령어를 날린다.

     - ```
       echo 'Hello Logstash' | nc localhost 9900
       ```

     - 그럼 kibana에서 결과 확인해보면 아래와 같다

     - <img width="726" alt="image-20220731185938195" src="https://user-images.githubusercontent.com/58017318/182023568-0c44b428-d29e-44b2-b23d-d21578d524f2.png">
       - 그림과 같이 logstash~~라는 index가 생성되었고 
       - message라는 필드로 전송된 값이 들어갔다.

> https://www.youtube.com/watch?v=E3CSlX--6Cc&list=PLhFRZgJc2afp0gaUnQf68kJHPXLG16YCf&index=23

---

# Logstash filter 설정

- Config 변경시 logstash 자동 재시작 설정
  - config/logstash.yml 에서 config.reload.automatic: true로 바꾸면 logstash를 재시작 안해도된다.
- 아파치 웹 로그 수집을 위한 filter 설정
  - grok : 메시지 스트림 파싱
  - geoip : IP주소에서 위치 및 지역정보 확장
  - useragent : 클라이언트 OS 및 브라우저 정보 확장
  - mutate : 불필요한 필드 삭제 및 타입 변환
  - date : 문자열로 된 날짜를 date 타입으로 변환



## 아파치 웹 로그 수집을 위한 filter 설정

### 1. 환경설정

아래 경로로 들어가 weblog-sample.log파일을 다운받는다.

> https://docs.google.com/document/d/1XJsI2mHlihP8sOKJRn_MgMY4STovOnd4IlaAkwyxbys/edit



**weblog-sample.log** 파일을 head명령어를 이용해 1줄만 읽어들인다.

```
head -n 1 ./weblog-sample.log

14.49.42.25 - - [12/May/2022:01:24:44 +0000] "GET /articles/ppp-over-ssh/ HTTP/1.1" 200 18586 "-" "Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.2b1) Gecko/20091014 Firefox/3.6b1 GTB5"
```



9900 포트로 읽어들여 elastic에 전송한다.

```
head -n 1 ./weblog-sample.log | nc localhost 9900
```

위 명령어를 실행하면 통으로 메시지가 들어간다.

<img width="809" alt="image-20220731192929104" src="https://user-images.githubusercontent.com/58017318/182023569-19e698da-93b2-4272-b33d-10146c16d3b7.png">
위 데이터는 **message**라는 field로 들어가기때문에 각 형태에 맞는 fields로 들어가도록 설정해보자.

그때 사용되는 필터는 **grok**이다.

apache 등 자주 사용되는 log는 형식이 지정되어있기때문에 pattern으로 지정되어있다.

> https://github.com/logstash-plugins/logstash-patterns-core/blob/main/patterns/legacy/httpd

위 내용을 가보면

<img width="1197" alt="image-20220731194247091" src="https://user-images.githubusercontent.com/58017318/182023572-cad898a5-1d76-409d-a2e6-6b0c7902f04c.png">


하여, Apache로그의 경우 **COMBINEDAPACHELOG**를 사용하면된다.

```json
filter {
	grok {
		match => { "message" => "%{COMBINEDAPACHELOG}" }
	}
}
```



해당 파일 설정 후 `elastic % head -n 1 ./weblog-sample.log | nc localhost 9900`를 날리면 각 field에 맞게 데이터가 들어간다.

```
14.49.42.25 - - [12/May/2022:01:24:44 +0000] "GET /articles/ppp-over-ssh/ HTTP/1.1" 200 18586 "-" "Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.2b1) Gecko/20091014 Firefox/3.6b1 GTB5"
```

```
{
  "@timestamp" => 2022-07-31T10:59:17.923Z,
  "timestamp" => "12/May/2022:01:24:44 +0000",
  "bytes" => "18586",
  "response" => "200",
  "agent" => "\"Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.2b1) Gecko/20091014 Firefox/3.6b1 GTB5\"",
  "host" => "localhost",
  "auth" => "-",
  "port" => 49429,
  "verb" => "GET",
  "@version" => "1",
  "request" => "/articles/ppp-over-ssh/",
  "message" => "14.49.42.25 - - [12/May/2022:01:24:44 +0000] \"GET /articles/ppp-over-ssh/ HTTP/1.1\" 200 18586 \"-\" \"Mozilla/5.0 (Windows; U; Windows NT 6.1; en-US; rv:1.9.2b1) Gecko/20091014 Firefox/3.6b1 GTB5\"",
  "httpversion" => "1.1",
  "clientip" => "14.49.42.25",
  "referrer" => "\"-\"",
  "ident" => "-"
}
```



그 외에도 아래 다양한 옵션을 적용할 수 있다.











