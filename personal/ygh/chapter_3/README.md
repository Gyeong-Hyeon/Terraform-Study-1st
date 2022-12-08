# Chapter3: 테라폼 상태 관리하기

테라폼 상태는 `terraform apply` 명령어를 실행하는 순간 프로젝트 폴더 안에 `terraform.tfstate`라는 파일에 기록된다. (JSON 형태)

이 후 테라폼을 실행할 때마다 현재 서버 상태와 해당 파일 안의 상태값을 비교하여 변화한 값을 찾아내어 적용한다.

> 권장하지 않지만 terraform state 파일을 조작해야하는 상황이 생길 경우, `terraform import`나 `terraform state`라는 명령어를 사용할 수 있다.

프로덕트 단에서 테라폼을 사용하기 위해서는 이 상태값을 공유해야하는데, 이 때 다음과 같은 요구사항들이 발생한다.

## State File을 공유하기

Git과 같은 version control platform들에 의존하는 것은 다음과 같은 이유로 권장되지 않는다.

1. 수동 작업으로 발생되는 에러

     작업 전 최신으로 pull 하는 것을 잊거나, 작업 후 push하는 것을 잊는 등 작업 공간(로컬)과 공유 허브 간 싱크를 맞추는 과정에서 수동 작업이 필요하여 에러 발생 가능

     → 다른 팀원이 최신 버전 이전의 코드에서 이후 작업을 이어나가는 문제 초래
    
2. 동시 작업 문제

     대부분의 version control platform들은 동시 작업을 막는 locking 기술을 지원하지 않음

3. 보안 문제

     테라폼 상태 파일은 모두 plain text로 작성

     version control platform에 민감 정보들을 plain text로 업로드하게 되면 보안 문제 발생

이러한 문제들을 방지하기 위해 테라폼에서 built in으로 제공하는 원격 백엔드를 사용할 수 있다.
- Default: Local disck
- AWS S3
- Azure Storage
- Google Cloud Storage
- HashiCorp's Terraform Cloud
- Terraform Enterprise

위의 백엔드는 다음과 같은 방법으로 앞서 말했던 문제들을 해결하면서 추가적인 benefit을 제공한다.

1. `terraform plan` 또는 `terraform apply`시 자동으로 테라폼 상태 파일을 백엔드로 업데이트
2. `terraform apply` 수행 시 백앤드를 잠금으로써 동시 업데이트 방지. 잠긴 동안 수행된 apply는 잠금이 풀린 후 수행
3-1. 테라폼 상태파일을 자동으로 암호화 하여 업데이트
3-2. S3등의 저장소에 접근하려면 access permission이 필요하므로 접근 권한 설정 가능
4. Managed service이므로 관리 리소스 및 데이터 유실 가능성 감소 (가용성, 내구성이 보장된 platform)
5. 버저닝 서비스 제공

## Terraform backend의 한계

### 원격 저장소를 새로 생성하는 경우:
 테라폼 상태 파일을 원격 저장소에 업데이트 하려면 원격 저장소가 생성이 되어 있어야하기 때문에, 원격 저장소를 생성하는 테라폼 코드를 먼저 실행시킨 후, 테라폼 상태 파일을 업데이트하는 코드를 한번 더 실행시켜야 함
     1. S3 버켓과 DynamoDB 테이블을 생성하는 테라폼 코드 작성 후 배포
     2. S3 버켓과 DynamoDB 테이블을 원격 백엔드 구성에 추가한 후 `terraform init`을 수행하여 1을 통해 생성한 S3 버켓과 DynamoDB에 로컬 상태를 copy

### 원격 저장소를 삭제하는 경우:
 테라폼 상태 파일을 로컬 디스크로 옮기는 과정이 필요하므로 역시 코드를 두 번 실행시키는 과정이 필요함
     1. backend 구성을 삭제하고 terraform init을 수행하여 테라폼 상태값을 로컬 디스크로 옮긴다.
     2. `terraform destroy`를 통해 테라폼 리소스 삭제

### 변수 사용 불가
 Backend 모듈에서는 변수를 사용할 수 없음
 ```
 terraform {
     backend "s3" "state" {
          bucket = var.bucket #사용불가
          region = var.region #사용불가
          key = "/terraform.tfstate"
          ...
     }
 }
 ```
 차선책으로 `{file_name}.hcl`이라는 파일([공식문서](https://developer.hashicorp.com/terraform/language/settings/backends/configuration#partial-configuration)에서는 `{file_name}.{backend_name}.tfbackend`형식을 권장한다)을 정의하여 backend configuration을 정의하고, `terraform init`시 `-backend-config` command line을 통해 사용할 수 있음
 ```
 # config.state.tfbackend
 
 bucket = "terraform-state"
 region = "us-northeast-2"
 ```

 ```
 # main.tf

 terraform {
     backend "s3" "state" {
          key = "/terraform.tfstate"
          ...
     }
 }
 ```

 ```bash
 terraform init -backend-config=config.state.tfbackend
 ```
 또는 Terragrunt라는 오픈소스툴 사용 가능 (Chpater10에서 소개 예정)

## 상태 파일 분리
