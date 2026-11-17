# lab-virtualizacao-containers

# Laboratório Prático: Bare Metal, Máquina Virtual, Containers e Balanceamento de Carga

Este roteiro organiza as duas aulas práticas da disciplina utilizando ambientes gratuitos em nuvem diretamente pelo navegador.

---

## 📘 Prática 1: Bare Metal, Máquina Virtual e Isolamento (Containers)
*Ambiente: **GitHub Codespaces** (não exige instalação de ferramentas extras).*

### Como iniciar o ambiente
1. No topo desta página, clique no botão verde **`<> Code`**.
2. Selecione a aba **`Codespaces`** e clique em **`Create codespace on main`**.
3. Aguarde o terminal carregar na parte inferior da tela.

### Etapa 1: Investigando o Servidor Físico (Bare Metal) vs. Máquina Virtual
Execute um comando por vez no terminal para identificar o hardware físico subjacente, o hypervisor e os recursos isolados para a sua VM:

```bash
lscpu | grep -E "Model name|Hypervisor|Virtualization"
```
```bash
free -h
```
```bash
lsblk
```
* **O que observar:** O processador físico pertence ao servidor *Bare Metal* no datacenter, enquanto o *Hypervisor* (Microsoft) entrega uma fatia isolada de vCPUs, memória RAM e discos virtuais para a sua VM.

### Etapa 2: Compartilhamento de Kernel e Isolamento de Processos (Base dos Containers)
Diferente de uma Máquina Virtual (que sobe um sistema operacional completo), um **container** é um conjunto de processos isolados por *namespaces* que compartilha o mesmo kernel do sistema hospedeiro.

1. Verifique a versão do kernel compartilhada pelo ambiente:
```bash
uname -r
```

2. Liste os processos ativos no seu ambiente isolado:
```bash
ps aux
```
* **O que observar:** Existem pouquíssimos processos em execução (apenas o seu terminal e os serviços do editor), comprovando que o ambiente já roda dentro de um container isolado da máquina virtual hospedeira.

3. Inspecione os *namespaces* (mecanismos de isolamento do kernel Linux) atribuídos ao seu ambiente:
```bash
ls -l /proc/self/ns
```
* **O que observar:** Cada identificador (`pid`, `net`, `mnt`, `uts`, `ipc`) representa uma camada de isolamento criada pelo kernel Linux para separar a sua sessão sem precisar de uma nova Máquina Virtual.

---

## 📗 Prática 2: Redes de Containers e Balanceamento de Carga
*Ambiente: **Play with Docker** ([labs.play-with-docker.com](https://labs.play-with-docker.com/)) — já vem com Docker nativo ativo.*

### Como iniciar o ambiente
1. Acesse **[labs.play-with-docker.com](https://labs.play-with-docker.com/)** e faça login com sua conta gratuita do Docker Hub.
2. Clique em **Start** e depois em **+ ADD NEW INSTANCE** no menu lateral esquerdo.

### Etapa 1: Criando a Rede Isolada e o Cluster de Aplicação
1. Crie uma rede interna (*bridge*) para resolução de nomes (DNS) entre os containers:
```bash
docker network create rede-cluster
```

2. Suba **3 réplicas** da aplicação web conectadas apenas à rede interna:
```bash
docker run -d --name app1 --network rede-cluster nginx:alpine
docker run -d --name app2 --network rede-cluster nginx:alpine
docker run -d --name app3 --network rede-
