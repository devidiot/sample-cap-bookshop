# 프로젝트 생성

CDS Sample을 추가한 CAP Node.js 신규 프로젝트 생성하여 HANA Cloud, XSUAA, AppRouter등을 추가하고 로컬 개발 환경 및 CF Deploy까지 실행하는 Example 소스

아래 단계로 진행하여 본 프로젝트를 생성함

```
cds init sample-cap-bookshop
cds add hana
cds add xsuaa
cds add approuter
```

## XSUAA 구성

xs-security.json 파일에 'xsappname', 'tenant-mode'와 OAuth 설정 'oauth2-configuration'을 추가함
```
{
  "xsappname": "sample-cap-bookshop",
  "tenant-mode": "dedicated",
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.us10-001.hana.ondemand.com/**"
    ]
  }
}
```
OAuth 설정 중 redirect-uris는 배포된 application의 URL Endpoint의 주소 패턴을 넣는다. 


## AppRouter 구성

app/xs-app.json 파일에 'authenticationMethod', 'authenticationType'을 지정한다.
authenticationMethod는 아래 route로 지정하고, authenticationType은 xsuaa로 지정한다. 
아래는 xs-app.json의 전체 

```
{
  "welcomeFile": "app/index.html",
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/app/(.*)$",
      "target": "$1",
      "localDir": ".",
      "cacheControl": "no-cache, no-store, must-revalidate",
      "authenticationType": "xsuaa"
    },
    {
      "source": "^/appconfig/",
      "localDir": ".",
      "cacheControl": "no-cache, no-store, must-revalidate",
      "authenticationType": "xsuaa"
    },
    {
      "source": "^/(.*)$",
      "target": "$1",
      "destination": "srv-api",
      "csrfProtection": true,
      "authenticationType": "xsuaa"
    }
  ]
}
```
authenticationType을 none으로 지정할 경우 해당 라우팅은 인증하지 않아도 서비스된다. 예를들어 "source": "^/app/(.*)$" 에 authenticationType을 none으로 할 경우 로그인 없이도 화면은 열린다.


<br/> <br/> <br/> <br/> 

# 배포 및 실행
사전에 cf에 login 해야한다.

https://help.sap.com/docs/btp/sap-business-technology-platform/log-on-to-cloud-foundry-environment-using-cloud-foundry-command-line-interface



## BTP Cloud Foundry에 배포

CF에 sample-cap-bookshop을 빌드 및 배포한다.

```
mbt build
cf deploy ./mta_archives/sample-cap-bookshop_1.0.0.mtar 
```

## Service Key 생성 (XSUAA & HDI)

배포 후 XSUAA Service와 HDI Container Service의 Service Key를 생성한다.
```
cf create-service-key sample-cap-bookshop-auth sample-cap-bookshop-auth-key
cf create-service-key sample-cap-bookshop-db sample-cap-bookshop-db-key
```

## 실행
BTP Cockpit에서 CF에 배포된 Application 'sample-cap-bookshop'을 선택 후 Application Routes URL을 클릭하여 실행한다.
admin 권한은 없으므로 Browse Books 만 실행 가능하며, Manage Books는 Forbidden 오류가 발생한다.


<br/> <br/> <br/> <br/> 

# 로컬 구동 - Hybrid Profile

결론적으로 CDS Backend 서비스와 AppRouter를 통한 Frontend 서비스가 각각 구동된다. 
Frontend인 Fiori Application이 Backend 서비스를 호출 하려면 Destionation을 이용해야한다. 

CF에 배포된 서비스의 경우 MTA를 통해 destionation 이름 srv-api 가 구성된다. (mta.yaml 참조)
로컬 서비스의 경우 Destination을 구성해야한다. 
혹시 app/default-env.json 파일이 없다면 아래와 같이 직접 파일을 생성하여 작성한다. 
```
{
  "destinations": [
    {
      "name": "srv-api",
      "url": "http://localhost:4004",
      "forwardAuthToken": true
    }
  ]
}
```

## CF와 Binding

CF에 생성된 HDI Container 서비스와 XSUAA 서비스에 로컬 개발환경을 연결하여 개발이 가능하도록 한다.
Hybrid 라는 이름의 profile을 구성하여 이를 통해 로컬 서비스를 구동한다.

아래 두 명령을 실행하여 HDI Container와 XSUAA 서비스를 바인드한다.
```
cds bind -2 sample-cap-bookshop-db
cds bind -2 sample-cap-bookshop-auth
```

## 실행

위 명령을 실행하면 프로젝트 Root 폴더에 .cdsrc-private.json 파일이 생성된다. 로컬에서 서비스 구동시 이를 활용하여 CF 서비스에 바인딩한다.
아래 명령으로 CDS 서비스를 구동한다. 
```
cds watch --profile hybrid
```

기본 포트번호 4004로 실행한 후에 모든 CDS Service는 Unauthorized가 된다. XSUAA를 활성화 했기 때문이며, 이는 AppRouter를 통해 접속해야한다. 

아래 명령으로 AppRouter 서비스를 구동한다. 
```
cds bind --exec -- npm start --prefix app
```

**이 소스는 AppRouter가 5001 포트로 구동하도록 구성되어 있다.**


## Destination 설정

AppRouter의 기본 포트는 5001 번이나 이 포트가 이미 사용중이라면 Port를 변경해야한다.
app/package.json 파일의 start script에 --port 옵션으로 5001을 추가한다.
```
{
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js --port 5001"
  }
}
```


## OAuth login callback URI 추가 구성

로컬 환경 또한 login/callback URL 패턴을 추가해야한다.
xs-security.json 파일에 OAuth 설정 'oauth2-configuration'의 redirect-uris에 BAS 또는 VSCode를 사용할 경우의 AppRouter URL 패턴을 추가한다.

```
{
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.us10-001.hana.ondemand.com/**",
      "https://*.applicationstudio.cloud.sap/**",
      "http://localhost:5001/**"
    ]
  }
}
```

**xs-security.json을 수정 한 후 이를 반영하기 위해 재 배포해야한다. Hybrid profile을 구성하여 CF에 바인딩 하여 구동하는 방식이므로 CF에 수정된 xs-security.json이 포함된 "sample-cap-bookshop"이 update되어야 한다.**


