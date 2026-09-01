# Argo CD Applications

Argo CD Root Application(App-of-Apps)이 관리할 Child `Application` CR 선언 전용 경로입니다.

각 파일은 실제 Workload Manifest를 직접 담지 않고, `apps/`, `platform/`, `cicd/`, `observability/` 아래의 Desired State 경로를 Argo CD에 연결합니다.

신규 Child Application 선언은 이 경로에 추가합니다.
