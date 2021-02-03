# MCP - Multi-Cloud Container Platform
## O que é Multi-Cloud Container Platform (MCP)?
A Multi-Cloud Container Platform (MCP) da HUAWEI CLOUD foi desenvolvida, com base em anos de experiência no campo de contêineres em nuvem e tecnologias avançadas da federação de clusters, para fornecer soluções conteinerizadas em multi-nuvem e nuvem híbrida, possibilitando o gerenciamento unificado de clusters entre nuvens e a implantação unificada e distribuição de tráfego de aplicativos em clusters. Ela não só resolve a recuperação de desastres em várias nuvens, mas também desempenha um papel importante no compartilhamento de tráfego, na dissociação do armazenamento de dados e processamento de serviços, na dissociação de ambientes de desenvolvimento e produção e na alocação flexível de recursos computacionais. 
Para saber mais visite: https://support.huaweicloud.com/en-us/usermanual-mcp/mcp_01_0001.html

![](https://support.huaweicloud.com/en-us/productdesc-mcp/en-us_image_0228801720.png "Multi-Cloud Container Platform")


Este documento tem o propósito de instruir os usuários do MCP em como importar cluster de Kubernetes de nuvens públicas, como AWS e GCP. Outras ofertas de nuvens públicas não foram testadas até o momento da criação deste documento.

O processo de importação de um cluster kubernetes no MCP consiste em utilizar o arquivo de configuração kubeconfig do cluster kubernetes que deseja federar com o MCP.
Veja o passo a passo [aqui](https://support.huaweicloud.com/en-us/usermanual-mcp/mcp_01_0007.html "Huawei Cloud Support Page")

Maiores informações sobre o arquivo de configuração kubeconfig podem ser acessadas [aqui](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/ "Kubernetes Documents Homepage"). 


### Vamos começar!
*Neste tutorial estamos considerando que os clusters GKE e/ou AKS já existem e vocês têm acesso ao kube-apiserver como administrador.*

Para que o MCP possa ter acesso ao cluster kubernetes na AWS/GCP é necessário criar um ServiceAccount, pois o arquivo kubeconfig padrão destes provedores utilizam tokens por padrão para se autenticar no kube-apiserver, como demonstrado nos exemplos abaixo:

Arquivo kubeconfig da AWS EKS:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiB=
    server: https://xxx1111wwwzzz.gr7.us-east-2.eks.amazonaws.com
  name: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
contexts:
- context:
    cluster: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
    user: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
  name: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
- context:
    cluster: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
    user: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
  name: aws-cluster
current-context: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
kind: Config
preferences: {}
users:
- name: arn:aws:eks:us-east-2:111111111111:cluster/eks-mcp-huawei
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - --region
      - us-east-2
      - eks
      - get-token
      - --cluster-name
      - eks-mcp-huawei
      command: aws
      env: null
      provideClusterInfo: false
```

Arquivo kubeconfig da GCP GKE:
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRV==
    server: https://127.0.0.1
  name: gke_huawei-mcp_us-central1-c_gcp-game-of-pods
contexts:
- context:
    cluster: gke_huawei-mcp_us-central1-c_gcp-game-of-pods
    user: gke_huawei-mcp_us-central1-c_gcp-game-of-pods
  name: gke_huawei-mcp_us-central1-c_gcp-game-of-pods
current-context: gke_huawei-mcp_us-central1-c_gcp-game-of-pods
kind: Config
preferences: {}
users:
- name: gke_huawei-mcp_us-central1-c_gcp-game-of-pods
  user:
    auth-provider:
      config:
        cmd-args: config config-helper --format=json
        cmd-path: /usr/lib/google-cloud-sdk/bin/gcloud
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
```

Como podemos notar nos arquivos kubeconfig, são utilizados os utilitários de linha de comando (CLI) para gerar e atualizar um token para acesso ao cluster kubernetes. Entretanto o MCP não possui estes utilitários em seu ambiente e tampouco as credenciais utilizadas para acessar recursos do provedor de nuvem. Então como fazer para o MCP acessar meu cluster?

Vamos ao passo-a-passo: 

Criar um ServiceAccount e gerar um token de acesso e utilizar esse token em nosso arquivo kubeconfig.
***

1. Criar um ServiceAccount:

Para isso precisamos criar um manifest yaml "my-mcp-serviceaccount.yaml", como por exemplo no diretório "/home/fabio":

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-user
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-user-role
rules:
  - apiGroups:
      - '*'
    resources:
      - '*'
    verbs:
      - '*'
  - nonResourceURLs:
      - '*'
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-user-role-binding
subjects:
  - kind: ServiceAccount
    name: my-user
    namespace: default
roleRef:
  kind: ClusterRole
  name: my-user-role
  apiGroup: rbac.authorization.k8s.io
```

Agora você deve executar o comando abaixo para criar os recursos descritos no manifest apresentado acima:

``` 
kubectl apply -f /home/fabio/my-service-account.yaml 
```

2. Obter um token

Para obter o token devemos digitar o seguinte comando:

  ```
kubectl get secret -n default `kubectl get secret -n default | grep my-user | awk '{print $1}'` -oyaml | grep token: | awk '{print $2}' | base64 -d
```

O comando acima deve gerar um token na console como no exemplo 👇🏽:

```
fabiobo@fabio-pc:~$ kubectl get secret -n default `kubectl get secret -n default | grep my-user | awk '{print $1}'` -oyaml | grep token: | awk '{print $2}' | base64 -d
eyJhbGciOiJSUzI1NiIsImtpZCI6IjIzVWdBNWF5WjlRVzVHUm1IVU16Z05YdnRxc2h4aDh5c2tpS1BEdzlzN28ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im15LXVzZXItdG9rZW4tdDRkZDQiLCJrdWJlcm5ldGVzLmlvL3
```

3. Criar e editar o arquivo kubeconfig

Agora devemos criar e editar nosso arquivo kubeconfig utilizando o usuário e conta de serviço que criamos nos passos anteriores.

Você pode pegar um dos exemplo JSON ou YAML aqui no repo e usar como modelo para seu arquivo kubeconfig, abaixo segue um exemplo de kubeconfig YAML:

```yaml
  ---
kind: Config
apiVersion: v1
preferences: {}
clusters:
- name: gcp-cluster-1
  cluster:
    server: https://127.0.0.1:6443 # change with the kubernetes API server URL for access 
    insecure-skip-tls-verify: true
users:
- name: user 
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6Ik5lVzJxbE9hVzZOWmwzb3ZkQTA2Y #replace it with the output of step 2 
contexts:
- name: cluster-1
  context:
    cluster: gcp-cluster-1
    user: user
current-context: user@cluster-1
```

4. Importar este aquivo em seu cluster MCP

As instruções para importar o kubeconfig para o MCP estão na página de suporte da Huawei Cloud, a qual você pode acessar [aqui](https://support.huaweicloud.com/en-us/usermanual-mcp/mcp_01_0007.html "Huawei Cloud Support").
