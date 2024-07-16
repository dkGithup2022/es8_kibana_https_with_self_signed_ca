# Configure https on es and kibana version 8

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

Elasticsearch 8부터는 Kibana와 통신할 때 HTTPS 통신을 하는 것이 강제됩니다. 이제 HTTP로는 Kibana와 통신이 안됍니다.

사실 7버전 이전부터 HTTPS를 활성화하는 것이 항상 베스트 프랙티스였어요. 하지만 강제적이지 않아서 xpack.security를 꺼놓고 내부망에서 운영하는 경우가 있었는데 이제는 그게 안돼요. Elasticsearch에 접속할 수 있어도 Kibana를 Elasticsearch에 연결할 수 없게 되었어요.

게다가 공식 문서를 보면 self-signed CA라는 개념이 나오는데, 개발 위주로 하는 백엔드 개발자 분들은 생소할 수도 있어서 이번 기회에 정리합니다.


## 공식문서 잠깐 보기: Elasticsearch security principles

공식문서를 볼때, 보안 관련 설정은 Configure Elasticsearch 탭을 보면 안나오고 아래에 Secure Elasticstack 부분을 조회해야 확인할 수 있습니다.
위에서부터 읽으면 해매게 되니까 링크
남깁니다 [Secure Elasticstack](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)


### 보안 권장사항 중 중요한 것 - Protect Elasticsearch from public internet traffic

Elasticsearch의 주소와 권한을 공개적으로 열어두지 마세요.

사실 Elasticsearch는 HTTP 프로토콜을 기반으로 통신하지만, 데이터베이스이기 때문에 보호가 필요합니다.

간단히 생각해도, 공개된 Elasticsearch를 포착한 공격자가 검색 쿼리를 반복해서 보내는 것만으로도 Elasticsearch를 다운시킬 수 있습니다. Basic Auth로 인증을 요구하더라도 요청 큐에 쌓이기 때문에 이는 유효한 공격입니다.

이 내용을 언급하는 이유는 아래의 self-signed CA를 적용하고 통신할 클라이언트에서 CA 파일을 공유하더라도 위와 같은 공격이 가능하기 때문입니다.

CA 파일이 공유되지 않더라도 -k 옵션을 통해 요청 자체는 보낼 수 있기 때문에 엔드포인트에 대한 보안은 필요합니다. Secure하지 않지만 요청 자체는 송수신이 되기 때문에 아래의 CA 설정과는 별개로 엔드포인트를 잘 숨겨놔야 보안이 확보된다고 볼 수 있습니다.

##  용어 정리

### 개념

### Self-singed CA

### 필요 작업

## ES, Kibana8 에 https 적용하기 .

### 1. self-signed ca 생성하기 .

### 2. certificate, private key 파일 생성하기 .

### 3. ca 등록

### 4. pem 키 생성 및 등록

### 5. kibana 에 ca 파일 옮기기.

## 부록 : java elasticsearch client 에서  self-signed-ca 적용된 es 와 통신하기.

