# ![logo-wide](https://github.com/me-X-us/server-architecture/blob/main/logo-wide.png?raw=true)  
트집은 웹브라우저와 웹캠만 있으면 언제 어디서든 헬스,스포츠,댄스등의 트레이닝서비스를 받을 수 있도록 하는 서비스 플랫폼입니다.

## 서버 구조
서비스 특성상, 미디어 스트리밍, 파일 업로드 등의 기능으로 IO부하가 발생, 머신러닝 구동으로 GPU자원이 필요하기 때문에 
서비스간 성능저하, 장애와 같은 간섭을 줄이기 위하여 마이크로서비스 아키텍처(MSA)를 적용하였습니다.  
![server architecture](https://github.com/me-X-us/server-architecture/blob/main/server-architecture.png?raw=true)  

### API Gateway
캐싱, 로드밸런싱, 보안설정을 위하여 최상단에 API Gateway를 두엇습니다.
API Gateway는 NGINX로 구성된 Reverse Proxy이며,모든 서비스는 이 프록시를 통해야만 접근이 가능하기 때문에 
모든 서비스 서버의 실 주소를 숨길 수 있어 보안상 더욱 안정적이며, 상황에 맞게 로드밸런싱 설정이 가능합니다.
추가적으로 Let's Encrypt SSL 인증서를 발급받아 적용하여 Https 연결을 지원합니다.

### API Server
[API Server](https://github.com/me-X-us/api-server) 는 계정 인증을 포함한 클라이언트 구동에 필요한 대부분의 무겁지 않은 API를 지원해주는 역할을 합니다.

### Streaming Server
[Streaming Server](https://github.com/me-X-us/straming-server) 는 Video Storage에서 요청받은 트레이너 영상/형상을 스트리밍해주는 역할을 합니다.  

### Upload Server
[Upload Server](https://github.com/me-X-us/upload-processing-server) 는 트레이너가 트레이닝 영상을 올릴 수 있도록 지원해주는 역할을 합니다.
영상 업로드 요청을 받으면 파일을 Video Storage에 저장하며, Pose Estimation 서버에 트레이너의 동작정보 추출을 의뢰하여 DB에 저장한 후,
Shape Estimation 서버에 트레이너 형상 영상 제작을 의뢰합니다. 

### Pose Estimation Server
[Pose Estimation Server](https://github.com/me-X-us/pose-estimation-server) 는 HRNet을 사용하여 트레이너의 동작 정보를 추정하여 반환하는 역할을 합니다.
CPU/GPU 두가지의 버전이 있으며, 빠른 서비스 제공을 위해 실 서비스에서는 GPU버전을 사용합니다.

### Shape Estimation Server
[Shape Estimation Server](https://github.com/me-X-us/shape-estimation-server) 는 maskRCNN을 사용하여 사용자의 웹캠에 합성되어 제공될 트레이너의 형상 영상을
제작하여 Video Storage에 저장합니다. CPU/GPU 두가지의 버전이 있으며, 빠른 서비스 제공을 위해 실 서비스에서는 GPU버전을 사용합니다.

## 배포 관리
본 프로젝트는 Github Actions를 사용하여 CI/CD 환경을 구성했습니다. 각 서버는 Docker 이미지로 관리가 되며,
정해진 branch에 변경이 생길때마다 새로운 docker 이미지 빌드 후 [조직의 Github Package Registry](https://github.com/orgs/me-X-us/packages) 에 등록되어 서버에 배포됩니다.

## 인증
서버는 JWT를 사용한 인증체계를 사용중입니다. 모든서버는 사용자가 발급받아 제출한 JWT토큰의 권한에 맞게 요청을 처리하며, 보안을 위해 AccessToken, RefreshToken을 사용합니다.
AccessToken은 10분주기로 갱신되어야 하며, RefreshToken을 사용하여 갱신받을 수 있습니다.

## 장애 대응 시스템
모든 서버는 장애대응 시스템을 포함한 상태입니다. 서버에서 핸들링 되지 않은 장애 발생시 클라이언트에는 500 에러를 반환한 후 지정된 Slack 채널에
아래와 같이 Stack Trace, 요청정보, 사용자 정보, 요청 브라우저 정보, 서버정보를 포함한 장애 보고서를 제출하도록 구성이 되어있습니다.  
![error-report](https://github.com/me-X-us/server-architecture/blob/main/error-report.png?raw=true)  
