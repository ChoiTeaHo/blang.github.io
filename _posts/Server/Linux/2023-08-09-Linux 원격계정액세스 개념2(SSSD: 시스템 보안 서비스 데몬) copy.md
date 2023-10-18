---
title:  "원격계정액세스 (SSSD: 시스템 보안 서비스 데몬)"
excerpt: "원격계정액세스 (SSSD: 시스템 보안 서비스 데몬) 입니다."

categories:
  - linux
tags:
  - [Linux, Ubuntu, Raspbian]

toc: true
toc_sticky: true

last_modified_at: 2023-08-09T20:00:00-05:00
---

## 요약
특정디렉토리에 걸린 소유자가 etc 디렉토리 내부의 passwd, group 안에 없는 경우 SSSD 라는 개념이 있다는것을 알게되었다.  

> ❗원격 서비스(NSS, SSSD) 중에서 하나의 방법이다.  
> 💡 참고)  
> [https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/7/html/system-level_authentication_guide/index](https://access.redhat.com/documentation/ko-kr/red_hat_enterprise_linux/7/html/system-level_authentication_guide/index)


## SSSD(시스템 보안 서비스 데몬)  
- ***SSD(시스템 보안 서비스 데몬)***는 원격 디렉터리 및 인증 메커니즘에 액세스하기 위한 시스템 서비스. 
- 로컬 시스템(SSSD 클라이언트)을 외부 백엔드 시스템(프로바이더)에 연결한다. 
- 이렇게 하면 SSSD 클라이언트가 SSSD 공급자를 사용하여 ID 및 인증 원격 서비스에 액세스할 수 있다. (원격 서비스 예시 아래참고.)
    - LDAP 디렉터리
    - IdM(Identity Management)
    - AD(Active Directory) 도메인 
    - Kerberos 영역



## SSSD 동작
1. 클라이언트를 ID 저장소에 연결하여 인증 정보를 검색한다.  
2. 가져온 인증 정보를 사용하여 클라이언트에서 사용자 및 자격 증명의 로컬 캐시를 생성한다.  
3. 그런 다음 로컬 시스템의 사용자는 외부 백엔드 시스템에 저장된 사용자 계정을 사용하여 인증할 수 있다.  
  
> ❗***특징***  
> 💡 SSSD는 로컬 시스템에서 사용자 계정을 생성하지 않는다.  
> 💡 대신 외부 데이터 저장소의 ID를 사용하고 사용자가 로컬 시스템에 액세스할 수 있도록 한다.  
> 💡 SSSD는 NSS(Name Service Switch) 또는 PAM(Pluggable Authentication Modules)과 같은 여러 시스템 서비스에 대한 캐시를 제공할 수도 있다.  



## SSSD 장점
1. ID 및 인증 서버에 대한 로드 감소
    - 정보를 요청할 때 SSSD 클라이언트는 캐시를 확인하는 SSSD에 문의한다. SSSD는 캐시에서 정보를 사용할 수 없는 경우에만 서버에 연결한다.

2. 오프라인 인증
    - SSSD는 선택적으로 원격 서비스에서 검색된 사용자 ID 및 자격 증명의 캐시를 유지한다. 
    - 이 설정에서는 원격 서버 또는 SSSD 클라이언트가 오프라인 상태인 경우에도 사용자가 리소스에 성공적으로 인증할 수 있다. 

3. 단일 사용자 계정(인증 프로세스의 일관성 향상)  
    - SSSD에서는 오프라인 인증을 위해 중앙 계정과 로컬 사용자 계정을 모두 유지 관리할 필요가 없다.
    - 원격 사용자에게는 종종 여러 사용자 계정이 있다. 예를 들어 VPN(가상 사설 네트워크)에 연결하려면 원격 사용자는 로컬 시스템을 위한 하나의 계정과 VPN 시스템의 다른 계정을 보유한다.
    - 원격 사용자는 캐싱 및 오프라인 인증으로 간단하게 로컬 시스템에 인증하여 네트워크 리소스에 연결할 수 있다. ***그런 다음 SSSD에서 네트워크 자격 증명을 유지한다.***



## 서비스 구성: NSS
> ❗***SSSD가 NSS로 작동하는 방법***  
> 💡 NSS(Name Service Switch) 서비스는 구성 소스와 함께 시스템 ID 및 서비스를 매핑한다.  
> 💡 여기에서는 서비스가 다양한 구성 및 이름 확인 메커니즘의 소스를 검색할 수 있는 중앙 구성 저장소를 제공한다.  
> 💡 SSSD는 NSS 맵의 공급자로 NSS를 사용할 수 있다.  
>   
> ❗***가장 중요한 내용***  
> 💡 사용자 정보( passwd 맵)  
> 💡 그룹( 그룹 맵)  
> 💡 netgroups( netgroups 맵)  
> 💡 서비스( 서비스 맵)  
  

    
## SSSD 설치

```
yum install sssd
```
  
## SSSD를 사용하도록 NSS 서비스 구성
### 1. authconfig 유틸리티를 사용하여 SSSD를 활성화 한다.

```bash
authconfig --enablesssd --update
```

위처럼 하면 /etc/nsswitch.conf 파일이 업데이트되어 아래 NSS 맵에서 SSSD를 사용할 수 있다.

```bash
vi /etc/nsswitch.conf 

passwd: files sss 
shadow: files sss 
group: files sss 

netgroup: files sss
```

### 2. sss 를 서비스 맵 줄에 추가

```bash
services: files sss
```


## NSS로 작동하도록 SSSD 구성
> ❗***나의 경우***  
> 💡 나의 투입환경에는 sssd 가 설치되어 있는데도 /etc/sssd 디렉토리가 보이지 않았다.
>   
> ```bash
> # 설치확인
> sudo yum list installed | grep sssd  #(CentOS, RHEL) 환경
> 
> # 설치위치 확인
> which sssd  #결과는 no sssd in...
> 
> ```
  
일단.. 나의 환경문제는 제외하고 기본적으로 사용 가이드를 작성하겠다.  
{: .notice--info}

  
  
### 1. [sssd] 섹션에서 NSS가 SSSD와 함께 작동하는 서비스 중 하나로 나열되어 있는지 확인

```bash
vi /etc/sssd/sssd.conf

[sssd] 
[... file truncated ...] 
services = nss, pam     # 이 줄의 내용처럼  'nss' 부분이 나열되어 있어야 한다.

```


### 2. [nss] 섹션에서 SSSD가 NSS와 상호 작용하는 방법을 구성

```bash
vi /etc/sssd/sssd.conf

# 이 사용가능한 옵션 설정가이드는 별도 리서치 필요.
[nss] 
filter_groups = root 
filter_users = root 
entry_cache_timeout = 300 
entry_cache_nowait_percentage = 75

```

### 3. SSSD 재기동

```bash
systemctl restart sssd.service

````


## [최종] 통합 작동 테스트

```bash
# 1. 사용자로 로그인. 
# 2. sssctl user-checks user_name auth 명령을 사용하여 SSSD 구성을 확인
sssctl user-checks user_name auth

```

