# Kubernetes 로깅 스택 구성 리포지토리

## 1. 프로젝트 개요

이 리포지토리는 Kubernetes 클러스터의 중앙 집중식 로깅을 위한 **EFK(Elasticsearch, Filebeat, Kibana) 스택**의 구성을 관리한다.

모든 설정은 GitOps 원칙에 따라 버전 관리되며, 공식 Helm 차트의 설정을 재정의(override)하는 `values.yaml` 파일들을 중심으로 구성된다. 이를 통해 로깅 스택의 일관되고 안정적인 배포 및 관리를 목표로 한다.

## 2. 핵심 구성 요소 및 전략

이 리포지토리는 공식 Elastic Helm 차트를 기반으로 로깅 스택을 배포하는 것을 전제로 한다. 각 컴포넌트의 상세 설정은 `helm-values` 디렉토리의 파일들을 통해 제어된다.

-   **Elasticsearch**: 수집된 로그를 저장하고 검색 및 분석 기능을 제공하는 분산 검색 엔진.
-   **Filebeat**: Kubernetes 클러스터의 각 노드에서 컨테이너 로그를 수집하여 Elasticsearch로 전송하는 경량 로그 수집기 (Shipper).
-   **Kibana**: Elasticsearch에 저장된 데이터를 시각화하고 탐색하기 위한 웹 기반 대시보드.

## 3. 디렉토리 구조

```
.
├── helm-values/          # 각 컴포넌트의 Helm Chart values.yaml 파일
│   ├── elasticsearch-values.yaml
│   ├── filebeat-values.yaml
│   └── kibana-values.yaml
│
├── kubernetes/           # 로깅 스택이 배포될 네임스페이스 등 기본 리소스
│   └── namespace.yaml
│
├── manifests/            # (필요 시) Helm 차트로 관리되지 않는 추가 매니페스트
│   ├── elasticsearch/
│   ├── filebeat/
│   ├── kibana/
│   └── logstash/
│
└── dashboards/           # Kibana에 임포트할 대시보드 JSON 파일 (현재 비어있음)
```

-   **`helm-values/`**: 이 리포지토리의 핵심. 각 Elastic 컴포넌트 Helm 차트의 기본 설정을 덮어쓰기 위한 커스텀 `values.yaml` 파일들을 관리한다. **로깅 스택의 모든 설정 변경은 이 디렉토리의 파일 수정을 통해 이루어져야 한다.**
-   **`kubernetes/`**: 로깅 스택이 격리되어 배포될 `logging` 네임스페이스와 같은 기본 리소스를 정의한다.
-   **`manifests/`**: Helm 차트만으로 구성하기 어려운 추가적인 Kubernetes 리소스(예: Ingress, NetworkPolicy 등)를 관리하기 위한 디렉토리다.
-   **`dashboards/`**: 커스텀 Kibana 대시보드의 JSON 정의를 저장하여 버전 관리하기 위한 공간이다.

## 4. 배포 전략 (GitOps)

> **주의:** 이 리포지토리의 구성은 `helm install` 명령어로 직접 설치하기보다, GitOps 워크플로우를 통해 배포되도록 설계되었다.

배포는 `k8s-gitops-configs` 리포지토리에 정의된 Argo CD `Application` 매니페스트에 의해 관리된다.

**워크플로우:**
1.  `k8s-gitops-configs` 리포지토리의 `argocd-apps/logging/` 디렉토리에 있는 `elasticsearch-app.yaml`, `filebeat-app.yaml`, `kibana-app.yaml` 파일들이 각각의 Argo CD 애플리케이션을 정의한다.
2.  각 Argo CD 애플리케이션은 `source` 필드를 통해 공식 Elastic Helm 차트 리포지토리를 바라본다.
3.  동시에 `helm.valueFiles` 필드를 통해 이 `k8s-logging-configs` 리포지토리의 `helm-values/*.yaml` 파일을 참조한다.
4.  Argo CD는 공식 차트의 기본값 위에 이 리포지토리의 `values` 파일을 병합하여 최종 매니페스트를 렌더링하고, 이를 클러스터에 배포 및 동기화한다.

## 5. 로깅 스택 커스터마이징

로깅 스택의 설정을 변경하려면, 이 리포지토리의 `helm-values/` 디렉토리 내에 있는 각 컴포넌트의 `values.yaml` 파일을 수정하고 Git에 푸시하면 된다. Argo CD가 변경 사항을 감지하여 자동으로 클러스터에 적용할 것이다.
