---
layout: post
title: Drone env setup
---

# 최종 결과

# **1. 연구의 배경 및 필요성**

- 실내 재난 구조, 대형 건물의 3D 매핑 등 GPS를 활용하지 못하는 실내나 지하공간과 같은 환경에서 위치 파악과 환경 매핑의 필요성 증가
- 라이더 센서와 같은 고가의 장비는 높은 초기 구축 비용과 부피와 무게로 인한 소형 기기 탑재에 어려움
- 센서의 의존도를 낮추고 카메라를 활용한 저비용 솔루션 개발 필요

# **2. 연구 목적**

- 카메라를 활용하여 주변 매핑을 할 수 있는 기술 개발
- SLAM 기술을 활용하여 실시간 상대 위치 데이터를 활용하여 NeRF 를 활용해 매핑을 할 수 있는 효율적인 경로 탐색, 이동

# 3. 연구 내용

- 드론 카메라를 활용하여 실시간으로 이미지 전송
- 전송된 이미지를 바탕으로 SLAM 을 활용하여 실시간 주변 환경 매핑 및 드론의 현재 위치 파악
- 파악된 현재 위치를 토대로 NeRF 를 활용하여 효과적인 주변 환경 매핑을 위한 이동 위치 제안
- 시스템의 실시간성 및 정확도 평가

# 4. 연구 과정

리눅스 컴퓨터를 로컬 환경이라고 함

## 1.  드론 실시간 영상 송출 , 수신

DJI mobile SDK 의 `LiveStreamManager` 를 사용하여서 구현하였다.  `LiveStreamManager` 는 RTMP 기술을 사용하여서 실시간 영상을 송출할 수 있는데 이를 이용하여 로컬 리눅스 컴퓨터 내부에 도커를 이용하여 RTMP 영상을 수신할 수 있는 서버를 구축하였다. 서버는 nginx 를 사용하여서 구축하였다. 

RTMP 는 브로드 캐스트에 주로 사용되는 영상 방식이다. 이때 `LiveStreamManager` 가 RTMP로 송출을 하려면 서버가 꼭 필요하기 때문에 이를 위해서 youtube 나 다른 사이트의 서버를 사용할 수도 있지만 로컬에서 구현을 하는 것이 목적이었기 때문에 도커 내부에 nginx 서버를 구축하였다.

드론으로 부터 서버에 정상적으로 영상을 수신 받기 위해서는 리눅스의 포트와 도커의 포트를 매핑해주어야 한다. 이는 처음 컨테이너를 생성해야 할때 해야하는 작업으로 이번 구현과 같은 경우는 rtmp 의 기본 포트인 1935 포트를 사용하여서 리눅스의 1935포트와 도커의 1935 포트를 매핑해주었다. 아래와 같이 해주면 된다. 

```python
docker run -it --rm -p 1935:1935 -p 8080:8080 nginx-rtmp
```

```python
[외부 RTMP 스트림 소스]
        ↓
        ↓ RTMP 프로토콜
        ↓ (rtmp://server-ip:1935/live/test)
+------------------+
|   Host Machine   |
|   (포트 1935)    |
|       ↓         |
|       ↓ 포트 매핑 (1935:1935)
|  +------------+ |
|  |  Docker    | |
|  |  Container | |
|  |            | |
|  | NGINX-RTMP | |
|  | (포트 1935) | |
|  |            | |
|  +------------+ |
+------------------+
        ↓
        ↓ RTMP 스트림 출력
[시청자/클라이언트]
```

nginx 서버 구현을 위해서는 nginx-rtmp 패키지 설치와 nginx.conf 설정을 해주어야 한다.

## 2. 드론의 실시간 영상을 바탕으로 ORB-SLAM3 사용

드론의 `LiveStreamManager` 로 부터 받아온 영상을 토대로 ORB-SLAM3 을 사용하여 실시간 포인트 클라우드를 생성하고 주변 상황에 대한 실시간 위치를 구성할 수 있도록 하였다. nginx 서버를 구성한 도커 내부에 ORB-SLAM3 을 같이 빌드하였다. 환경은 ubuntu 20.04 에 ROS noetic 버전을 사용하여서 구성하였다. 각 ubuntu  버전마다 맞는 ROS 버전이 있는데, ORB-SLAM3 이 공식적으로 ROS2  humble 은 지원하지 않기 때문에 ubuntu 22.04 가 아닌 ubuntu 20.04 버전을 사용하였다.

ORB-SLAM3 은 다양한 데이터 셋을 활용할 수 있는데, 단안 카메라를 사용할 경우 mono_tum.cc 의 파일을 수정하여서 가져오는 데이터의 유형을 설정해 주어야한다. 아래는 ORB-SLAM3을 사용하는 예시이다. (기본 ORB-SLAM3 에서  EuRoC Dataset 을 사용하는 방법 [https://taeyoung96.github.io/slamtip/ORBSLAM3_build/](https://taeyoung96.github.io/slamtip/ORBSLAM3_build/) 참고)

```python
./Monocular/mono_euroc ../Vocabulary/ORBvoc.txt ./Monocular/EuRoC.yaml /home/dataset ./Monocular/EuRoC_TimeStamps/MH02.txt
```

내가 한 프로젝트에서 mono_tum.cc를 수정하여서 rtmp 라이브 스트림을 바탕으로 작동할 수 있게 하였다. 

(mono_tum.cc 파일을 원래대로 돌려서 예시를 돌리려면  ORB-SLAM3 폴더의 [build.sh](http://build.sh) 를 다시 실행해서 빌드 시켜주어야함)

```python
./Examples/Monocular/mono_tum Vocabulary/ORBvoc.txt ./Examples/Monocular/TUM1.yaml rtmp://165.194.27.212:1935/live/test
```

가장 먼저 영상 설정파일, `PATH_TO_VOCABULARY`  `PATH_TO_SETTINGS_FILE`  livestream 링크 순으로 명령어를 구성하면 된다. 

## 3. ORB-SLAM3 으로 구성한 화면 보기

ORB-SLAM3 를 도커 내부에서 구현하여 실시간 결과물을 보지 못한다는 단점이 있었다. 이를 위하여 novnc 를 사용하였다. ORB-SLAM3 를 구현한 도커 이외에 novnc 를 사용하는 도커 이미지를 pull 하여 가져와서 동시에 컨테이너를 작동시킨다. ORB-SLAM3 을 사용하는 도커의 8080 포트를 로컬머신의 포트와 매핑시켜주어야한다. 8080 포트를 통하여 화면을 가져와서 구현하기 때문이다. 

## 4. 드론 제어

드론제어는 DJI SDK `flightcontroller` class 를 사용하여서 구현하였다. 원래 기존의`flightcontroller`는 비행기의 움직임을 제어할 수 있는 class 이다.  이를 활용하여서 드론의 `takeOff` `landing`  를 제어하였다. 추가로 드론의 움직임은 버튼을 이용하여서 제어할 수 있도록 페이지에 버튼을 추가하여서 구현하였다.

`VirtualStickView.java` 파일을 보면 `SendVirtualStickDataTask` 를 통하여 지속적으로 `FlightControlData` 를 `flightController` 에 전송하는데 이때  `FlightControlData` 의 `pitch` `roll` `yaw`  의 값을 수정하여서 드론이 이동할 수 있도록 하였다. 

## 5. NeRF

NeRF studio 를 사용하여서 씬을 생성한다. NeRF studio 를 이용하여 직접 촬영한 영상을 바탕으로 학습을 시키고 원하는 뷰포인트 만들어서 원하는 시점에서의 영상을 제작한다. 향후 연구에서는 NVIDIA 가 제공하는 Realtime NeRF 를 사용하거나 실시간으로 NeRF 를 구성하는 nerf bridge 를 사용해야할 것이다. ([https://arxiv.org/abs/2305.09761](https://arxiv.org/abs/2305.09761))

# 5. 환경설정 방법

## 참고 명령어

```python
sudo service nginx stop
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl status nginx
ps aux | grep nginx
sudo netstat -tulpn

service nginx start 
service nginx status
cat /var/log/nginx/error.log

#Nginx가 실행되었는지 확인:
ps aux | grep nginx
#설정파일 유효성 확인
nginx -t
#설정파일 수정
vim /etc/nginx/nginx.conf

```

## 에러

### #issue1

docker: Error response from daemon: Ports are not available: exposing port TCP 0.0.0.0:1935 -> 0.0.0.0:0: listen tcp 0.0.0.0:1935: bind: address already in use.

포트가 겹쳐서 나는 문제이다 

```jsx
sudo netstat -tulpn
```

명령어로 1935 포트를 누가 사용하는지 확인하고 도커와 로컬 컴퓨터를 연결하는데에 다른 포트를 쓰거나, 1935 포트를 사용하는 서비스를 죽여준다.

나의 경우 기존 로컬에서 테스트하던 nginx 서버를 계속 돌리고 있어서 포트가 겹치는 에러가 있어서 `sudo systemctl stop nginx` 를 해주었다.

### #issue2

 아래 명령어를 실행했을때

```python
./Examples/Monocular/mono_tum Vocabulary/ORBvoc.txt ./Examples/Monocular/TUM1.yaml rtmp://165.194.27.212:1935/live/test
```

이렇게 실행했는데 아무것도 안나오면 

혹은 아래처럼 에러나거나 하면

```jsx
root@79bd221553fb:/home/ORB_SLAM3# ./Examples/Monocular/mono_tum Vocabulary/ORBvoc.txt ./Examples/Monocular/TUM1.yaml rtmp://165.194.27.212:1935/live/test
[ WARN:0] global /tmp/opencv/modules/videoio/src/cap_gstreamer.cpp (480) isPipelinePlaying OpenCV | GStreamer warning: GStreamer: pipeline have not been created
[ERROR:0] global /tmp/opencv/modules/videoio/src/cap.cpp (140) open VIDEOIO(GSTREAMER): raised OpenCV exception:

OpenCV(4.4.0) /tmp/opencv/modules/videoio/src/cap_gstreamer.cpp:743: error: (-215:Assertion failed) uridecodebin in function 'open'

[ERROR:0] global /tmp/opencv/modules/videoio/src/cap.cpp (140) open VIDEOIO(CV_IMAGES): raised OpenCV exception:

OpenCV(4.4.0) /tmp/opencv/modules/videoio/src/cap_images.cpp:253: error: (-5:Bad argument) CAP_IMAGES: can't find starting number (in the name of file): rtmp://165.194.27.212:1935/live/test in function 'icvExtractPattern'

Could not open RTMP stream: rtmp://165.194.27.212:1935/live/test

```

dji sdk 에서 livestream 서버 껐다 켜보시면 됩니다 

### #issue3

DJI SDK 에서 livestream start 를 했는데 에러나면서 livestream 이 실행되지 않는다면 (status change 가 없으면) nginx 서버가 켜졌는지 확인하자 서버가 있어야 작동하더라

## 전체 프로세스

**리눅스 컴퓨터 설정**

```jsx
docker image pull ahnavocado/orbslam3
```

기존에 만들어 놓은 이미지를 가져온다. 

이미지에는 ubuntu 20.04 환경에 ros noetic, orbslam3 외에 많은 패키지들이 설치되어있다.

```python
docker run -it --net=ros -p 1935:1935 \ 
-v /home/ahn/shared:/shared \
--env="DISPLAY=novnc:0.0" --env="ROS_MASTER_URI=http://roscore:11311" \
ahnavocado/orbslam3 /bin/bash
```

- `—net=ros` → novnc 와 연결하기 위한 설정
- `1935:1935` rtmp 를 위한 포트 연결
- `-v /home/ahn/shared:/shared`  공유 디렉토리, 설정해도 되고 안해도 됨, 대신 하려면 docker desktop 에서 연결시켜주어야함
- `--env="DISPLAY=novnc:0.0" --env="ROS_MASTER_URI=http://roscore:11311"`   잘모르겠다 novnc 연결하는 예시에서 있길래 따라함
- `ahnavocado/orbslam3`  도커 이미지 이름
- `/bin/bash` 도커 실행 후 적용하는 명령어 - 바로 bash 로 접속할 수 있도록 해준다.

이 도커 컨테이너를 컨테이너1 이라고 하자

컨테이너 내부에서 아래 명령어 실행

```python
apt install nginx libnginx-mod-rtmp
apt install ffmpeg 도 해주자...
service nginx start
service nginx status

```

nginx 서버를 위한 코드. 도커에 안넣었다… ㅎ

추가로 /etc/nginx/nginx.conf 에 아래 코드 넣기

- /etc/nginx/nginx.conf
    
    ```jsx
    ahn@ahn-X570:~/workspace/liveStreamServerLocalTest$ cat /etc/nginx/nginx.conf 
    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    include /etc/nginx/modules-enabled/*.conf;
    
    events {
    	worker_connections 768;
    	# multi_accept on;
    }
    
    http {
    
    	##
    	# Basic Settings
    	##
    
    	sendfile on;
    	tcp_nopush on;
    	types_hash_max_size 2048;
    	# server_tokens off;
    
    	# server_names_hash_bucket_size 64;
    	# server_name_in_redirect off;
    
    	include /etc/nginx/mime.types;
    	default_type application/octet-stream;
    
    	##
    	# SSL Settings
    	##
    
    	ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
    	ssl_prefer_server_ciphers on;
    
    	##
    	# Logging Settings
    	##
    
    	access_log /var/log/nginx/access.log;
    	error_log /var/log/nginx/error.log;
    
    	##
    	# Gzip Settings
    	##
    
    	gzip on;
    
    	# gzip_vary on;
    	# gzip_proxied any;
    	# gzip_comp_level 6;
    	# gzip_buffers 16 8k;
    	# gzip_http_version 1.1;
    	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
    
    	##
    	# Virtual Host Configs
    	##
    
    	include /etc/nginx/conf.d/*.conf;
    	include /etc/nginx/sites-enabled/*;
    
    	server {
            	listen 8082;
    
            location /hls {
                types {
                    application/vnd.apple.mpegurl m3u8;
                    video/mp2t ts;
                }
                root /tmp;  # HLS 파일이 저장될 디렉터리
                add_header Cache-Control no-cache;
            }
        }		
    	
    }
    
    #mail {
    #	# See sample authentication script at:
    #	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
    #
    #	# auth_http localhost/auth.php;
    #	# pop3_capabilities "TOP" "USER";
    #	# imap_capabilities "IMAP4rev1" "UIDPLUS";
    #
    #	server {
    #		listen     localhost:110;
    #		protocol   pop3;
    #		proxy      on;
    #	}
    #
    #	server {
    #		listen     localhost:143;
    #		protocol   imap;
    #		proxy      on;
    #	}
    #}
    
    rtmp {
        server {
            listen 1935; 
    	chunk_size 4096;
            application live {
                live on;
                record off;
    
    	    hls on;
    	    hls_path /tmp/hls;
    	    hls_fragment 2s;
            }
        }
    }
    
    ```
    

그 후

```python
service nginx restart
```

해줘서 적용시켜주어야한다. 

**novnc 설정** 

[https://wiki.ros.org/docker/Tutorials/GUI](https://wiki.ros.org/docker/Tutorials/GUI) 참고/

```python
docker network create ros
docker pull theasp/novnc:latest
docker run -d --rm --net=ros \
   --env="DISPLAY_WIDTH=3000" --env="DISPLAY_HEIGHT=1800" --env="RUN_XTERM=no" \
   --name=novnc -p=8080:8080 \
   theasp/novnc:latest
```

새로운 터미널에 위 명령어 실행시켜서 열어둔다 (컨테이너2 라고 명명)

그후 아래 접속

[http://localhost:8080/vnc.html](http://localhost:8080/vnc.html)

잘 되는지 확인하려면 기존 위 `ahnavocado/orbslam3` 를 실행시킨 컨테이너1 에서 

```python
xeyes 
```

실행시켜보고 눈이 잘 나오는지 확인

**드론 설정** 

[https://github.com/avocado-gamja/drone_control](https://github.com/avocado-gamja/drone_control)

위 페이지에서 클론하여 프로젝트 실행 

livestream 만 진행할 예정이면 기본 DJI mobile sdk v4 써도 상관없음 

livestream 페이지에서 혹은 

```python
private String liveShowUrl = "please input your live show url here";
```

이 값에서

RTMP livestream url 을

```python
rtmp://localhost:1935/live/test

rtmp://<로컬 기기의 고정 ip 주소>:1935/live/test
```

와 같이 설정한다. 

그리고 server 가 꼭 켜져있어야 에러가 나지 않는다.

---

컨테이너 1에서 아래 명령어를 통하여서 영상이 잘 들어오는지 확인 

```python
ffmpeg -i rtmp://localhost:1935/live/test -c copy -f segment -segment_time 5 -reset_timestamps 1 -movflags +faststart /home/ahn/workspace/liveStreamData/segment_%03d.mp4 -loglevel debug
```

/home/ahn/workspace/liveStreamData/ 에 영상이 쪼개져서 잘 들어오면 이상없는 거임

컨테이너 1에서 아래 명령어를 통하여 orbslam3 실행 실행 위치는 `/home/ORB_SLAM3` 폴더 내부에서

```python
./Examples/Monocular/mono_tum Vocabulary/ORBvoc.txt ./Examples/Monocular/TUM1.yaml rtmp://165.194.27.212:1935/live/test
```

[http://localhost:8080/vnc.html](http://localhost:8080/vnc.html) 에서 확인이 가능할 것이다.

# 6. 향후 연구

Tello 의 python SDK 를 활용하여 orb slam3 를 사용하고 이를 통하여 카메라로만 경로 결정 및 제어

[https://www.youtube.com/watch?v=2eU0rLp464s](https://www.youtube.com/watch?v=2eU0rLp464s)

https://www.youtube.com/watch?v=A487ybS7E4E&t=294s

[https://www.youtube.com/watch?v=IMSozUpFFkU](https://www.youtube.com/watch?v=IMSozUpFFkU)

[https://www.youtube.com/watch?v=0ql20JKrscQ&t=48s](https://www.youtube.com/watch?v=0ql20JKrscQ&t=48s)