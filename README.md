# Projeto_Kubernets_argoCD


---

#  Projeto: GitOps na Prática

Este projeto é uma imersão prática no universo do **GitOps**, utilizando **GitHub**, **Rancher Desktop com Kubernetes** e **ArgoCD** para implantar uma aplicação de microsserviços de forma automatizada.

Ao seguir este guia, você vivenciará na prática como as empresas modernas operam em ambientes **cloud-native**, tornando os processos de deploy mais auditáveis, previsíveis e versionados.¹

---

##  Objetivo

Executar o conjunto de microsserviços **"Online Boutique"** em um cluster **Kubernetes local** usando **Rancher Desktop**, controlado por **GitOps com ArgoCD**.

---

##  Pré-requisitos

Antes de começar, garanta que seu ambiente local atenda aos seguintes requisitos:

* Git instalado e configurado.
* Docker funcionando localmente.
* Rancher Desktop instalado com o Kubernetes habilitado.
* `kubectl` configurado e funcionando (verifique com `kubectl get nodes`).
* Uma conta no GitHub.

---

##  Parte 1: Deploy com Repositório Público

Nesta primeira parte, faremos o deploy utilizando um repositório público, conforme o fluxo principal do projeto.

---

###  Etapa 1: Preparar o Repositório GitHub

O primeiro passo é criar um repositório Git que servirá como a única fonte da verdade para a configuração da nossa aplicação.

####  Faça o Fork do Repositório Original

Crie uma cópia pessoal do projeto de demonstração **"Online Boutique"** para a sua conta do GitHub.

1. Navegue até: [https://github.com/GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)
2. Clique no botão **Fork** no canto superior direito.

####  Crie um Novo Repositório Público

Crie um novo repositório público na sua conta do GitHub.
O nome recomendado é `NomeRepositorio`.

####  Estruture os Manifestos

Agora, vamos clonar o novo repositório e organizar os arquivos de manifesto do Kubernetes dentro dele.

```bash
git clone https://github.com/SEU-USUARIO/NomeRepositorio.git
cd NomeRepositorio

# Crie a estrutura de diretórios recomendada
mkdir k8s
```

####  Adicione o Manifesto da Aplicação

Copie o arquivo de manifesto `release/kubernetes-manifests.yaml` do repositório que você forkou para a pasta `k8s/` do seu novo repositório, renomeando-o para `online-boutique.yaml`.¹

####  Envie as Alterações para o GitHub

```bash
# Adicione o novo arquivo ao Git
git add k8s/online-boutique.yaml

# Faça o commit da alteração
git commit -m "Adiciona manifesto inicial da Online Boutique"

# Envie para o GitHub
git push origin main
```

---

###  Etapa 2: Instalar o ArgoCD no Cluster Local

Com o cluster Kubernetes rodando no Rancher Desktop, instale o ArgoCD.¹

```bash
# 1. Crie um namespace dedicado para o ArgoCD
kubectl create namespace argocd

# 2. Aplique o manifesto de instalação oficial
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

###  Etapa 3: Acessar a Interface do ArgoCD

Para acessar a interface web do ArgoCD, é necessário redirecionar a porta do serviço.¹

####  Redirecione a Porta

Em um novo terminal, execute o comando abaixo. Ele ficará em execução para manter a conexão ativa.

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

####  Acesse a UI e Faça Login

Abra seu navegador e acesse:
➡️ **[https://localhost:8081](https://localhost:8081)**

* Nome de usuário padrão: `admin`
* Para obter a senha inicial, execute o seguinte comando:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Use a senha retornada para fazer login.

---

###  Etapa 4: Criar a Aplicação no ArgoCD

Agora, vamos conectar o ArgoCD ao nosso repositório Git.

1. Na interface do ArgoCD, clique em **+ NEW APP**.
2. Preencha os seguintes campos:

| Campo            | Valor                                                                                    |
| ---------------- | ---------------------------------------------------------------------------------------- |
| Application Name | `online-boutique`                                                                        |
| Project Name     | `default`                                                                                |
| Sync Policy      | `Manual`                                                                                 |
| Repository URL   | URL HTTPS do seu repositório (`https://github.com/SEU-USUARIO/NomeRepositorio.git`) |
| Revision         | `HEAD`                                                                                   |
| Path             | `k8s`                                                                                    |
| Cluster URL      | `https://kubernetes.default.svc`                                                         |
| Namespace        | `default`                                                                                |

3. Clique em **CREATE** no topo da página.

---

###  Etapa 5: Sincronizar e Acessar a Aplicação

A aplicação aparecerá como **OutOfSync**. Vamos sincronizá-la para fazer o deploy.

####  Sincronize a Aplicação

1. Clique no card da aplicação **online-boutique**.
2. Clique no botão **SYNC**.
3. Na janela de confirmação, clique em **SYNCHRONIZE**.

####  Verifique os Pods

Aguarde até que o status da aplicação mude para **Healthy** e **Synced**.
Em seguida, verifique no terminal se todos os pods estão rodando:

```bash
kubectl get pods -n default
```

####  Acesse o Frontend

O serviço do frontend precisa de um redirecionamento de porta para ser acessado externamente.

```bash
kubectl port-forward svc/frontend-external 8082:80 -n default
```

Abra seu navegador e acesse:
➡️ **[http://localhost:8082](http://localhost:8082)**
Você deverá ver a loja **Online Boutique!**

---

##  Parte 2: Utilizando um Repositório Privado

Trabalhar com repositórios privados é o cenário mais comum em ambientes corporativos.
O processo é quase idêntico ao de um repositório público, com uma etapa adicional: configurar a autenticação para que o ArgoCD tenha permissão de acesso.

---

### Modificações

* **Na Etapa 1.2:** ao criar o repositório `NomeRepositorio`, marque a opção **Private**.
* **Antes da Etapa 4:** adicione uma nova etapa para configurar as credenciais de acesso no ArgoCD.

---

###  Chave de Deploy SSH

Este método concede acesso de leitura a um único repositório, seguindo o princípio do menor privilégio.

#### 1️Gere um par de chaves SSH

```bash
ssh-keygen -t ed25519 -C "argocd-key" -N "" -f ~/.ssh/argocd_deploy_key
```

#### 2️ Adicione a Chave Pública ao GitHub

1. Copie o conteúdo da sua chave pública (`~/.ssh/argocd_deploy_key.pub`).
2. No seu repositório privado do GitHub, vá para **Settings > Deploy keys** e clique em **Add deploy key**.
3. Dê um título (ex: "ArgoCD Key"), cole a chave pública e **não marque** a opção *Allow write access*.

#### 3️ Adicione a Chave Privada ao ArgoCD

1. Na interface do ArgoCD, vá para **Settings > Repositories**.
2. Clique em **Connect Repo using SSH**.
3. Use a URL SSH do seu repositório (ex: `git@github.com:SEU-USUARIO/NomeRepositorio.git`).
4. Cole o conteúdo da sua chave privada (`~/.ssh/argocd_deploy_key`).
5. Clique em **Connect**.

Continue com a **Etapa 4**:
Ao criar a aplicação no ArgoCD, no campo *Repository URL*, certifique-se de usar a URL correspondente ao método de autenticação que você configurou (**SSH** ou **HTTPS**).
O restante do processo é exatamente o mesmo.

---

###  Etapa 6: Testando o Fluxo GitOps

Faça uma alteração no repositório e observe o ArgoCD.

####  Altere o Manifesto

No seu clone local do repositório `NomeRepositorio`, edite o arquivo `k8s/online-boutique.yaml`.
Encontre o **Deployment do productcatalogservice** e mude o campo **réplicas** de `1` para `3`.

####  Envie a Alteração

```bash
git add .
git commit -m "Escala o productcatalogservice para 3 réplicas"
git push
```

#### Observe no ArgoCD

Volte para a interface do ArgoCD.
Em alguns minutos, o status da aplicação mudará para **OutOfSync**.
Clique em **SYNC** e depois em **SYNCHRONIZE**.

O ArgoCD aplicará a mudança, e você verá novos pods do **productcatalogservice** sendo criados.

---

#### Resultado : 

![resultado](resource/Screenshot%20from%202025-10-06%2021-35-39.png)