# OpenSSL로 사설 인증서 생성하기

[설치환경](#설치-환경)
[OpenSSL 설치](#openssl-설치)
[서버 인증서 생성](#서버-인증서-생성)
    - [개인키 생성](#1.-개인키-생성)
    - [CSR생성](#2.-CSR(Certificate-Signing-Request)-생성)
    - [CRT생성](#3.-CRT(Certificate)-생성)
[인증서 변환](#인증서-변환)
[기타](#기타)


## 설치 환경
1. CentOS release 6.10(Final)
2. OpenSSL 1.0.1e-fips 11 Feb 2013


## OpeenSSL 설치
1. yum install 명령어를 사용해서 openssl을 설치한다.
~~~
# yum install -y openssl*
~~~
2. 설치 후 openssl 버전을 확인한다.
~~~
# rpm -qa openssl
~~~
or
~~~
# openssl version
~~~


## 서버 인증서 생성
### 1. 개인키 생성
: 비밀번호가 없는 개인키를 생성한다.
~~~
# openssl genrsa -out {file name}.pem 2048
~~~
ex) openssl genrsa -out private.pem 2048

Generating RSA private key, 2048 bit long modulus  
...........................................................+++  
..............................................................  .............................................................+++  
e is 65537 (0x10001)


### 2. CSR(Certificate Signing Request) 생성
> CSR?
> SSL 인증 정보를 암호화하여 인증기관으로 부터 인증서를 발급받기 위한 신청서이다.
다음 명령어를 입력하여 CSR을 생성한다.
~~~
# openssl req -new -key {private key name} -out {csr file name}.csr [-config {cert config file}]
~~~
ex) openssl req -new -key private.pem -out private.csr

|구분|내용|예|
|---|---|---|
|Country Name|국가코드|KR|
|State or Province Name|시/도의 전체이름|Seoul|
|Locality Name|시/군/구 등의 이름|Mapo-gu|
|Organization Name|회사이름|Test|
|Organizational Unit Name|부서명|Test|
|Common Name|SSL인증서를 설치할 서버의 도메인|www.test.com|
|Email Address|이메일 주소|test@test.com|

__Common Name 에는 도메인의 이름만 넣어야 한다.(IP 주소, 포트번호, 경로명, http:// 나 https:// 포함불가능)__

You are about to be asked to enter information that will be incorporated  
into your certificate request.  
What you are about to enter is what is called a Distinguished Name or a DN.  
There are quite a few fields but you can leave some blank  
For some fields there will be a default value,  
If you enter '.', the field will be left blank.  

Country Name (2 letter code) [XX]:KR  
State or Province Name (full name) []:Seoul  
Locality Name (eg, city) [Default City]:Mapo-gu  
Organization Name (eg, company) [Default Company Ltd]:Test  
Organizational Unit Name (eg, section) []:Test  
Common Name (eg, your name or your server's hostname) []:www.test.com  
Email Address []:test@test.com  

Please enter the following 'extra' attributes  
to be sent with your certificate request  
A challenge password []:test  
An optional company name []:test  


### 3. CRT(Certificate) 생성
#### 사설 CA에서 인증받은 인증서 생성
###### 3-1. rootCA.key 생성하기
~~~
# openssl genrsa -{Encryption Algorithm} -out {rootCA key name}.key 2048
~~~
ex) openssl genrsa -aes256 -out rootCA.key 2048

Generating RSA private key, 2048 bit long modulus  
.+++  
.............................................+++  
e is 65537 (0x10001)  
Enter pass phrase for rootCA.key:  
Verifying - Enter pass phrase for rootCA.key:  

###### 3-2. rootCA 사설 CSR 생성하기
~~~
# openssl req -x509 -new -nodes -key {rootCA key name}.key -days {period} -out {rootCA CSR name}.pem [-config{cert config file}]
~~~
ex) openssl req -x509 -new -nodes -key rootCA.key -days 3650 -out rootCA.pem

Enter pass phrase for rootCA.key:  
You are about to be asked to enter information that will be incorporated  
into your certificate request.  
What you are about to enter is what is called a Distinguished Name or a DN.  
There are quite a few fields but you can leave some blank  
For some fields there will be a default value,  
If you enter '.', the field will be left blank.  

Country Name (2 letter code) [XX]:KR  
State or Province Name (full name) []:Seoul  
Locality Name (eg, city) [Default City]:Mapo-gu  
Organization Name (eg, company) [Default Company Ltd]:Test  
Organizational Unit Name (eg, section) []:Test  
Common Name (eg, your name or your server's hostname) []:www.test.com  
Email Address []:test@test.com  

###### 3-3. CRT 생성하기
: [2. CSR생성](#2.-CSR(Certificate-Signing-Request)-생성) 에서 만든 csr을 rootCA인증을 받아 cert.cst로 생성한다.
~~~
# openssl x509 -req -in {private csr name}.csr -CA {rootCA csr name}.pem -CAkey {rootCA key}.key -CAcreateserial -out {cert name}.crt -days {period} [-config {cert config file}]
~~~
ex) openssl x509 -req -in private.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out cert.crt -days 3650

Signature ok  
subject=/C=KR/ST=Seoul/L=Mapo-gu/O=Test/OU=Test/CN=www.test.com/emailAddress=test@test.com  
Getting CA Private Key  
Enter pass phrase for rootCA.key:  

#### 단순히 개인키로 인증서 생성
~~~
# openssl req -x509 -days {period} -key {private key file} -in {csr file name} -out {cert name}.crt -days {period}
~~~
ex) openssl req -x509 -days 3650 -key private.pem -in private.csr -out cert.crt -days 3650


## 인증서 변환
### pkcs12 생성
~~~
openssl pkcs12 -export -in {crt file} -inkey {private key file} -out {p12 file name}.p12
~~~
ex) openssl pkcs12 -export -in cert.crt -inkey private.pem -out private.p12

### keystore 생성
~~~
openssl pkcs12 -export -in {crt file} -inkey {private key file} -out {keystore file name}.keystore -name {alias}
~~~
ex) openssl pkcs12 -export -in cert.crt -inkey private.pem -out private.keystore -name tomcat

### jks 생성
~~~
keytool -importkeystore -srckeystore {p12 file} -srcstoretype PKCS12 -destkeystore {jks file name}.jks -deststoretype JKS
~~~
ex) keytool -importkeystore -srckeystore private.p12 -srcstoretype PKCS12 -destkeystore abc.jks -deststoretype JKS  

__keytool을 사용하기 위해서는 java가 설치되어 있어야 한다.__

키 저장소 private.p12을(를) abc.jks(으)로 임포트하는 중...  
대상 키 저장소 비밀번호 입력: (jks에 사용할 비밀번호)  
새 비밀번호 다시 입력:  
소스 키 저장소 비밀번호 입력: (p12파일 생성시 입력한 비밀번호)  
1 별칭에 대한 항목이 성공적으로 임포트되었습니다.  
임포트 명령 완료: 성공적으로 임포트된 항목은 1개, 실패하거나 취소된 항목은 0개입니다.  

### crt -> pem
~~~
openssl x509 -in {crt file} -out {converted file name}.pem -outform PEM
~~~
ex) openssl x509 -in cert.crt -out cert.pem -outform PEM

## 기타
### 인증서 내용 보기
~~~
openssl x509 -text -noout -in {crt file name}
~~~
ex) openssl x509 -text -noout -in cert.crt

### jks 확인
~~~
keytool -list -v -keystore {jks file name}.jks
~~~
ex) keytool -list -v -keystore abc.jks