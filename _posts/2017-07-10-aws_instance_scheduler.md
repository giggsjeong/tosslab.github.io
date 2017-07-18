---
layout: post
title: "AWS 비용 얼마까지 줄여봤니?"
author: ali
categories: [backend]
tags: [aws, lambda, ec2, rds, boto]
fullview: true
---

이 포스팅은 총 2부로 이어지며 현재는 1부 입니다.

> 1. 1부 : AWS 비용 얼마까지 줄여봤니?
> 2. 2부 : Instance Scheduler Bot 적용기

최근들어 스타트업의 인프라는 DevOps의 유행과 함께 IDC에서 클라우드로 급속도로 이전해가고 있습니다. 많은 클라우드 업체가 있지만 그 중에서도 Amazon Web Service (AWS) 가 가장 선호되고 있고 잔디도 AWS를 이용하여 서버 인프라를 구성하고 있습니다. 하지만 AWS 비용은 예상보다 만만치 않습니다. 비용을 줄이기 위한 여러가지 방법이 있지만 잔디에서는 어떻게 비용을 줄이는지 공유하도록 하겠습니다.

# AWS는 저렴한가?

AWS는 '저렴한 비용'을 자사 서비스의 큰 강점이라고 홍보하지만 실제 사용해보면 막상 ***'과연 정말 저렴한가?'*** 라는 의문을 가지게 됩니다. 여러 클라우드업체의 비용을 비교한 리포트를 보더라도 AWS는 절대 저렴하지 않습니다. 오히려 클라우드 업체 중 가장 비싼곳 중 하나입니다. 그렇다고 이제 와서 클라우드 업체를 옮기는건 배보다 배꼽이 더 클수도... (들어올때는 맘대로지만 나갈땐 아니란다.)

## 예약 인스턴스? 스팟 인스턴스? 온디맨드?

AWS에서는 제공하는 요금 할인 방법은 예약인스턴스나 스팟 인스턴스를 이용하는 것입니다.

**예약인스턴스**는 계약 기간에 따라 최대 60% 까지 저렴한 가격으로 이용할 수 있습니다. 하지만 정확한 기간과 수요예측을 하지 못한다면 잉여 인스턴스가 될 수 있습니다.

**스팟 인스턴스**는 입찰가격을 정해놓고 저렴할때 이용할 수 있습니다. 하지만 그 때가 언제일지도 알 수 없고 인스턴스를 가져갔다고 하더라도 더 높은 입찰가격을 제시한 새ㄲ..아니 사용자에게 인스턴스를 뺏길 수 있습니다. 마치 KTX를 입석티켓으로 빈 좌석에 앉아서 가다가 좌석 티켓 주인이 나타나 ***'내 자린데요?'*** 하면 얄짤없면 좌석을 내줘야 하는 느낌입니다. 그때 느끼는 그 서러움은 느껴보지 못한자는 알 수 없습니다.

**온디맨드**는 사용한 만큼 할인없이 비용을 지불하는 것입니다. 언제든지 필요할때 사용하고 사용한 만큼만 과금되어 가장 적절해보이지만 예약이나 스팟에 비해 역시나 비쌉니다. 비싸지만 현실적으로 가장 많이 사용됩니다.

## 개발서버는 얼마 안쓰는데 좀 깍아줘!

일반적으로 개발서버도 라이브와 같이 구성합니다. 고가용성은 고려하지 않더라도 아키텍쳐는 똑같이 구성하게 됩니다. 그리고 아키텍쳐가 복잡해질수록 구성하는 서버도 많아지고 언제부턴가는 개발서버도 비용을 무시할 수 없는 수준에 이르게 됩니다. 하지만 개발서버는 24시간 사용하지도 않고 업무시간에만 사용합니다. 이쯤되면 한번쯤 이런 생각을 하게 됩니다. ***'개발서버는 실제로 얼마 쓰지도 않는데 좀 깍아줘야 되는거 아냐?'*** 어디 개발서버 뿐이겠는가 정해진 시간만 사용하는 모든 서버들은 해당될 것입니다.

# EC2 Scheduler

AWS는 이러한 원성(?)을 들었는지 [EC2 Scheduler](https://aws.amazon.com/ko/answers/infrastructure-management/ec2-scheduler/) 라는 간단한 솔루션을 소개했습니다. 내용을 보면 설정된 시간과 요일에 자동으로 EC2 인스턴스가 자동으로 켜지고 꺼집니다. 하루 10시간 가용한다면 주말 제외 월~금요일만 작동시켜 비용을 70%나 절감할 수 있습니다.

![](/assets/media/post_images/ec2-scheduler-savings.png)

이대로만 된다면 왠만한 스팟이나 예약 인스턴스보다 더 저렴하게 개발서버를 이용할 수 있습니다. 하지만 이 솔루션을 그대로 도입하기에는 문제점들이 있었습니다.

## EC2 Scheduler 의 문제점

EC2 Scheduler는 다음과 같은 문제점들이 있습니다.

>1. **서버 아키텍쳐에 따라서 의존성이 있어 서버 실행 순서가 보장되어야 하는 경우가 고려되지 않는다.**
* 단순히 EC2 한두대 띄워서 사용하는게 아니고 훨씬 더 복잡한 서버 의존 관계를 가지게 됩니다다. 예를 들어 `DB -> Middleware -> API -> Batch` 같은 관계가 있다고 한다면 의존관계에 있는 서버들이 순차적으로 실행되어야 합니다.
>2. **스케쥴 시간이 UTC로만 작동한다.**
* UTC로만 작동하기 때문에 시간설정을 할때는 항상 UTC 기준으로 변환해야하는 불편함이 있습니다.
>3. **스케쥴링의 예외적인 상황이 고려되지 않는다.**
* 평일이 공휴일인 경우에는 서버를 작동할 필요가 없고 평소보다 서버를 일찍 켜야하거나 야근을 하게 되어 중지 시간을 변경해야 되는 경우에는 해당 일자에만 변경이 가능해야 했습니다.
>4. **EC2 에 대해서만 작동하도록 되어 있다.**
* EC2보다 비싼 RDS도 최근에 Stop 시킬 수 있도록 추가되었습니다.  `Aurora는 미지원`

잔디의 서버 아키텍쳐는 훨씬 복잡하여 서버의 실행순서가 맞지 않으면 정상작동을 하지 않기 때문에 1번은 반드시 해결되어야 하는 가장 치명적인 문제였습니다.

# Instance Scheduler

EC2 Scheduler의 문제점을 보안한 Instance Scheduler를 개발하였습니다. EC2나 RDS 모두 하나의 서버를 Instance라를 부르기 때문에 Instance Scheduler라 하였습니다. Instance Scheduler는 Serverless 아키텍쳐인 Cloudwatch + Lambda를 이용하여 구성되어 있습니다.

![](/assets/media/post_images/aws_instance_scheduler.png)

## 작동방식

Cloudwatch Event를 이용하여 Lambda를 함수를 실행시키고 Dynamo DB에 저장된 스케쥴 정보와 Instance의 Tag값을 기반으로 RDS와 EC2를 조회하고 Instance를 시작하거나 중지합니다. 그리고 Jandi의 [Incoming Webhook](http://blog.jandi.com/ko/2016/03/02/jandi-connect-webhook/)을 이용하여 토픽에 알림 메시지를 보내줍니다.

## Cloudwatch Event

Instance Scheduler Lambda 함수를 작동시키는 트리거는 Cloudwatch Event를 이용한다. 5분마다 작동시키도록 되어 있으며 각각의 사용 환경에 따라 변경할 수 있습니다.

>Cron 식 `0/5 * * * ? *`, 대상은 Instance Scheduler Lambda를 지정합니다.

![](/assets/media/post_images/cloud-watch-cron.png)

## Dynamo DB

Dynamo DB에는 Schedule, Schedule 예외 설정, Schedule 서버그룹에 대한 정보가 정의되어 있습니다.

### 1. Schedule

Schedule 작동에 대한 기본 정보를 정의하고 있습니다.

```json
{
  "ScheduleName": "Development",
  "TagValue": "Development",
  "DaysActive": "weekdays",
  "Enabled": true,
  "StartTime": "09:30",
  "StopTime": "22:00",
  "ForceStart": false
}
```

1. **ScheduleName**
    * Schedule의 Unique한 이름이다.
2. **TagValue**
    * 적용대상 Instance를 조회할때 참조하는 Tag 값입니다. Instance를 Schedule에 적용대상에 포함시키기 위해서는 해당 Instance의 Tag에 ScheduleName이라는 Key에 TagValue를 Tagging하면 됩니다.
2. **DaysActive**
    * Schedule 적용요일 입니다. 아래와 같은 옵션이 적용됩니다.
        * `all` : 매일
        * `weekdays` : 월~금
        * `mon,wed,fri` : 월,수,금요일
3. **Enabled**
    * Schedule의 작동여부 입니다.
4. **StartTime, StopTime**
    * 서버 시작시간과 중지시간 입니다.
5. **ForceStart**
    * Schedule 강제 시작 여부를 나타냅니다. (Enabled 여부에 상관없이 작동합니다.)

### 2. Schedule Server Group

하나의 Schedule에는 N개의 Server Group 을 정의 할 수 있고 각각은 먼저 실행되어야 하는 의존관계 Server Group 들을 정의하고 있습니다. 의존관계에 있는 Server Group의 Instance Status를 확인하여 시작 여부를 결정하도록 하였습니다. 그러면 의존관계가 없는 Server Group 부터 시작하고 의존관계의 Depth 가장 깊은 Server Group 은 가장 늦게 시작하게되어 서버 실행 순서를 보장하게 됩니다.

```json
{
  "Dependency": [
    "MONGODB",
    "RDB",
    "KAFKA",
    "REDIS"
  ],
  "GroupName": "API",
  "InstanceType": "EC2",
  "ScheduleName": "Development"
}
```
1. **Dependency**
    * 의존관계 Server Group 목록입니다.
2. **GroupName**
    * Server Group의 Unique한 이름입니다.
3. **InstanceType**
    * `EC2`와 `RDS` 를 지원합니다.

### 3. Schedule Exception

공휴일이나 야근등으로 인해 스케쥴을 미작동 시키거나 시간을 변경해야하는 경우에 예외사항들을 정의하고 있습니다.

```json
{
  "ExceptionUuid": "414faf09-5f6a-4182-b8fd-65522d7612b2",
  "ScheduleName": "Development",
  "ExceptionDate": "2017-07-10",
  "ExceptionType": "stop",
  "ExceptionValue": "21:00"
}
```

1. **ScheduleName**
    * 예외 적용 대상 Schedule 입니다.
1. **ExceptionDate**
    * 예외발생일 (YYYY-MM-DD)
2. **ExceptionType**
    * `start` : 시작
    * `stop` : 중지
3. **ExceptionValue**
    * `None` : 미작동
    * `H:M` : 변경시간

## Lambda

Instance Scheduler의 Lambda 코드는 Python으로 개발되었으며 [Github](https://github.com/tosslab/instance_scheduler)에 오픈소스로 공개하였습니다.
boto3는 배포 package에 Dependency를 추가하지 않아도 Lambda 실행환경에서 가용 라이브러리로 사용할 수 있다. 하지만 현재 기본적으로 사용할 수 있는 boto3 버전에서는 RDS Instance를 stop 할 수 있는 함수가 없기 때문에 최신버전이 필요합니다. 따라서 boto3 버전을 변경하여 함께 packaging 하여 업로드 하여야한다. 배포는 Lambda 관리도구인 [Apex](https://github.com/apex/apex)를 이용합니다. Apex를 이용하면 Dependency package 및 Lambda 생성 및 업데이트, 환경변수 설정 등을 모두 한번에 할 수 있습니다.

> 참조 : [Lambda Execution Environment and Available Libraries](http://docs.aws.amazon.com/ko_kr/lambda/latest/dg/current-supported-versions.html)

> AWS SDK는 Python [boto3](https://boto3.readthedocs.io/en/latest/) (`botocore:1.5.75`,`boto3:1.4.4`) 를 이용합니다.

## Jandi 메신저와 연동

AWS Instance Scheduler 는 Jandi 메신저의 Incoming Wehbook 을 이용하여 Webhook URL을 Lambda의 환경변수에 설정하면 서버의 시작과 중지에 대한 알람과 중지 10분전 부터 곧 서버가 중지된다는 알람을 발송하여 필요하다면 서버 중지 시간을 연장 할 수 있도록 합니다.

### Incoming Webhook 설정

Jandi의 토픽에서 Incoming Webhook을 연결하고 Webhook URL을 복사합니다.

![](/assets/media/post_images/incoming_webhook.png)

배포된 Lambda 함수의 Code 탭에서 Environment variables 에 WEBHOOK_URL을 등록합니다.

![](/assets/media/post_images/lambda.png)

### Instance Scheduler 알람

Server Group 시작되면 아래와 같이 알람 메시지를 표시합니다.

>![](/assets/media/post_images/aic_sc_2.png)

서버가 중지되기 전에 알람 메시지를 표시합니다.

>![](/assets/media/post_images/aic_sc_3.png)

# 정리

Instance Scheduler 는 EC2 Scheduler 에 비해서 다음과 같은 기능이 추가되었습니다.

1. 스케쥴 시간의 타임존 적용
2. 서버 그룹 설정 및 의존관계 설정
3. 스케쥴의 예외 설정
4. RDS 스케쥴 추가
5. 스케쥴에 상관없이 강제 시작 및 중지
6. 상태 알람

Lambda와 Dynamo DB의 사용 요금이 추가되었지만 Instance를 미사용시 stop 시켜 비용을 줄일수 있었습니다. Lambda와 Dynamo DB의 사용요금은 월 만원도 넘지 않는 금액이고 스케쥴에 의해 아껴지는 Instance 비용은 년 천단위 절감할것으로 예상하고 있습니다.

다음장에는 스케쥴 컨트롤을 위한 Bot 적용기를 다음장에서 소개하도록 하겠습니다.