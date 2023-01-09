下面是执行 SonarQube 扫描的模板：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: library-sonarqube
spec:
  templates:
  - name: scan
    inputs:
      parameters:
        - name: projectKey
        - name: basedir
          default: "."
    container:
      image: sonarsource/sonar-scanner-cli:4.8.0
      args:
        - -Dsonar.projectBaseDir=/work/{{inputs.parameters.basedir}}
        - -Dsonar.projectKey={{inputs.parameters.projectKey}}
        - -Dsonar.go.coverage.reportPaths=coverage.out
      env:
        - name: SONAR_HOST_URL
          value: http://10.121.218.184:30008/
        - name: SONAR_TOKEN
          value: sqa_15cdc7a6ef5dabdae005baadd2d15f8689b3932e
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
