下面是执行 Maven 构建的模板：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: library-maven
spec:
  templates:
  - name: maven
    inputs:
      parameters:
        - name: target
          default: "package"
        - name: basedir
          default: "."
    container:
      image: maven:3.8.7-openjdk-18
      args:
        - mvn
        - "{{inputs.parameters.target}}"
        - -f
        - "{{inputs.parameters.basedir}}/pom.xml"
      workingDir: /work
      volumeMounts:
        - name: work
          mountPath: /work

  volumeClaimTemplates:
    - metadata:
        name: work
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 64Mi
EOF
```
