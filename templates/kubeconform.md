[Kubeconform](https://github.com/yannh/kubeconform) 可以校验 Kubernetes 资源 YAML 文件的格式：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: library-validation
spec:
  templates:
  - name: kubeconform
    inputs:
      parameters:
        - name: target
          default: "."
        - name: schema1
          default: 'https://ghproxy.com/https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master/[[.NormalizedKubernetesVersion]]-standalone[[.StrictSuffix]]/[[.ResourceKind]][[.KindSuffix]].json'
        - name: schema2
          default: 'https://ghproxy.com/https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/[[.Group]]/[[.ResourceKind]]_[[.ResourceAPIVersion]].json'
    container:
      args:
        - -schema-location
        - "{{inputs.parameters.schema1}}"
        - -schema-location
        - "{{inputs.parameters.schema2}}"
        # - -ignore-missing-schemas=true
        - -strict
        - -delims=[[,]]
        - -summary
        - -output=json
        - -n=18
        - "{{inputs.parameters.target}}"
      command:
        - /kubeconform
      # image: ghcr.io/yannh/kubeconform:master
      # see also https://github.com/yannh/kubeconform/pull/161
      image: surenpi/kubeconform@sha256:2371d1e73e5c1f53f9311ba3a80a5094d7b82eb7a0478ccfa507997eff798cbb
      workingDir: /work
      volumeMounts:
        - name: work
          mountPath: /work
```
