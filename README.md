# Helm Chart - Release & Rollback

Passos:
1. Criar um exemplo de Chart.
2. Instalar o Chart criado.
3. Atualizar para uma imagem diferente.
4. Rollback da Release para a versao anterior.
5. Desinstalar o Chart.

---
### 1. Criar um Exemplo de Chart
``` bash
% helm create <chart_name>
```

``` bash
% helm create my_application
Creating my_application
```

Delete todos os arquivos contidos em templates:
``` bash
% rm -Rvf my_application/templates/*
my_application/templates/NOTES.txt
my_application/templates/_helpers.tpl
my_application/templates/deployment.yaml
my_application/templates/hpa.yaml
my_application/templates/ingress.yaml
my_application/templates/service.yaml
my_application/templates/serviceaccount.yaml
my_application/templates/tests/test-connection.yaml
my_application/templates/tests
```

Criar dois arquivos de deployment e service:
```bash
% touch my_application/templates/deployment.yaml
% touch my_application/templates/service.yaml  
```

OBSERVAÇÃO: *Chart.yaml* contém metadados sobre nossos Charts e *values.yaml* contém dados configuráveis para nossa definição de recurso do Kubernetes e os valores definidos nesses arquivos podem ser acessados em arquivos de definição de recurso. Podemos acessar os dados definidos nesses arquivos com a ajuda da seguinte sintaxe:
```bash
{{ .Chart.x.y }} and {{ .Values.x.y.z }}
```

Defina o arquivo *values.yaml* conforme abaixo:
``` yaml
replicaCount: 1

image:
  repository: kodekloud/simple-webapp
  pullPolicy: IfNotPresent
  tag: "green"

service:
  name: my-service
  type: NodePort
  port: 8080
  NodePort: 30007

data:
  name: my-deployment
  labels:
    app: nginx
  port:
    number: 8080
    name: http
    protocol: TCP

configmap:
  name: my-config-map
```

Agora popule o arquivo *deployment.yaml* conforme abaixo:
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.data.name }}
  labels:
    app: {{ .Values.data.labels.app }}
spec:
  selector:
    matchLabels:
      app: {{ .Values.data.labels.app }}
  replicas: {{ .Values.replicaCount }}
  template:
    metadata:
      labels:
        app: {{ .Values.data.labels.app }}
    spec:
      containers:
      - name: {{ .Values.data.name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: {{ .Values.data.port.name }}
          containerPort: {{ .Values.data.port.number }}
          protocol: {{ .Values.data.port.protocol }}
```

Faca o mesmo com *service.yaml* conforme abaixo:
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.service.name }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.data.port.number }}
    targetPort: {{ .Values.data.port.number }}
    nodePort: {{ .Values.service.NodePort }}
  selector:
    app: {{ .Values.data.labels.app }}
```

---
### 2. Instalar o Chart criado.
``` bash
% helm install <release_name> <chart_name>
```

``` bash
% helm install my-release my_application --description "My First Install"
NAME: my-release
LAST DEPLOYED: Thu May  5 22:56:49 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Você pode verificar os detalhes da versão com a ajuda com:
##### Historico
``` bash
% helm history <release_name>
```
``` bash
% helm history my-release  
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Thu May  5 22:56:49 2022        deployed        my_application-0.1.0    1.16.0          My First Install
```

##### Get all
``` bash
% helm get all <release_name>
```
``` bash
% helm get all my-release
NAME: my-release
LAST DEPLOYED: Thu May  5 22:56:49 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
null

COMPUTED VALUES:
configmap:
  name: my-config-map
data:
  labels:
    app: nginx
  name: my-deployment
  port:
    name: http
    number: 8080
    protocol: TCP
image:
  pullPolicy: IfNotPresent
  repository: kodekloud/simple-webapp
  tag: green
replicaCount: 1
service:
  NodePort: 30007
  name: my-service
  port: 8080
  type: NodePort

HOOKS:
MANIFEST:
---
# Source: my_application/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30007
  selector:
    app: nginx
---
# Source: my_application/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: my-deployment
        image: "kodekloud/simple-webapp:green"
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
```

---
### 3. Atualizar para uma imagem diferente.

A maneira simples de fazer isso será alterar seus valores no arquivo values.yaml e usar o seguinte comando:
``` bash
% helm upgrade <release_name> <chart_name> —description “My custom message”
```

``` bash
% helm upgrade my-release my_application --description "Update Version."
Release "my-release" has been upgraded. Happy Helming!
NAME: my-release
LAST DEPLOYED: Thu May  5 23:02:46 2022
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

---
### 4. Rollback da Release para a versao anterior.

Para fazer isso, primeiro precisamos verificar as revisões anteriores. O comando para isso é:

``` bash
% helm history my-release                                               
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Thu May  5 22:56:49 2022        superseded      my_application-0.1.0    1.16.0          My First Install
2               Thu May  5 23:02:46 2022        deployed        my_application-0.1.0    1.16.0          Update Version.
```
Podemos notar que temos números de “REVISÃO” para nossas instalações e atualizações. Podemos usá-lo para reverter para nossas especificações anteriores. Para isso, vamos executar o comando:
``` bash
% helm rollback <release_name> <revision_number>
```
``` bash
% helm rollback my-release 1               
Rollback was a success! Happy Helming!

% helm history my-release                  
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Thu May  5 22:56:49 2022        superseded      my_application-0.1.0    1.16.0          My First Install
2               Thu May  5 23:02:46 2022        superseded      my_application-0.1.0    1.16.0          Update Version. 
3               Thu May  5 23:05:24 2022        deployed        my_application-0.1.0    1.16.0          Rollback to 1
```

---
### 5. Desinstalar o Chart.