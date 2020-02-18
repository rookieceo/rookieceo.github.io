---
published: false
---
## AWS Codepipeline을 사용하여 CI/CD 환경 구성하기(beanstalk)

작성중....

순서
1. commit 설정 (AWS Codecommit)
![/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-14.jpg](/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-14.jpg)
2. build (Codebuild)
- 설정
![/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-29.jpg](/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-29.jpg)
- buildspec.yml

- cache 설정(S3)

3. Deploy(AWS Elastic beanstalk)
![/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-35.jpg](/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-35.jpg)

4. 결과 확인
![/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-43.jpg](/assets/images/2020-02-18-aws-codepipeline/2020-02-18 PM 04-06-43.jpg)
