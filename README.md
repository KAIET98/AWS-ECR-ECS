# AWS-ECR-ECS
Este repositorio busca replicar el funcionamiento de lo que sería un Docker pero en los servicios de AWS incluído el Application Load Balancer.


# ECS



Primero de todo lo que vamos a ahcer es levantar un Cluster de ECS. 

## Creación cluster

Nombre: 

```
DemoCluster2

```
En cuanto a los VPCs vamos a dejar los que estaban en Default. En cuanto a los Subnets también dejamos los que estaban existentes, será ahí donde se desplieguen nuestras aplicaciones. 


Infraestructura: 

Lo que nos dice por defecto es que usemos el Fargate, un pay as you go. Se puede también lanzar Instancias de EC2, creando un Auto Scaling Group y demás manualmente. 

Creamos un Auto scaling group seleccionando "Create new ASG", por un ASG AWS con el Free Tier no nos cobrará dinero. Por lo que una de dos, nos aseguramos con atnerioridad a crear esto que no tenemos más ASGs levantados en nuestra cuenta, o lo mismo, las borramos. 

1. Operating sistem: Selecccioamos una simple, Amazon LInux 2
2. EC2 Instance Tyep: t2.micro. (Es decir la más pequeña que tenemos disponible, y la que entra ene el free tier). 
3. Desired capacity: los dejamos en un mínimo de 0, y un máximo de 5 por ahora. Vamos a proceder a hacer ediciones sobre esto luego. 

En cuanto a Monitorinig y Tags los dejamos tal cual están por ahora. 

Por ahora hemos creado las dos launch types: 
1. Fargate launch type (by default)
2. EC2 launch type 

Además, para ver comofunciona el EC2, podemos entrar al clsuter que hemos creado, ir a "Insfrastructure" y en "Capacity providers" podemos ver que hay uno del tipo "ASG Provider", y que tiene un hiperlink, si clickamos ahí, nos mandara a la consola de EC2 donde están los Application Load Balancers. Donde en detalles, te da la ifnoración de la capacidad deseada, la mínima capacidad, como la máxima. Todo ello creado por el cluster de ECS. 

Este cluster, tendrá instancias de EC2 para lanzár TASKS en él.

Aparte, siguiendo en el apartado de Infraestructrue en nuestro cluster, podemos ver que tenemos FARGATEs y FARGATE-SPOTs donde podmeos lanzar TASKS en ambos a parte de en la instancia de EC2.

## Despligue DOCKER 

### Despliegue en EC2: 

Vamos a ver cómo funciona esto de la instancia en el EC2. Vamos  a ir a EC2>Auto Scaling Groups, y en "Desired Capacity" vamos a cambiar de 0 a 1. Automáticamente, se deberái de crear una instancia EC2. Y es así. Lo jodido es que podemos ver su existencia bien en el Auto Scaling Group como en ECS>CLUSTER. 
Mejor dicho, ECS>CLUSTER>Capacity providers, podemos ver que en el ASG, en la columna de "Current Size" aparecera un 1. Y justo en la ventana de abajo en 
"Container Instances" habrá un EC2 corriendo y listo. 

### Despliegue en Fargate: 

Hemos hecho el despliegue en EC2, pero hay que tenerlo en cuenta que también se puede hacer el despliegue en Fargate. 


### Siguiendo con EC2: 


Primero de todo tenemos que crear un TASK DEFINITION 

#### TASK DEFINITION: 



Vamos a ir a ECS a Task Definitions , loq ue viene siendo una indicación de Cómo Crear Un Task en ECS. 

##### STEP1: Configure taskd efinition and containers

Nombre: 

```
nginxdemos-hello
```

(Es una imagen de docker que luego vamos a utilizar, vamos que puede ser el nombre otro cualquiera en sí)

Lo mismo, podemos ver la imagen de Docker, en DockerHu si googleamos y ponemos 'nginxdemos-hello' en google. 

CONTAINER-1: 

Name: 
```
nginxdemos-hello
```
Y en la Image URL: vamos a poenr lo que viene siendo la ruta de donde se va a leer (https://hub.docker.com/r/nginxdemos/hello/), vamos a decir el "tag" con el que se hacia el PUSH en docker. 

```
nginxdemos/hello
```

Además, le vamos a indicar que crea un HTTP server que escuche desde el puerto 80 por lo que va a ser:
1. Container port: 80 
2. Protocol: TCP


##### STEP 2: CONFIGURE ENVIRONMENTS, STORAGE, MONITORING AND TAGS

1. Environment

Se puede elegir bien lanzarlo en Fargate mode es decir, serverless, pay as you go, easy, o en modo EC2, nosotros por mantener las cosas simles vamos a lanzar en 
"AWS FARGATE (Serverless)". 

Operating sistem: linux

CPU: .5 CPU
Memory: 1 GB


Task roles: 

Si el Task estuviera haciendo API calls por ejemplo a S3 o así, habria que indicr un IAM TASK ROLE. 

Los demás los dejamos en Default

##### STEP 3: Review and create

Le damos a create. 


## Comenzar el Task Definition


Para poder lanzar el Task Definition, tenemos que hacerlo sobre un Application Load Balancer por lo que, vamos a tener que levantar también 2 Grupos de seguridad: 

### Grupos de Seguridad:

1. Primer grupo de seguridad: 

Va a ser mi Grupo de seguridad dirigido para poder comunicarse el ALB con mi ECS.
Es decir, vamos a permitir que el usuario se conecte al ECS por medio del ALB. El ALB actúa comoun puente entre el cliente y el ECS. Entonces, el cliente se ocnectara desde cualquier sitio por lo que: 

Nombre: 

```
alb-ecs-sg
```


INBOUND RULES: 

-Type: CUSTOM TCP ; Port Range: 80, Source Type: Anywhere - IP V4. 
-Type: CUSTOM TCP: port range: 80, Source Type: Anywhere - IP V6


Cramos el grupo de seguridad. 


2. Segundo grupo de seguridad. 

Entonces una vez que el cliente acceda al ECS, ahora tiene que acceder a un ECS-TASK específico. Por loq ue tenemos que indicar un grupo d eseguridad indicado para este Task. 

Name: 

```
nginx-demo-sg
```

Descripción 

```
Grupo de seguridada par ael NGIX
```

INBOUND RULES: 

Type: ALL TCP
Ahora, vamos a permitir que se conecte por "cualquier sitio", porl o que PORT RANGE: 0, pero ese cualquier sitio nos importa, especialmente que Source Type
sea el gurpo de seguridad externo que acabmos de crear, por lo que prestemos mucha atención en la lista que nos aparezca para que pillemos aquél que tenga los carácteres de: 

```
alb-ecs-sg
```

En la descrición vamos a poner un mensaje aclarativo que nos puede venir muy bien para entenderlo como: 

```
Permitiendo el trá desde el ALB
```

### Task Service: 

Entramos a ECS a nuestro Cluster y dentro de "Services" le damos a "Deploy". 

En la configuración de le daremos a "Service". 

Family, el que acabamos de crear: 

```
nginxdemos-hello
```
Revision: 1

Service name: 

```
nginxdemos
```
Y los tasks serán de: 1

#### ALB: 

En cuanto a Application Load Balancer, Sí que queremos que tenga, por lo que le damos a Create a new load Balancer. 

Name

```
DemoALBForECS
```

Port: 80 
Protocolo: HTTP

Target group name: 

```
nginx-ecs
```

Protocol: HHTP
Health check path: /
Health Check grace period: 20 (seg.)

#### Networking: 


No en VPC, pero sino en el Security Group, escogeremos uno que ya exite, y seleccioaremos el que teng ala etiqueta externa que  hemos creado antes: USER-ALB: 

```
alb-ecs-sg
```
y solo dejamos ese Security-group.

A esta altura, nuestro servicio se desplegara y nuestro ALB se va activar. 

## Vamos a ver qué ha construido


Vamos a EC2, sen Load Blanacers, entramos al que hemos creado, y por casualidad si nso ah pueso el security group relativo a "nginx.demo-sg" lo cambianmos a "alb-ecs-sg" para poder acceder al ALB. 

Pillamos el DNS y lo ponemos en Chrome.

Nos deberia de aparecer el serivdor de NGINX :))))) . 

Para terminar; vamos a nuestro lcuster, a servicios, a lo que hemos creado, y en edit, cambiamos el Desired Tasks = 0. 
Si tenemos más de un ALB con el que hemos creado ahora, borrarlo.

## (PASO AVANZADO): SCALADO D SERVICIOS. 

hata ahora hemos tenido 1 ECS Task corriendo en el ALB, vamos a lanzar más. 

Muchos ojo con este paso, porque si lo haces y lodejas corriendo te van a cobrar pasta. .

Nos vamos a ECS> CLUSTER, sevicios, clicammos en lo que tenemos desplegado ene l hiperlink y nos pone que el Desired Task = 1, bien , podemos editarlo y en Desired tasks poner rollo 4. 

Entonces, si loa ctualizamos y le damos un poco de tiempo, va a lanzar 4 TASKS en Fargate, por lo que; en 0, nos va a provisinonar los 4 tasks y nos los pondra en marhca. 

Esto, si le damos un poco de tiempo rollo 1 minuto y medio; si vamos a la ventna que hemos abiarto en el servidor, en chrome, si actualizamos una y otra vez, los Server adresses se cambiaran entre 4, esd ecir porque tenemos 4 contenedores de Docker corriendo en Fargate una pasada.


Para terminar; vamos a nuestro lcuster, a servicios, a lo que hemos creado, y en edit, cambiamos el Desired Tasks = 0. 

Si tenemos más de un ALB con el que hemos creado ahora, borrarlo.
:)
