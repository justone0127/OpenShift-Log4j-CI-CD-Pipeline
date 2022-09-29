# OpenShift-Log4j-CI-CD-Pipeline

### 1. 구성 준비

### 1.1) 사전 준비 사항

- **OpenShift 4.x Cluster**

-  **OpenShift Pipeline Operator** 
- **Red Hat Advanced Cluster for Security Operator**
- **Red Hat Quay Operator**
- **Sample Application** : https://github.com/justone0127/log4shell-vulnerable-app.git



### 2. Log4J 취약점 정책 설정 (RHACS)

### 2.1) Log4Shell vulnerability Policy 적용

 애플리케이션이 배포 될 때 해당 애플리케이션에 `Log4Shell: log4j Remote Code Execution vulnerability`가 포함된 경우 배포되지 않도록 time enforcement 정책을 적용합니다.

- Log4Shell: log4j Remote Code Execution vulnerability Policy 수정

  - **RHACS 콘솔 접속 > Platform Configuration > Policy Management > Log4Shell: log4j Remote Code Execution vulnerability Policy 검색 **

    ![00_policy](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\00_policy.png)
    
  - **Edit > Response method : Inform and enforce 선택 > Configure enforcement behavior : Deploy 활성화**

    ![04_log4j_policy_settings_rhacs_3.71](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\04_log4j_policy_settings_rhacs_3.71.png)

- Log4Shell Policy 활성화 확인

  ![05_log4jshell_policy](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\05_log4jshell_policy.png)

### 3. CI/CD Pipeline 구성

### 3.1) 프로젝트 생성

```bash
oc new-project log4j-test
```

### 3-2) Application 배포

- 콘솔 접속 > 개발자 콘솔 > Import from Git > 내용 입력 > Pipelines 부분에 Add pipeline 체크

  - Git Repo URL : https://github.com/justone0127/log4shell-vulnerable-app.git

  - Pipelines : Add pipeline Check

    ![01_import_from_git](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\01_import_from_git.png)

### 3.3) Pipeline 생성

- 파이프라인 예시

  ![02_cicd_pipeline](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\02_cicd_pipeline.png)

- Stackrox Tasks 추가

  >  RHACS를 통해 Image Scan 및 Image Check를 수행하기 위해 다음 Tasks를 Pipeline에서 사용할 수 있도록 추가합니다.

  - image-scan-tasks.yaml

    ```yaml
    ---
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
      annotations:
        tekton.dev/categories: Security
        tekton.dev/displayName: Scan an image with StackRox/RHACS
        tekton.dev/pipelines.minVersion: 0.18.0
        tekton.dev/platforms: linux/amd64
        tekton.dev/tags: security
      name: stackrox-image-scan
      labels:
        app.kubernetes.io/version: '0.1'
    spec:
      description: >-
        Scan an image with StackRox/RHACS This tasks allows you to return full
        vulnerability scan results for an image in JSON, CSV, or Pretty format. It's
        a companion to the stackrox-image-check task, which checks an image against
        build-time policies.
      params:
        - description: |
            Secret containing the address:port tuple for StackRox Central
            (example - rox.stackrox.io:443)
          name: rox_central_endpoint
          type: string
        - description: Secret containing the StackRox API token with CI permissions
          name: rox_api_token
          type: string
        - description: |
            Full name of image to scan (example -- gcr.io/rox/sample:5.0-rc1)
          name: image
          type: string
        - default: json
          description: Output format (json | csv | pretty)
          name: output_format
          type: string
        - default: 'false'
          description: |
            When set to `"true"`, skip verifying the TLS certs of the Central
            endpoint.  Defaults to `"false"`.
          name: insecure-skip-tls-verify
          type: string
      steps:
        - env:
            - name: ROX_API_TOKEN
              valueFrom:
                secretKeyRef:
                  key: rox_api_token
                  name: $(params.rox_api_token)
            - name: ROX_CENTRAL_ENDPOINT
              valueFrom:
                secretKeyRef:
                  key: rox_central_endpoint
                  name: $(params.rox_central_endpoint)
          image: >-
            docker.io/centos@sha256:a1801b843b1bfaf77c501e7a6d3f709401a1e0c83863037fa3aab063a7fdb9dc
          name: rox-image-scan
          resources: {}
          script: |
            #!/usr/bin/env bash
            set +x
            export NO_COLOR="True"
            curl -s -k -L -H "Authorization: Bearer $ROX_API_TOKEN" \
              "https://$ROX_CENTRAL_ENDPOINT/api/cli/download/roxctl-linux" \
              --output ./roxctl  > /dev/null; echo "Getting roxctl"
            chmod +x ./roxctl > /dev/null
            ./roxctl image scan \
              $( [ "$(params.insecure-skip-tls-verify)" = "false" ] && \
              echo -n "--insecure-skip-tls-verify") \
              -e "$ROX_CENTRAL_ENDPOINT" --image "$(params.image)" \
              --format "$(params.output_format)"
    ```

    - `insecure-skip-tls-verify` 파라미터의 기본값은 `true`로 TLS verify 체크를 스킵할 경우 `false`로 변경하여 사용

  - image-check.yaml

    ```yaml
    ---
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
      name: stackrox-image-check
      labels:
        app.kubernetes.io/version: "0.1"
      annotations:
        tekton.dev/tags: security
        tekton.dev/categories: Security
        tekton.dev/displayName: "Policy check an image with StackRox/RHACS"
        tekton.dev/platforms: "linux/amd64"
        tekton.dev/pipelines.minVersion: "0.18.0"
    spec:
      description: >-
        Policy check an image with StackRox/RHACS
    
        This tasks allows you to check an image against build-time policies
        and apply enforcement to fail builds.  It's a companion to the
        stackrox-image-scan task, which returns full vulnerability scan
        results for an image.
      params:
        - name: rox_central_endpoint
          type: string
          description: |
            Secret containing the address:port tuple for StackRox Central)
            (example - rox.stackrox.io:443)
        - name: rox_api_token
          type: string
          description: Secret containing the StackRox API token with CI permissions
        - name: image
          type: string
          description: |
            Full name of image to scan (example -- gcr.io/rox/sample:5.0-rc1)
        - name: insecure-skip-tls-verify
          type: string
          description: |
            When set to `"true"`, skip verifying the TLS certs of the Central
            endpoint.  Defaults to `"false"`.
          default: "false"
      results:
        - name: check_output
          description: Output of `roxctl image check`
      steps:
        - name: rox-image-check
          image: docker.io/centos@sha256:a1801b843b1bfaf77c501e7a6d3f709401a1e0c83863037fa3aab063a7fdb9dc
          env:
            - name: ROX_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: $(params.rox_api_token)
                  key: rox_api_token
            - name: ROX_CENTRAL_ENDPOINT
              valueFrom:
                secretKeyRef:
                  name: $(params.rox_central_endpoint)
                  key: rox_central_endpoint
          script: |
            #!/usr/bin/env bash
            set +x
            curl -s -k -L -H "Authorization: Bearer $ROX_API_TOKEN" \
              "https://$ROX_CENTRAL_ENDPOINT/api/cli/download/roxctl-linux" \
              --output ./roxctl  \
              > /dev/null
            chmod +x ./roxctl  > /dev/null
            ./roxctl image check \
              $( [ "$(params.insecure-skip-tls-verify)" = "false" ] && \
              echo -n "--insecure-skip-tls-verify") \
              -e "$ROX_CENTRAL_ENDPOINT" --image "$(params.image)"
    ```
    
- `insecure-skip-tls-verify` 파라미터의 기본값은 `true`로 TLS verify 체크를 스킵할 경우 `false`로 변경하여 사용
    
- stackrox-secret 설정 파일
  
  ```yaml
    apiVersion: v1
    stringData:
      rox_central_endpoint: "{{ central_addr }}:{{ central_port }}"
      # The address:port tuple for StackRox Central (example - rox.stackrox.io:443)
      rox_api_token: "{{ rox_api_token }}"
      # StackRox API token with CI permissions
      # Refer to https://help.stackrox.com/docs/use-the-api/#generate-an-access-token
    kind: Secret
    metadata:
      name: stackrox-secret
    type: Opaque
    ```
  
  - rox_central_endpoint : central-rhacs-operator.apps.ocp4.sandbox824.opentlc.com:443
    - rox_api_token : Integrations > StackRox API Token 생성 후 해당 내용 복사
  
- quay-secret 설정 파일 생성
  
  > Pipeline 생성 시 추가로 quay-secret 정보가 포함된 파일을 생성한다.
  
- Tekton의 가장 큰 장점은 재 사용성 이므로 각각의 Tasks를 기반으로 필요한 변수는 파라미터로 값을 전달 받아서 실행 할 수 있다.
  
- CI/CD Pipeline Sample YAML

  ```yaml
  apiVersion: tekton.dev/v1beta1
  kind: Pipeline
  metadata:
    labels:
      app.kubernetes.io/instance: log-4-shell-vulnerable-app-git
      app.kubernetes.io/name: log-4-shell-vulnerable-app-git
      operator.tekton.dev/operand-name: openshift-pipelines-addons
      pipeline.openshift.io/strategy: docker
      pipeline.openshift.io/type: openshift
    name: log-4-shell-vulnerable-app-git
  spec:
    params:
      - default: log-4-shell-vulnerable-app-git
        name: APP_NAME
        type: string
      - default: 'https://github.com/justone0127/log4shell-vulnerable-app.git'
        name: GIT_REPO
        type: string
      - default: main
        name: GIT_REVISION
        type: string
      - default: >-
          image-registry.openshift-image-registry.svc:5000/log4j-test/log-4-shell-vulnerable-app-git
        name: IMAGE_NAME
        type: string
      - default: .
        name: PATH_CONTEXT
        type: string
      - default: >-
          rh-registry-quay-quay.apps.ocp4.sandbox824.opentlc.com/admin/log-4-shell-vulnerable-app-git:latest
        name: QUAY_REGISTRY
        type: string
    tasks:
      - name: fetch-repository
        params:
          - name: url
            value: $(params.GIT_REPO)
          - name: revision
            value: $(params.GIT_REVISION)
          - name: deleteExisting
            value: 'true'
        taskRef:
          kind: ClusterTask
          name: git-clone
        workspaces:
          - name: output
            workspace: workspace
      - name: build
        params:
          - name: IMAGE
            value: $(params.IMAGE_NAME)
          - name: TLSVERIFY
            value: 'false'
          - name: CONTEXT
            value: $(params.PATH_CONTEXT)
        runAfter:
          - fetch-repository
        taskRef:
          kind: ClusterTask
          name: buildah
        workspaces:
          - name: source
            workspace: workspace
      - name: deploy
        params:
          - name: SCRIPT
            value: oc rollout status dc/$(params.APP_NAME)
        runAfter:
          - stackrox-image-check
        taskRef:
          kind: ClusterTask
          name: openshift-client
      - name: skopeo-copy
        params:
          - name: srcImageURL
            value: 'docker://$(params.IMAGE_NAME)'
          - name: destImageURL
            value: 'docker://$(params.QUAY_REGISTRY)'
          - name: srcTLSverify
            value: 'false'
          - name: destTLSverify
            value: 'false'
        runAfter:
          - build
        taskRef:
          kind: ClusterTask
          name: skopeo-copy
        workspaces:
          - name: images-url
            workspace: workspace
      - name: stackrox-image-scan
        params:
          - name: rox_central_endpoint
            value: stackrox-secret
          - name: rox_api_token
            value: stackrox-secret
          - name: image
            value: $(params.QUAY_REGISTRY)
          - name: output_format
            value: csv
          - name: insecure-skip-tls-verify
            value: 'false'
        runAfter:
          - skopeo-copy
        taskRef:
          kind: Task
          name: stackrox-image-scan
      - name: stackrox-image-check
        params:
          - name: rox_central_endpoint
            value: stackrox-secret
          - name: rox_api_token
            value: stackrox-secret
          - name: image
            value: $(params.QUAY_REGISTRY)
          - name: insecure-skip-tls-verify
            value: 'false'
        runAfter:
          - stackrox-image-scan
        taskRef:
          kind: Task
          name: stackrox-image-check
    workspaces:
      - name: workspace
  ```

- 파이프라인 생성의 경우 기본으로 체크한 파이프라인에 필요한 Tasks를 GUI 환경에서 추가하여 생성해도 되고, 위와 같이 YAML 형태로 생성할 수 있다.

  

### 4. Pipeline 실행

파이프라인을 수행하게 되면, 마지막 `image-check` 단계에서 파이프라인이 실패하는 것을 볼 수 있습니다.

![06_pipeline_result](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\06_pipeline_result.png)

상세하게 로그를 확인해보면, 다음과 같이 `Log4Shell: log4j Remote Code  Execution vulnerability ` 관련 취약점 및 `Spring4Shell `  취약점 등이 검증된 것을 확인 할 수 있습니다.

![07_rhacs_image_check](C:\Works\01_자료\01_OCP\05_OCP_Demo_hyou\OCP_4.10_CICD_RHACS_Log4J_Pipeline\07_rhacs_image_check.png)

이는 RHACS에서 취약점이 발견되면 배포가 되지 않도록 Policy를 적용했기 때문입니다.

이처럼, OpenShift 환경에서는 RHACS를 통해 Image 취약점을 스캔하는 과정을 Tekton 기반의 OpenShift Pipeline과 통합하여 사용할 수 있습니다.
