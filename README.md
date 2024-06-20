# aws-cluster-simple

## Criar EC2

- Criar conta na AWS
- Criar grupo de segurança (firewall)
- Criar instância manager EC2 (cluster-manager-1, recomendo o uso de somente um manager devido a ferramentas que iremos utilizar)
- Criar um IP elástico e associar ele a instância criada anteriormente
- Criar instâncias workers (cluster-worker-1, cluster-worker-2)

## Instalar Docker, Docker Compose e Git

Execute os seguintes comandos nas instancias criadas

```
# atualizar dependencias
sudo yum update

# instalar o git
sudo yum install -y git

# instalar o docker
sudo yum install docker -y

# iniciar o docker
sudo service docker start

# colocar a inicialização automática
sudo chkconfig docker on

# recomendado reiniciar a maquina
sudo reboot
```

Após reiniciar execute o seguinte comando

```
# baixar docker-compose (latest)
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

# Verificar se funcionou o docker-compose
docker-compose version

# Verificar se funcionou o docker
sudo systemctl status docker.service
```

####

Execute o seguinte comando para não ter que rodar comandos do docker com usuário super

```
sudo usermod -a -G docker ec2-user
id ec2-user
# Reload a Linux user's group assignments to docker w/o logout
newgrp docker
```

## Instalar Docker Swarm

Na máquina `cluster-manager-1` execute o seguinte comando

```
docker swarm init
```

A saída do comando deve ser parecida com:

```
Swarm initialized: current node (zfly9by3ni4tmgywzfd8hjs38) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1431rhqt6h2hg61ugcp1q1e82vbqs5n9pe9y6fwmj7yeez0mev-8wvohdt0br7m9jqrejx2x4n3l 172.31.37.174:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Nas máquinas `cluster-worker-1` e `cluster-worker-2` execute o seguinte comando

```
docker swarm join --token SWMTKN-1-1431rhqt6h2hg61ugcp1q1e82vbqs5n9pe9y6fwmj7yeez0mev-8wvohdt0br7m9jqrejx2x4n3l 172.31.37.174:2377
```

Para confirmar que o Swarm foi configurado corretamente execute o seguinte comando na maquina `cluster-manager-1`

```
docker node ls
```

A saída do comando deve ser parecida com:

```
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
zfly9by3ni4tmgywzfd8hjs38 *   manager    Ready     Active         Leader           20.10.25
wfdqfc6gbc5fwipz9gymwarys     worker_1   Ready     Active                          20.10.25
igikbrytkndykcb7xcapd8mdo     worker_2   Ready     Active                          20.10.25
```

Certifique que todos os nós (managers e workers) estejam listados

## Instalando Portainer e Traefik

Todos os comandos serão realizados na maquina `cluster-manager-1`, execute o seguinte comando:

```
# cria uma rede para comunicação dos serviços
docker network create -d overlay traefik_proxy
```

Nesse repositório temos duas pastas, `cluster` e `stacks`, a pasta cluster é destinada a todas as stacks que serão executadas dentro do próprio Docker Swarm e as stacks são arquivos docker-compose.yml que podem ser executadas dentro do Portainer posteriormente.

####

Acesse a pasta cluster/traefik e execute o seguinte comando

```
docker stack deploy traefik -c docker-compose.yml
```

Acesse a pasta cluster/portainer e execute o seguinte comando

```
docker stack deploy portainer -c docker-compose.yml
```

Execute o seguinte comando para verificar se todos os services estão funcionando

```
docker service ls
```

A saída deve ser parecida com:

```
ID             NAME                  MODE         REPLICAS   IMAGE                           PORTS
hg4cfiz1ew23   portainer_agent       global       1/1        portainer/agent:2.20.3
xuv9swxaybyp   portainer_portainer   replicated   1/1        portainer/portainer-ce:2.20.3
495myk7anzuv   traefik_traefik       replicated   1/1        traefik:v3.0                    *:80->80/tcp, *:443->443/tcp
```

####

Os dominios e subdominios utilizados devem ser apontados para o Traefik (o IP elástico criado no inicio), definido como tipo A, os certificados são gerados e renovados automáticamente pelo Let's Encrypt

####

Você já pode subir aplicações normalmente utilizando a interface do portainer

## Monitoramento do Cluster e Docker Swarm
