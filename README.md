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
      "http://localhost:4004/**",
      "https://*.cfapps.us10-001.hana.ondemand.com/**"
    ]
  }
}
```
OAuth 설정 중 redirect-uris는 로컬 및 배포된 application의 URL Endpoint의 주소 패턴을 넣어야 한다.


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




# 배포 및 실행

## BTP Cloud Foundry에 배포

CF에 sample-cap-bookshop을 빌드 및 배포한다. 사전에 cf login 해야한다.

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

