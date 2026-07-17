## 支持本地缓存文件

开启后会生成类似这样的文件：

```
/mnt/ssd/argo-pipeline/kaniko-cache/
├── <cacheKey1>          # layer tarball
├── <cacheKey1>.json     # manifest
├── <cacheKey2>
├── <cacheKey2>.json
└── ...
```

## 编译与构建镜像

```shell
# - 方式 1：直接编译出二进制
make out/executor
# - 方式 2：构建完整镜像（推荐）
docker build -t your.registry/kaniko-executor:local-cache -f deploy/Dockerfile .
docker push your.registry/kaniko-executor:local-cache
```

## 更新 Argo Workflows 使用自定义镜像

把foundation/workflows/base/sub-template/image-template.yaml 里 Kaniko 容器镜像改成你刚推送的镜像，例如：

```
container:
  image: your.registry/kaniko-executor:local-cache
  args:
    - "--cache=true"
    - "--cache-dir=/cache"
    - "--cache-copy-layers=true"
    - "--cache-run-layers=true"
    - "--no-push-cache"
    # - "--cache-ttl=24h"   # 默认两周，可按需保留
    ...
```

然后同步 Argo CD / kubectl apply-k 对应 overlay。

## 验证

重新触发发布后，检查宿主机：

```
kubectl get pod -n <ns> -l workflows.argoproj.io/workflow=<wf-name> -o wide
# 到该节点上执行
ls -lah /mnt/ssd/argo-pipeline/kaniko-cache/
```

如果层缓存命中，日志里会出现：

```
Checking for locally cached layer /cache/<cacheKey>... Saving layer <cacheKey>
to local cache /cache/<cacheKey>
```

## 指定路径

```
foundation/workflows/base/sub-template/image-template.yaml

        volumeMounts:
          - name: kaniko-secret
            mountPath: /kaniko/.docker
          - name: work
            mountPath: /work
          - name: kaniko-cache
            mountPath: /cache

foundation/workflows/overlays/gprod/pipeline-template.yaml
volumes:
      - name: kaniko-cache
      hostPath:
        path: /mnt/ssd/argo-pipeline/kaniko-cache
        type: DirectoryOrCreate

```
