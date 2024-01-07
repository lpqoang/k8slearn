01

```yaml
# 01-task-git-clone.yaml
apiVersion: tekton.dev/v1betal
kind: Task
metadata:
  name: git-clone
spec:
  description: Clone the code to the workspace
  params:
    - name: url
      type: string
      description: git url to clone
      default: ""
    - name: branch
      type: string
      description: git branch to checkout
      default: "master"
  workspaces:
    - name: source
      description: The git repo will be cloned onto the volume backing this workspace
  steps:
    - name: git-clone
      image: alpine/git
      script: git clone -b $(params.branch) -v $(params.url) $(workspaces.source.path)/source
```

02

```yaml
# 02-task-source-build.yaml
apiVersion: tekton.dev/v1betal
kind: Task
metadata:
  name: build-to-package
spec:
  description: build application and package the files to image
  workspaces:
    - name: source
    description: The git repo that cloned onto the volume backing this workspace
  steps:
    - name: build
      image: maven:3.8-openjdk-11-slim
      workingDir: $(workspaces.source.path)/source
      volumeMounts:
        - name: m2
          mountPath: /root/.m2
      script: mvn clean install
  volumes:
    - name: m2
      persistentVolumeClaim:
        claimName: maven-cache
```

03

[GitHub - GoogleContainerTools/kaniko: Build Container Images In Kubernetes](https://github.com/GoogleContainerTools/kaniko)生成认证信息

echo -n USER:PASSWORD | base64

{
  "auths": {
    "artprod.company.com": {
      "auth": "xxxxxxxxxxxxxxx"
    }
  }
}

kubectl create secret generic docker-config --from-file=<path to .docker/config.json>

```yaml
# 03-task-build-image.yaml

apiVersion: tekton.dev/v1betal
kind: Task
metadata:
  name: image-build-and-push
spec:
  description: package the application files to image
  params:
    - name: dockerfile
      description: The path to the dockerfile to build (relative to the context)
      default: Dockerfile
    - name: image-url
      description: Url of image repository
    - name: image-tag
      description: Tag to apply to the built image
      default: latest
  workspaces:
    - name: source
    - name: docker-config
      # Secret resource which contains identity to image registry
      mountPath: /kaniko/.docker
  steps:
    - name: image-build-and-push
      image: gcr.io/kaniko-project/executor:debug
      securityContext:
        runAsUser: 0
      env:
        - name: DOCKER_CONFIG
          value: /kaniko/.docker
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(params.dockerfile)
        - --context=$(workspaces.source.path)/source
        #- --no-push
        # 标记非安全仓库
        - --insecure-registry
        - --destination=$(params.image-url):$(params.image-tag)
```

04

```yaml
# 04-pipeline-source-to-image.yaml
apiVersion: tekton.dev/v1betal
kind: Pipeline
metadata:
  name: source-to-image
spec:
  params:
    - name: git-url
    - name: pathToContext
      description: The path to the build context, used by Kaniko
      default: .
    - name: image-url
      description: Url of image repository
    - name: image-tag
      description: Tag to apply to the built image
  workspaces:
    - name: codebase
    - name: docker-config
  tasks:
    - name: git-clone
      taskRef:
        name: gie-clone
      params:
        - name: url
          value: "$(params.git.url)"
      workspaces:
        - name: source
          workspace: codebase
    - name: build-to-package  
      taskRef: 
        name: build-to-package
      workspaces:
        - name: source
          workspace: codebase
      runAfter:
        - git-clone
    - name: image-build-and-push
      taskRef:
        name: image-build-and-push
      params:
        - name: image-url
          value: "$(params.image-url)"
        - name: image-tag
          value: "$(params.image-tag)"
      workspaces:
        - name: source
          workspace: codebase
      runAfter:
        - build-to-package
```

05

```yaml
# 05-piplinerun-source-to-image.yaml
apiVersion: tekton.dev/v1betal
kind: PipelineRun
metadata:
  name: code-to-image-push
spec:
  piplineRef:
    name: source-to-image
  params:
    - name: git-url
      value: http:xxxx
    - name: image-url
      value: ikubernetes/spring-boot-helloworld
    - name: image-tag
      value: latest
  workspaces:
    - name: codebase
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: nfs-client
    - name: docker-config
      secret:
        secretName: docker-config
```

tkn task list

tkn pipeline list
