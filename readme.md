# Configure https on es and kibana version 8

/*
1. TODO: https 관련 설명과 인증서 파일 포맷은 설명 보충 하기. 
2. 전체 k8s 마니패스트도 정리해서 공유해주기 
2. **/
## 목차

- [소개](#소개)
- [공식문서 잠깐 보기: Elasticsearch security principles](#공식문서-잠깐-보기-elasticsearch-security-principles)
    - [원칙 - Protect Elasticsearch from public internet traffic](#원칙---protect-elasticsearch-from-public-internet-traffic)
- [Intorouce self-signed CA](#intorouce-self-signed-ca)
    - [개념](#개념)
        - [용어 정리](#용어-정리)
    - [필요 작업](#필요-작업)
- [ES, Kibana8 에 https 적용하기](#es-kibana8-에-https-적용하기)
    - [1. self-signed ca 생성하기](#1-self-signed-ca-생성하기)
    - [2. certificate, private key 파일 생성하기](#2-certificate-private-key-파일-생성하기)
    - [3. ca 등록](#3-ca-등록)
    - [4. pem 키 생성 및 등록](#4-pem-키-생성-및-등록)
    - [5. kibana 에 ca 파일 옮기기](#5-kibana-에-ca-파일-옮기기)
- [부록 : java elasticsearch client 에서 self-signed-ca 적용된 es 와 통신하기](#부록--java-elasticsearch-client-에서-self-signed-ca-적용된-es-와-통신하기)

---

## 소개

이 글에서는 Elasticsearch 8 버전에서 HTTPS를 활성화하고 Kibana와 연결하는 방법과 k8s 환경에서 적용한 예시를 다룹니다.

Elasticsearch 8부터는 Kibana와 통신할 때 HTTPS 통신을 하는 것이 강제됩니다. 이제 HTTP로는 Kibana와 통신이 안됩니다.

사실 7버전 이전부터 HTTPS를 활성화하는 것이 항상 베스트 프랙티스였어요. 하지만 강제적이지 않아서 xpack.security를 꺼놓고 내부망에서 운영하는 경우가 있었는데 이제는 그게 안돼요. Elasticsearch에 접속할 수 있어도 Kibana를 Elasticsearch에 연결할 수 없게 되었어요.

게다가 공식 문서를 보면 self-signed CA라는 개념이 나오는데, 개발 위주로 하는 백엔드 개발자 분들은 생소할 수도 있어서 이번 기회에 정리합니다.

---

## 공식문서 잠깐 보기: Secure Elasticstack

공식문서를 볼때, 보안 관련 설정은 Configure Elasticsearch 탭을 보면 안나오고 아래에 Secure Elasticstack 부분을 조회해야 확인할 수 있습니다.
위에서부터 읽으면 해매게 되니까 링크
남깁니다 [Secure Elasticstack](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)


### 보안 권장사항 중 중요한 것 - Protect Elasticsearch from public internet traffic

Elasticsearch의 주소와 권한을 공개적으로 열어두지 마세요.

사실 Elasticsearch는 HTTP 프로토콜을 기반으로 통신하지만, 데이터베이스이기 때문에 보호가 필요합니다.

간단히 생각해도, 공개된 Elasticsearch를 포착한 공격자가 검색 쿼리를 반복해서 보내는 것만으로도 Elasticsearch를 다운시킬 수 있습니다. Basic Auth로 인증을 요구하더라도 요청 큐에 쌓이기 때문에 이는 유효한 공격입니다.

이 내용을 언급하는 이유는 아래의 self-signed CA를 적용하고 통신할 클라이언트에서 CA 파일을 공유하더라도 위와 같은 공격이 가능하기 때문입니다.

CA 파일이 공유되지 않더라도 -k 옵션을 통해 요청 자체는 보낼 수 있기 때문에 엔드포인트에 대한 보안은 필요합니다. Secure하지 않지만 요청 자체는 송수신이 되기 때문에 아래의 CA 설정과는 별개로 엔드포인트를 잘 숨겨놔야 보안이 확보된다고 볼 수 있습니다.

이 내용에 대한 정확한 예시는  아래의 [부록 : java elasticsearch client 에서 self-signed-ca 적용된 es 와 통신하기](#부록--java-elasticsearch-client-에서-self-signed-ca-적용된-es-와-통신하기) 에서 다루도록 하겠습니다.

---

##  https 관련 파일의 개념과 확장자

#### 1. CA :Certificate Authority
- https 통신 중 인증 기관의 역할
- https 통신 과정은 해당 링크 참고 ( [https 설명링크](https://www.semrush.com/blog/what-is-https/) )

#### 2. http 인증서

-  https 통신을 위해 클라이언트에게 전송되는 ca 인증서.
- .crt 또는 .cer 확장자를 가지고 있다 .


#### 3.  pem 파일

- 신뢰할 수 있는 CA의 인증서를 pem 파일 형식으로 가지고 있는 것 

#### 4. PKCS#12 

- 인증서나 공개키를 암호화해서 파일로 전송할때 쓰는 기술 & 포멧 중 하나를 PKCS#12 라고 한다.
- 아래의 예시에선 .p12  확장자를 사용한다.


### Self-singed CA

클라이언트 입장에서는 self-signed ca 를 이용한 https 통신은 해당 ca 가 유효한 ca 인지 알 수 없습니다.
따라서 통신 이전에 https 통신의 암호화를 위해서 ca 관련 정보를 클라이언트 사이드로 미리 전송해주는 단계가 필요합니다.

일반 웹 개발에서는 google 이나 cloud flare 처럼 잘 알려진 ca 를 통해 https 의 커넥션의 보안이 보장되기 때문에 이런일이 일어나지 않아요.

하지만 self-signed ca 에서는 kibana 와 통신할때 생성된 ca 스펙에 관련된 파일을 kibana 의 로컬로 집어 넣어줘야합니다. 마찬가지로 java elasticsearch api client를 쓰는 Jvm application 에서는 jvm 에 직접 ca 를 등록을 해줘서 
https 통신을 보장하는 jvm 스택이 정상 동작하도록 설정이 필요해요. 


---


# 필요 작업

## ES, Kibana8 에 https 적용하기 .

### 1. self-signed ca 생성하기 .

- es 의 노드에서 elasticsearch/bin 에서 아래의 커맨드를 수행합니다.
```shell
bin/elasticsearch-certutil ca
```
1.  Generate the certificate authority
2.  자체 인증된 ca 를 등록하는 커맨드입니다.

   1. 경로, password 를 지정하라는 ui 가 나오는데, 둘 다 빈칸이여도 상관 없습니다.
   2. password 를 지정했다면 이 링크에서 설명하는 커맨드로 비밀번호를 등록하세요
        1. 링크: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup.html
   3. 경로를 지정하지 않았다면 elasticsearch/elastic-stack-ca.p12 경로에 ca 파일이 생성됩니다.

### 2. certificate, private key 파일 생성하기 .

```shell
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```

커맨드를 실행하면 "elastic-certificates.p12" 파일이 생성됩니다. 이 키스토어 파일은 node certificate , ca certificate 가 포함되어 있습니다

### 3. elasticsearch.yml에  ca 위치 등록 

ca의 위치를 등록하는 것을 포함한 ssl 에 필요한 설정입니다. 

```yml
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate 
xpack.security.transport.ssl.client_authentication: required
xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
```

파일의 경로를 config/certs 로 이동 후, 아래와 같은 yml 을 config 파일로 만들어둡시다.

- keystore path 를 읽는 경로의 시작이 elasticsearch/config 임에 주의 합니다.
- 이 설정은 자체 생성한 ca 를 사용하기 위한 설정일 뿐, 아직 https 통신할 수 없습니다. https 통신을 위해선 https 를 위한 pem 키값을 만들어 클라이언트와 공유해야 합니다.

### 4. pem 키 생성 및 등록

```shell
./bin/elasticsearch-certutil http
```

- https 통신을 위한 pem 키 생성을 위해 아래의 커맨드를 실행합니다.
    - want to generate a CSR, enter `n`.
        - 본 예시에선 public 를 사용하지 않습니다. no
    - want to use an existing CA, enter `y`.
        - 위에 certutil로 등록된 ca 를 사용합니다. yes
    - Enter the path to your CA.
        - 어떤 ca 에 대한 pem 파일을 만들건지 물어보는 겁니다. elastic-stack-ca.p12 의 절대 경로로 등록해줍니다.


- 위의 작업을 마치면 `elasticsearch-ssl-http.zip` 파일이 생성됩니다. unzip 합시다.

```
$ unzip elasticsearch-ssl-http.zip
```

```
/elasticsearch
|_ README.txt
|_ http.p12
|_ sample-elasticsearch.yml
```

```
/kibana
|_ README.txt
|_ elasticsearch-ca.pem
|_ sample-kibana.ym
```

### 5. kibana 에 ca 파일 옮기기.

1. 위의 elasticsearch-ssl-http.zip을 통해 만들어진  elasticsearch-ca.pem 파일을 kibana 의 노드로 옮깁니다.
2. kibana.yml 에서 해당 파일의 위치를 elasticsearch.ssl.certificateAuthorities 경로에 등록해줍니다 .

아래는 kibana 로 파일을 옮긴 마니페스트의 예시입니다 .

```shell
    server:
      name: kibana
      host: "0.0.0.0"
      publicBaseUrl: https://xxx.xxxx/
    elasticsearch:
      username: kibana_system
      password: XXX
      hosts: ["https://xxx.xxxx"]
      ssl:
        certificateAuthorities: ["/usr/share/kibana/config/elasticsearch-ca.pem"]
        verificationMode: certificate
```


## 부록 : java elasticsearch client 에서  self-signed-ca 적용된 es 와 통신하기.

