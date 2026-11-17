# lab-virtualizacao-containers

# Laboratório Prático: Virtualização, Containers e Balanceamento de Carga

Este ambiente roda integralmente na nuvem via **GitHub Codespaces**, com Docker pré-instalado.

## 🚀 Como iniciar o ambiente de aula
1. No topo desta página, clique no botão verde **`<> Code`**.
2. Selecione a aba **`Codespaces`** e clique em **`Create codespace on main`**.
3. Aguarde o ambiente carregar até que o terminal esteja pronto para receber comandos.

---

## 📘 Aula 1: Bare Metal, Máquina Virtual e Containers

### Etapa 1: Investigando a Máquina Virtual e o Bare Metal
Execute os comandos abaixo no terminal para identificar o processador virtualizado, o hypervisor subjacente e os recursos alocados à sua VM no servidor físico:
```bash
lscpu | grep -E "Model name|Hypervisor|Virtualization"
free -h
lsblk
```

### Etapa 2: VM vs. Containers na Prática
1. **Compartilhamento de Kernel:** Compare a versão do kernel da VM hospedeira com o kernel enxergado por um container Alpine:
```bash
uname -r
docker run --rm alpine uname -r
```

2. **Isolamento de Processos:** Suba um servidor web em container e observe como o processo aparece tanto dentro do container quanto na VM hospedeira:
```bash
docker run -d --name web-teste -p 8080:80 nginx:alpine
docker top web-teste
ps aux | grep nginx
```

3. **Consumo de Recursos:** Verifique o consumo de memória RAM e CPU do container em execução:
```bash
docker stats --no-stream
```

### Etapa 3: Desafio de Fixação
Suba dois containers distintos na mesma VM:
* Um servidor **NGINX** (`nginx:alpine`) respondendo na porta **`8081`**.
* Um servidor **Apache** (`httpd:alpine`) respondendo na porta **`8082`**.

Personalize a página inicial de cada um e valide o acesso utilizando `curl http://localhost:8081` e `curl http://localhost:8082`.

---

## 📗 Aula 2: Redes de Containers e Balanceamento de Carga

> **Limpeza prévia:** Caso exista algum container da Aula 1 em execução, remova-o antes de iniciar:
> ```bash
> docker rm -f $(docker ps -aq)
> ```

### Etapa 1: Criando a Rede Isolada e o Cluster de Aplicação
1. Crie uma rede interna (bridge) para que os containers se comuniquem pelo nome (DNS interno do Docker):
```bash
docker network create rede-cluster
```

2. Suba **3 réplicas** da aplicação conectadas apenas à rede interna:
```bash
docker run -d --name app1 --network rede-cluster nginx:alpine
docker run -d --name app2 --network rede-cluster nginx:alpine
docker run -d --name app3 --network rede-cluster nginx:alpine
```

3. Personalize o HTML de cada réplica para identificar qual servidor respondeu a cada requisição:
```bash
docker exec app1 sh -c 'echo "Resposta do Servidor 01" > /usr/share/nginx/html/index.html'
docker exec app2 sh -c 'echo "Resposta do Servidor 02" > /usr/share/nginx/html/index.html'
docker exec app3 sh -c 'echo "Resposta do Servidor 03" > /usr/share/nginx/html/index.html'
```

### Etapa 2: Subindo o Load Balancer (NGINX)
1. Inicie o container do balanceador conectado à `rede-cluster`, expondo apenas a porta **`8080`** e utilizando o arquivo `aula-02/nginx.conf`:
```bash
docker run -d --name load-balancer --network rede-cluster -p 8080:80 -v $(pwd)/aula-02/nginx.conf:/etc/nginx/nginx.conf:ro nginx:alpine
```

2. Teste o algoritmo **Round Robin** disparando 9 requisições sequenciais e observe a distribuição igualitária entre os nós:
```bash
for i in {1..9}; do curl -s http://localhost:8080; done
```

### Etapa 3: Alta Disponibilidade (Failover) e Pesos (Weighted)
1. **Simulação de Falha:** Derrube o servidor `app2` e repita as requisições para observar a tolerância a falhas:
```bash
docker stop app2
for i in {1..6}; do curl -s http://localhost:8080; done
```

2. **Balanceamento com Pesos (Weighted Round Robin):**
* Religue o segundo servidor: `docker start app2`
* Abra o arquivo `aula-02/nginx.conf` e altere o bloco `upstream` para dar peso maior ao `app1`:
```nginx
upstream cluster_web {
    server app1:80 weight=3;
    server app2:80 weight=1;
    server app3:80 weight=1;
}
```
* Reinicie o balanceador (`docker restart load-balancer`) e dispare novamente as 9 requisições com `curl` para verificar a nova proporção de tráfego.
