---
title:  "톰캣 root 계정이 아닌 계정으로 실행하기"
excerpt: "톰캣 root 계정이 아닌 계정으로 실행하기 입니다."

categories:
  - linux
tags:
  - [Linux, Ubuntu, Raspbian]

toc: true
toc_sticky: true

last_modified_at: 2023-08-20T20:00:00-05:00
---

## 개요
root 계정으로 운영중인 WAS(톰캣)를 다른 계정으로 실행해야하는 상황이 발생했다.


## STEP1. 톰캣 전용 계정의 그룹정보 생성
### 1. 계정 확인
```bash
#현재 기본적인 상태
id mfx000
uid=1001(mfx000) gid=1001(mfx000) groups=1001(mfx000)

```

### 2. 그룹 생성
```bash
sudo groupadd grmfx
sudo groupadd wheel

```

### 3. 기타그룹 추가
```bash
sudo usermod -a -G grmfx, wheel

# 확인 (자신 이외의 새로운 그룹 2개 안에도 포함시켰다.)
id mfx000
uid=1001(mfx000) gid=1001(mfx000) groups=1001(mfx000), 1002(grmfx), 1003(wheel)

```

> ❗정리  
> 💡 mfx000 계정으로 톰캣을 실행하기 위해  
> 💡 mfx000 계정을 grmfx 그룹안에 포함시켰다.  



## STEP2. 아파치톰캣 디렉토리 전체 소유권변경
### 1. chwon 수행
```bash
sudo chown -R root:grmfx apache-tomcat

```

### 2. 확인
그룹의 소유권 권한이 내부 디렉토리 레벨까지 변경되었는지 재차 확인한다. 
- mfx000 계정이 권력을 행사할 수 있다. (즉, sudo를 붙이지 않아도 된다)
- 톰캣의 모든 파일에 대해 소유그룹을 grmfx 으로 지정해뒀기 때문이다.  


### 3. mfx000 으로 톰캣실행 (==> 실패한다.)
```bash
startup.sh # 퍼미션에러 발생

```

> ❗정리  
> 💡 파일들의 권한들을 보면 각각 권한들이 제한적이다.  
> 💡 권한들을 하나씩 해결해보자  



## STEP3. 🎆아파치톰캣 logs 디렉토리 쓰기권한 부여
### 1. sudo chmod -R g+w [아파치경로/logs]
```bash
# 그룹(g)에 쓰기(w)권한 부여
sudo chmod -R g+w logs/

```
  
### 2. 확인
```bash
drwxrwx--- 2 cmadmin grmfx 57344 10월 18 09:31 logs  #정상반영

```

### 3. mfx000 으로 톰캣실행 (==> 성공한다. 하지만 이상하다.)
```bash
# started 는 된다.
startup.sh 

# 안나온다.. 왜지?
ps -ef | grep tomcat 

# 이 명령어로 정말 실행되었는지 확인할 수 있다고 한다.  
# 하지만 안나온다.
w3m http://localhost:8080 

```

> ❗정리   
> 💡 logs 디렉토리말고 권한부여가 필요한 곳이 존재한다는 뜻이다.  



## STEP3. 🎆아파치톰캣 conf 디렉토리 쓰기권한 부여
### 1. sudo chmod -R g+w [아파치경로/logs]
```bash
# 그룹(g)에 읽기(r)와 실행(x)권한 부여
sudo chmod -R g+rx conf/ 

```


### 2. 확인
```bash
drwxr-x--- 2 cmadmin grmfx 57344 10월 18 09:31 conf  #정상반영

```

### 3. mfx000 으로 톰캣실행 (==> 성공한다.)
```bash
# 성공
startup.sh 
w3m http://localhost:8080 

```