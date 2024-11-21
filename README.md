# 배포 파이프라인 구축 가이드

## **배포 다이어그램**

---

![배포다이어그램](/public/diagram.png)

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다.

1. 저장소를 체크아웃합니다.
2. Node.js 18.x 버전을 설정합니다.
3. 프로젝트 의존성을 설치합니다.
4. Next.js 프로젝트를 빌드합니다.
5. AWS 자격 증명을 구성합니다.
6. 빌드된 파일을 S3 버킷에 동기화합니다.
7. CloudFront 캐시를 무효화합니다.

# 주요 링크

- S3 버킷 웹사이트 엔드포인트:  
  [http://hyen-bucket.s3-website.ap-northeast-2.amazonaws.com/](http://hyen-bucket.s3-website.ap-northeast-2.amazonaws.com/)
- CloudFront 배포 도메인 이름:  
  [https://dm1ip4fw2ps0h.cloudfront.net/](https://dm1ip4fw2ps0h.cloudfront.net/)

# 주요 개념

### GitHub Actions과 CI/CD 도구:

  <details>
    <summary>CI/CD란?</summary>
    <div markdown="1">
    <p><strong>CI</strong> : 코드 변경 사항을 정기적으로 테스트하고, 빌드하여 저장소에 통합하는 과정입니다.</p>
    <ul>
    <li>예: 테스트 자동화, 빌드, Docker 이미지 생성 및 저장.</li>
    </ul>
    <p><strong>CD</strong> : CI 과정을 통해 통합된 코드를 실제 프로덕션 환경(사용자에게 배포되는 환경)에 배포하는 과정입니다.</p>
    <p>CI/CD는 코드 변경 사항을 빌드, 테스트, 배포까지 자동화하는 프로세스를 의미합니다.</p>
    </div>
  </details>

  <details>
    <summary>GitHub Actions란?</summary>
    <div markdown="1">
      <ul>
        <li>🟡  GitHub에서 제공하는 <strong>CI/CD 및 자동화 도구</strong>입니다. 코드의 빌드, 테스트, 배포 및 기타 작업을 자동화할 수 있습니다.</li>
        <li>🟡 
        <strong>Workflow: </strong>
          GitHub Actions에서 실행되는 하나 이상의 Job으로 구성된 자동화 프로세스입니다.
          <ul>
            <li>.github/workflows 디렉토리에 <strong>YAML 파일</strong>로 설정을 정의합니다.</li>
            <li>push, pull_request 등의 GitHub 이벤트에 의해 트리거됩니다.</li>
            <li>정의된 YAML 파일에 따라 GitHub Actions가 자동으로 작업을 실행합니다.</li>
          </ul>
        </li>
        <li>추가 내용은 <a href="https://docs.github.com/ko/actions">깃허브 사용설명서</a>를 참고해주세요.</li>
      </ul>
    </div>
</details>

### S3와 스토리지:

<details>
  <summary>Amazon S3 (Simple Storage Service)란?</summary>
  <div markdown="1">
    🟡 <strong>Amazon S3 (Simple Storage Service)</strong>  
    AWS에서 제공하는 <strong>클라우드 객체 스토리지 서비스</strong>입니다.
    <ul>
      <li><strong>객체 스토리지</strong>:
        <ul>
          <li>정적 파일(이미지, 동영상, 문서 등)의 업로드, 삭제, 업데이트만 가능합니다.</li>
          <li>소프트웨어 설치나 실행은 불가합니다.</li>
          <li>블록 스토리지(EBS, EFS)와는 다릅니다.</li>
        </ul>
      </li>
      <li><strong>무제한 용량 지원</strong>:
        <ul>
          <li>단일 객체(파일) 크기는 <strong>0바이트~5TB</strong> 제한됩니다.</li>
          <li>파일 개수는 무제한으로 업로드 가능합니다.</li>
        </ul>
      </li>
      <li><strong>Bucket</strong>:
        <ul>
          <li>파일을 저장하는 단위로, 디렉토리와 유사한 개념입니다.</li>
          <li><strong>Bucket</strong> 이름은 전 세계적으로 고유해야 합니다.</li>
          <li>정적 웹 호스팅 시, 도메인 이름과 Bucket 이름이 동일해야 합니다.</li>
        </ul>
      </li>
    </ul>
  </div>
</details>

### CloudFront와 CDN:

<img src="/public/cdn.png" alt="배포다이어그램" width="500"/>

<details>
  <summary>CDN (Content Delivery Network)</summary>
  - **CDN**은 **사용자에게 빠르게 콘텐츠를 전달**하기 위해 데이터를 여러 지역의 서버(엣지 서버)에 분산하여 캐시하고, **사용자에게 가장 가까운 서버**에서 콘텐츠를 제공하는 네트워크입니다.
  - **DNS**를 사용하여, **오리진 서버** 대신 **가장 가까운 엣지 서버**에 연결되도록 하여, **물리적 거리**와 관계없이 더 빠르고 안정적으로 콘텐츠를 제공합니다.
  - CDN은 **정적 콘텐츠**와 **동적 콘텐츠**를 효율적으로 처리합니다.

  <br>

**🟡 정적 캐싱 (Static Caching)**

- 미리 데이터를 **엣지 서버에 캐시**해 두고, 사용자가 요청할 때 바로 해당 데이터를 제공하는 방식입니다.
- 사용 예시: **HTML, CSS, JS, 이미지**, **동영상** 등의 정적 파일은 미리 캐싱해 두어 빠르게 제공됩니다.
- 비유: **냉장고**에 이미 식재료가 준비되어 있어 필요할 때 바로 꺼내 쓸 수 있는 상태.

  <br>

**🟡 동적 캐싱 (Dynamic Caching)**

- 사용자가 요청할 때마다 **서버에서 실시간으로 데이터를 받아오거나**, **캐시 미스**(cache miss)가 발생하면 원본 서버에서 데이터를 받아와서 제공하는 방식입니다.
- 사용 예시: **실시간 날씨 데이터**나 **API 요청**과 같은 동적 데이터를 처리할 때 사용됩니다.
- 비유: 프랜차이즈 **식당에서 손님이 원하는 요리를 그때그때마다 본사에 재료를 주문하여 조리**하는 방식. 즉, 필요한 재료가 없으면 본사에서 주문해와서 바로 제공하는 방식입니다.

  <br>

**🟡 CDN의 장점**

- **오리진 서버의 부하 감소**: CDN을 사용하면 트래픽이 분산되어 오리진 **서버로의 요청**이 줄어들어 **서버 비용**과 **대역폭 비용**이 절감됩니다.
- **전송 속도 향상**: **엣지 서버**에서 데이터를 제공함으로써 **물리적 거리**를 단축시켜, **로딩 속도**가 향상됩니다.
- **대역폭 비용 절감**: **대역폭**은 얼마나 많은 데이터를 동시에 전송할 수 있는지에 대한 개념으로, CDN은 대역폭을 절약하여 오리진 서버의 네트워크 부담을 줄입니다.

  - 비유: **도로폭**이 넓을수록 더 많은 차선이 있어 동시에 많은 차량(데이터)이 이동할 수 있습니다.

  <br>

**🟡 CDN 활용 예시**

- **전 세계적인 웹사이트**: 글로벌 웹사이트나 **전자상거래 사이트**에서 CDN을 사용하여 **빠른 페이지 로딩**과 **효율적인 콘텐츠 제공**을 구현합니다.
- **비디오 스트리밍**: YouTube나 Netflix와 같은 **스트리밍 서비스**는 CDN을 사용하여 **지리적으로 분산된 서버**에서 콘텐츠를 전달하여 사용자 경험을 향상시킵니다.

</details>

### 캐시 무효화(Cache Invalidation):

<details>
  <summary> 캐시 무효화 및 CloudFront 설정</summary>
  캐시 무효화 및 CloudFront에서의 캐시 관리
  캐시 무효화는 CDN(Content Delivery Network)에서 캐시된 콘텐츠를 갱신하거나 삭제하여, 최신 데이터를 사용자에게 적절하게 제공하는 중요한 단계입니다. 이를 통해 오래된 콘텐츠가 제공되지 않도록 보장하고, 데이터의 정확성과 최신성을 유지할 수 있습니다.

🟡 자동 캐시 무효화

- TTL (Time to Live): 캐시된 데이터의 유효 기간을 설정하는 값입니다. TTL이 지나면 캐시된 데이터는 만료되고, 다시 원본 서버에서 데이터를 받아옵니다.
- 동적 콘텐츠(예: 실시간 API 요청, 사용자 맞춤형 데이터)는 매번 새로 받아오기 때문에 캐시 무효화에 대해 신경 쓸 필요가 없습니다.

🟡 수동 캐시 무효화

- CDN은 캐시 무효화를 위한 기능(예: `Cache Invalidation` 명령)을 제공합니다. CloudFront, Akamai, Cloudflare 등에서는 콘솔이나 API를 통해 특정 콘텐츠를 무효화하거나 삭제할 수 있습니다.
- 캐시 버스팅(Cache Busting) 방식을 사용하면, 파일 이름에 버전 정보나 타임스탬프를 추가하여 새로운 파일로 인식되게 할 수 있습니다. 이 방법은 파일을 수동으로 관리해야 할 수 있습니다.
  예시: 기존 파일: `styles.css` → 캐시 버스팅 후: `styles.v1.css`

🟡 캐시 무효화 비유: 프랜차이즈 레스토랑에서 기존에 주문된 식재료가 오래되었거나 상태가 나쁘다면, 해당 식재료를 본사에 요청하여 새로운 신선한 재료로 교체해야 합니다. 마찬가지로, 캐시 무효화는 오래된 데이터를 삭제하고 새로운 데이터를 제공하기 위해 기존 캐시를 갱신하는 과정입니다.

🟡 TTL(Time to Live) 설정을 통해 캐시된 파일의 유효 기간을 조정하고, 유효 기간이 지난 파일은 자동으로 새로 갱신되도록 할 수도 있습니다.

**CloudFront에서 캐시 무효화를 수행하는 방법은 두 가지가 있습니다.**

- **Cache Invalidation**:
  - 콘솔에서 직접 삭제할 파일을 선택하거나 API를 통해 특정 파일을 캐시에서 삭제할 수 있습니다.
  - 기본적으로 무효화 명령은 시간이 다소 걸릴 수 있으며, 비용이 발생할 수 있습니다.

🟡 **파일 이름 변경 후 자동 캐시 갱신**

- 파일 이름을 변경하는 것은 CloudFront에서 자동적으로 캐시를 갱신하는 방법 중 하나입니다.
- 예를 들어, `cat.png`를 `cat2.png`로 변경하면, CloudFront는 `cat2.png`를 새로운 파일로 인식하여 자동으로 캐시합니다.
- 중요한 점은 기존 파일(예: `cat.png`)은 더 이상 사용되지 않게 되며, 이 파일의 캐시는 그대로 남아 있을 수 있습니다. 이 경우에는 수동으로 캐시 무효화를 하지 않아도 새로운 파일(`cat2.png`)은 정상적으로 캐시되고 제공됩니다.

🟡 **캐시 무효화 명령을 사용한 수동 처리**

- `aws cloudfront create-invalidation --paths "/cat.png"` 명령처럼 캐시 무효화 명령을 사용하면, CloudFront 캐시에서 특정 파일을 삭제하거나 갱신할 수 있습니다.
- 만약 파일 이름을 변경했지만, CloudFront에서 여전히 이전 버전의 파일을 제공하고 있다면, 캐시 무효화 명령을 사용하여 이를 수동으로 삭제할 수 있습니다.
- 예시:
  - `aws cloudfront create-invalidation --paths "/cat.png"`는 `/cat.png` 파일의 캐시를 무효화합니다.
  - 여러 파일을 무효화할 때는 `/*` 와일드카드를 사용하여 모든 파일을 일괄적으로 처리할 수 있습니다.
  - 예시: `aws cloudfront create-invalidation --paths "/*"`는 배포된 모든 파일의 캐시를 무효화합니다.

</details>

### Repository secret과 환경변수:

<details>
  <summary>GitHub Secrets 설정 및 민감 정보 관리</summary>
  
  - **GitHub Secrets**

    - GitHub 리포지토리에서 민감한 정보를 안전하게 저장하는 기능입니다.
    - **암호화되어 저장**되며, **저장소 외부에서 볼 수 없도록 보호**됩니다.
    - **API 키**나 **인증 토큰**과 같은 민감한 정보를 코드에 노출시키지 않고, 워크플로 내에서 안전하게 활용할 수 있습니다.
    - GitHub Actions 워크플로에서 **환경 변수처럼** 사용될 수 있습니다.

    🟡 **GitHub Secrets 설정**

    - GitHub 리포지토리에서 **Settings** > **Secrets** > **New repository secret**으로 이동하여, 필요한 정보를 추가합니다.

    🟡 YAML 파일에서 Secrets 사용

    - GitHub Actions 워크플로 내에서 Secrets를 환경 변수처럼 사용할 수 있습니다.
    - Secrets는 `secrets` 객체를 통해 참조됩니다.

    ```yaml
    name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: ${{ secrets.AWS_REGION }}
    ```

    Q. 민감한 정보를 GitHub Actions 워크플로에서 **secrets** 없이 코드나 `yml` 파일에 하드코딩하면 어떻게 될까요?

    A. 여러 보안 문제가 발생할 수 있습니다:

    1. **정보 노출**: 코드에 민감한 정보가 포함되면 누구나 이를 볼 수 있고, 공개 리포지토리에서 완전히 노출될 수 있습니다.
    2. **탈취 위험**: 해커가 하드코딩된 민감 정보를 쉽게 탈취해 악용할 수 있습니다.
    3. **Git 이력 문제**: 커밋 기록에 민감 정보가 포함되면, 정보 삭제가 어려워 추후에도 노출될 수 있습니다.
    4. **협업 시 위험**: 코드 리뷰나 PR을 통해 실수로 민감 정보가 노출될 수 있습니다.
    5. **즉각적인 대응 어려움**: 정보 유출 시 즉시 대처가 어려워 보안 사고가 커질 수 있습니다.
       따라서 민감한 정보는 반드시 **GitHub Secrets**와 같은 안전한 방법을 사용해 관리해야 합니다.

    🟡 GitHub Secrets 외의 민감 정보 관리 방법

    - **환경 변수**를 사용하여 로컬 개발 환경이나 서버에서 민감한 정보를 관리할 수 있습니다. `.env` 파일을 통해 환경 변수로 처리하고, 이를 `.gitignore`에 추가하여 Git에 민감 정보가 올라가지 않도록 해야 합니다.
    - **AWS Secrets Manager** 또는 **Azure Key Vault**와 같은 **클라우드 서비스의 보안 관리 도구사용**
    - **Encryption**: 데이터를 저장하거나 전송할 때 암호화 기술을 사용하여 정보를 보호

</details>

---

# **CDN과 성능 최적화 보고서**

## **프로젝트 개요**

- 배포 방식:
  1. S3 단독 정적 웹 호스팅
  2. S3 + CloudFront 호스팅
- **목표**: 두 가지 배포 방식의 성능 차이를 비교하고, 각 방식의 특징을 이해

---

## CDN 캐싱 적용 여부에 따른 성능 비교

| **지표**                      | **S3**   | **CloudFront** |
| ----------------------------- | -------- | -------------- |
| **TTFB (Time To First Byte)** | 30.94 ms | 8.59 ms        |
| **DNS Lookup Time**           | 37.96 ms | 12.61 ms       |
| **FCP**                       | 56.66ms  | 66.39          |
| **Load**                      | 614ms    | 132ms          |
| **DOMContentLoaded**          | 517ms    | 54ms           |
| **transferred**               | 706kb    | 429kb          |

---

## 캐싱 효율성

- **S3**는 파일을 저장하는 곳이고, 그 자체로 캐시 상태를 다루지 않습니다.
- **CloudFront**는 캐시와 관련된 여러 가지 헤더(`X-Cache` 등)를 추가하며, 캐시를 처리하는 역할을 합니다.
- 따라서 `X-Cache`와 같은 캐시 관련 헤더는 CloudFront에서만 볼 수 있고, S3에서는 나타나지 않습니다.
- `Content-Encoding`은 전송되는 데이터의 압축 여부를 다루고, `X-Cache`는 캐시 상태를 나타내어 서버와 캐시 시스템 간의 상호작용을 추적합니다.
- **좌 - S3 , 우- CloudFront**
  ![배포다이어그램](/public/cache.png)

---

## TTFB(Time to First Byte)

- TTFB(Time to First Byte)는 서버가 첫 번째 바이트를 응답하는 데 걸리는 시간입니다. **0.8초 이하**의 TTFB를 목표로 하면, 사용자가 빠르게 콘텐츠를 경험할 수 있습니다. 이는 **FCP**나 **LCP** 같은 주요 성능 지표가 원활하게 동작할 수 있도록 도와줍니다.
  ![배포다이어그램](/public/ttfb.png)
- TTFB는 다음 요청 단계의 합계입니다.
  - 리디렉션 시간 / 서비스 워커 시작 시간(해당하는 경우)
  - DNS 조회 / 연결 및 TLS 협상 / 응답의 첫 번째 바이트가 도착할 때까지의 요청
- 🟡 **S3**

  **DNS Lookup:** 37.96 ms  
  **Waiting for server response:** 30.94 ms

  ![DNS Lookup -s3](/public/s33.png)

- 🟡  **CloudFront**  
  **DNS Lookup:** 12.61 ms  
  **Waiting for server response:** 8.59 ms

  ![DNS Lookup -cf](/public/cf3.png)

---

## HTTP와 HTTPS의 차이: SSL/TLS 핸드셰이크

**S3 : HTTP , cloudFront: HTTS**

HTTP에서는 `Security` 탭에 정보가 나타나지 않고 HTTPS에서만 관련 정보가 표시됩니다.

이유는 SSL/TLS 핸드셰이크가 HTTPS에서만 발생하기 때문입니다.

**SSL/TLS 핸드셰이크**는 클라이언트와 서버 간의 보안 연결을 설정하는 과정으로, 이 과정이 HTTP보다 더 긴 시간이 걸리므로 **HTTPS**에서는 **페이지 로딩 시간**이 늘어날 수 있습니다.

![DNS Lookup -cf](/public/http_screat.png)

---

## **HTTP(S)** 캐시와 네트워크 최적화, 서버 처리 속도 차이를 비교

이미지 순서: S3 , **CloudFront**

![DNS Lookup -cf](/public/s3.png)
![DNS Lookup -cf](/public/cf.png)

### Load Time (614ms vs. 132ms)

: 페이지의 모든 리소스가 완전히 로드되는 데 걸린 시간

- **CloudFront는** 전 세계 여러 엣지 서버에 캐시된 콘텐츠를 제공하므로 **S3**에서 직접 제공되는 것보다 빠르게 리소스를 로드할 수 있습니다. **S3**는 원본 서버에서 직접 데이터를 가져오기 때문에 더 긴 시간이 걸릴 수 있습니다.

### DOMContentLoaded (517ms vs. 54ms)

: DOM이 완전히 로드되어 사용자가 상호작용할 수 있게 되는 시점

- **CloudFront**의 캐시가 더 효율적으로 동작하여 캐시된 콘텐츠가 클라이언트에 가까운 서버에서 제공되기 때문에 페이지의 DOM 구성이 더 빨리 완료됩니다.

### Finish (7.53s vs. 6.98s)

: 페이지 로드가 완료되기까지의 전체 시간을 의미

- 이 시간은 **CloudFront**가 약간 더 빠른 성능을 보이지만, 차이는 약간의 차이로, 주로 캐시 효율성, 네트워크 지연, 서버 응답 시간 등에 따라 달라질 수 있습니다.

### Transferred (706KB vs. 429KB)

**:** 네트워크를 통해 전송된 데이터의 양

- **S3**는 **CloudFront**를 통해 제공되기 전, 클라이언트에 의해 직접 데이터를 요청하므로 더 많은 데이터가 전송되고, **CloudFront**는 캐시된 리소스를 제공하여 네트워크 트래픽을 줄입니다.

---

## **결론**

CDN은 성능 최적화에 큰 이점이 있지만, **무조건 사용하자**는 접근보다는 **상황에 맞게 활용**하는 것이 더 효과적으로 보여집니다.

- **S3 단독**:
  - 간단한 프로젝트에 적합 합니다.
  - 단순히 정적 파일을 제공하는 용도로만 사용할 때 혹은 HTTPS 같은 추가적인 보안 요구가 없는 단순 파일 공유의 경우 S3만으로 충분할 수 있습니다.
  - 사용자 지연 시간이 크게 중요하지 않은 경우 선택하는 것이 좋겠습니다.
- **S3 + CloudFront**:
  - 높은 성능과 글로벌 사용자를 대상으로 하는 애플리케이션에 적합 할 것으로 보여 집니다.
  - HTTPS, 보안 기능 등 추가적인 기능성이 필요한 경우 사용하면 유용할 것 같습니다.

---

### **S3와 CloudFront관련 추가 학습 및 참고사항.**

<details>
  <summary>😇</summary>
  성능, 비용, 기능 및 렌더링 방식 비교

### (1) 성능 비교

| **항목**      | **S3 단독** | **S3 + CloudFront** |
| ------------- | ----------- | ------------------- |
| **TTFB**      | 300ms       | 50ms (캐싱된 경우)  |
| **전송 속도** | 1Mbps       | 5Mbps               |
| **지연 시간** | 200ms       | 50ms                |
| **캐싱 지원** | 없음        | 있음 (TTL: 1시간)   |

---

### (2) 비용 비교

| **항목**             | **S3 단독** | **S3 + CloudFront** |
| -------------------- | ----------- | ------------------- |
| **월간 요청 수**     | 1,000,000건 | 1,000,000건         |
| **월간 데이터 전송** | 10GB        | 10GB                |
| **총 비용**          | $2.50       | $3.00               |

---

### (3) 기능 및 확장성 비교

| **항목**       | **S3 단독** | **S3 + CloudFront**     |
| -------------- | ----------- | ----------------------- |
| **HTTPS 지원** | 제한적      | 쉬움 (CloudFront SSL)   |
| **WAF 지원**   | 지원 안 함  | 지원 가능               |
| **확장성**     | 낮음        | 글로벌 트래픽 분산 가능 |

---

### (4) 렌더링 방식, SEO 최적화 및 속도 비교

| **항목** | **S3 단독**               | **S3 + CloudFront**              |
| -------- | ------------------------- | -------------------------------- |
| **CSR**  | 가능                      | 가능                             |
| **SSG**  | 가능                      | 가능                             |
| **SSR**  | 불가능                    | Lambda@Edge를 사용하면 가능      |
| **SEO**  | CSR은 SEO에 불리          | Lambda@Edge를 활용하면 개선 가능 |
| **속도** | 정적 파일만 서빙으로 빠름 | CloudFront 캐싱으로 더 빠름      |

## 페이지가 화면에 로드되는 순서

1. **리다이렉트** (필요한 경우)
2. **서비스워커 초기화** (서비스워커가 있는 경우)
3. **서비스워커 fetch 이벤트** (서비스워커가 요청을 가로챌 때)
4. **DNS 조회**
5. **TCP 연결**
6. **HTTP 요청**
7. **Early Hints** (사용되는 경우)
8. **서버 응답 처리 및 파싱** (Processing)
9. **페이지 로드** (Load)

[페이지 로드 순서 관련 문서](https://developer.chrome.com/docs/devtools/network/reference?utm_source=devtools&hl=ko#timing-explanation)

</details>
