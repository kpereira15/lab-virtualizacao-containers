# lab-virtualizacao-containers

# Laboratório Prático: Bare Metal, Máquina Virtual, Containers e Balanceamento de Carga

Este roteiro organiza as duas aulas práticas da disciplina utilizando ambientes gratuitos em nuvem diretamente pelo navegador.

---

## 📘 Aula 1: Bare Metal, Máquina Virtual e Isolamento (Containers)
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
Diferente de uma Máquina Virtual (que sobe um sistema operacional completo), um **container** é apenas um processo isolado que compartilha o mesmo kernel do sistema hospedeiro.

1. Verifique a versão atual do kernel da sua VM:
```bash
uname -r
```

2. Crie um ambiente isolado (*namespace* de processos, exatamente como um container faz por baixo dos panos) e liste os processos ativos lá dentro:
```bash
sudo unshare --pid --fork --mount-proc bash
```
```bash
ps aux
```
* **O que observar:** Dentro desse ambiente isolado, você enxerga apenas os processos pertencentes a ele (PID 1), mas se executar `uname -r` novamente, verá que o kernel continua exatamente o mesmo da VM.

3. Para sair do ambiente isolado e voltar ao terminal normal da VM, execute:
```bash
exit
```

---

## 📗 Aula 2: Redes de Containers e Balanceamento de Carga
*Ambiente: **Play with Docker** ([labs.play-with-docker.com](https://labs.play-with-docker.com/)) — já vem com Docker nativo ativo.*

### Como iniciar o ambiente
1. Acesse **[labs.play-with-docker.com](https://labs.play-with-docker.com/)** e faça login com sua conta gratuita do Docker Hub.
2. Clique em **Start** e depois em **+ ADD NEW INSTANCE** no menu lateral esquerdo.

### Etapa 1: Criando a Rede Isolada e o Cluster de Aplicação
1. Crie uma rede interna (bridge) para resolução de nomes (DNS) entre os containers:
```bash
docker network create rede-cluster
```

2. Suba **3 réplicas** da aplicação web conectadas apenas à rede interna:
```bash
docker run -d --name app1 --network rede-cluster nginx:alpine
docker run -d --name app2 --network rede-cluster nginx:alpine
docker run -d --name app3 --network rede-cluster nginx:alpine
```

3. Personalize a página de resposta de cada réplica para identificar qual servidor atendeu a requisição:
```bash
docker exec app1 sh -c 'echo "Resposta do Servidor 01" > /usr/share/nginx/html/index.html'
docker exec app2 sh -c 'echo "Resposta do Servidor 02" > /usr/share/nginx/html/index.html'
docker exec app3 sh -c 'echo "Resposta do Servidor 03" > /usr/share/nginx/html/index.html'
```

### Etapa 2: Configurando e Subindo o Load Balancer (NGINX)
1. Crie o arquivo de configuração `nginx.conf` diretamente no terminal:
```bash
cat << 'EOF' > nginx.conf
events {}
http {
    upstream cluster_web {
        server app1:80;
        server app2:80;
        server app3:80;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://cluster_web;
        }
    }
}
EOF
```

2. Inicie o container do balanceador de carga expondo a porta **`8080`**:
```bash
docker run -d --name load-balancer --network rede-cluster -p 8080:80 -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx:alpine
```

3. Teste o algoritmo **Round Robin** disparando 9 requisições sequenciais:
```bash
for i in {1..9}; do curl -s http://localhost:8080; done
```

### Etapa 3: Alta Disponibilidade (Failover)
1. Simule a queda repentina do segundo servidor (`app2`) e repita as requisições:
```bash
docker stop app2
for i in {1..6}; do curl -s http://localhost:8080; done
```
* **O que observar:** O balanceador detecta a indisponibilidade do `app2` e redistribui automaticamente todo o tráfego entre `app1` e `app3` sem falha para o cliente.
