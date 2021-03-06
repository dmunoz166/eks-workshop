# WorkShop Elastic Kubernetes Services
**Amazon EKS** es un servicio de kubernetes completamente administrado por AWS 
![imagen](./img/EKS_HOL.png?raw=true "Lab eks")


## Ingresar al Portal AWS Workshop
Ingresar [aquí](https://dashboard.eventengine.run/)
- Ingresar el hash, lo redirijira al Team Dashboard.
- Acceder al AWS Conosele, en servicios buscar Cloud9 y seleccionarlo.

## Creando CD9 Workspace
Lanzar Cloud 9 en la región del laboratorio, en este caso "us-east-1 o North Virginia":
* Seleccionar **Create a environment**
* Nombrar el ambiente "eksworkshop", y selecionar "t3.small" en instance type
![Enviroment CD9](./img/eks_cd9.png?raw=true "CD9 Environment")
*Dejar las demas opciones como estan 
* Puede cerrar la ventana actual o directamente abrir una nueva terminal:
![term CD9](./img/cd9_term.png?raw=true "nueva terminal")
* Una vez se abra la nueva terminal continuaremos instalando las herramientas necesarias para el laboratorio.

## Instalando las Herramientas de Kubernetes
AWS EKS requiere instalar kubectl y kubelet, docker ya se encuentra instalado.

### Instalar kubectl
```bash
sudo curl --silent --location -o /usr/local/bin/kubectl \
  https://amazon-eks.s3.us-west-2.amazonaws.com/1.17.7/2020-07-08/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

### Actualizar awscli
Siguiendo la [documentación](https://docs.aws.amazon.com/cli/latest/userguide/install-linux.html) brindada por AWS
```bash
sudo pip install --upgrade awscli && hash -r
```

### Instalndo complementos utiles para el laboratorio
Instalamos [jq (manejo de archivos JSON)](https://linuxhint.com/bash_jq_command/), [envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html) y bash-completion
```bash
sudo yum -y install jq gettext bash-completion moreutils
```

### Instalar yq para procesamiento YAMLs
```bash
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
```

### Verificar que los binarios estan en la ruta adecuada y son ejecutables
```bash
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```
Debe ver que `kubectl in path jq in path envsubst in path aws in path`
### Habilitar el autocompletar para kubectl en bash
```bash
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```
### Definir la version del ALB (Application Load Balancer)
```bash
echo 'export ALB_INGRESS_VERSION="v1.1.8"' >>  ~/.bash_profile
.  ~/.bash_profile
```

## Crear un rol de IAM para el workspace
* Seguir este enlace para [crear un rol con acceso de administrador](https://console.aws.amazon.com/iam/home#/roles$new?step=review&commonUseCase=EC2%2BEC2&selectedUseCase=EC2&policies=arn:aws:iam::aws:policy%2FAdministratorAccess)
Confirm that AdministratorAccess is checked, then click Next: Tags to assign tags.
Take the defaults, and click Next: Review to review.
Enter eksworkshop-admin for the Name, and click Create role. 
### Enlazar el rol al workspace
* Puede seguir este enlace para [encontrar la instancia de EC2](https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances:tag:Name=aws-cloud*;sort=desc:launchTime)
* Select the instance, then choose Actions / Instance Settings / Attach/Replace IAM Role c9instancerole
* Elija el nombre que le asigno a su rol y enlacelo a la instancia

## Actualizar las opciones para el workspace
* Return to your workspace and click the gear icon (in top right corner), or click to open a new tab and choose “Open Preferences”
* Select AWS SETTINGS
* Turn off AWS managed temporary credentials
* Close the Preferences tab 
![CD9-IAM](./img/cd9-credenciales.png?raw=true)

- Asegurar que no hayan creadenciales temporales, se eliminara cualquier archivo actual.
```bash
rm -vf ${HOME}/.aws/credentials
```

- Configurar nuestras credenciales
```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
```
#### Validar el rol de IAM 
Debemos asegurarnos de estas en la región adecuada
```bash
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
```
```bash
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

```bash
aws sts get-caller-identity --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```
- En caso de que no funcione el comando verificar que el nombre `eksworkshop-admin` sea el que se creo anteriormente.

[x] **NO PROCEDER HASTA QUE SE VALIDE EL ROL DE IAM OBTENIENDO `IAM role valid`**

## Clonar el repositorio de los servicios
```bash
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

## Crear una llave SSH
lanzar el siguiente comando y oprimir enter para los 3 pasos siguientes
```bash
ssh-keygen
```
Subir la llave a EC2
```bash
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub
```

If you got an error similar to An error occurred (InvalidKey.Format) when calling the ImportKeyPair operation: Key is not in valid OpenSSH public key format then you can try this command instead

```bash
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material fileb://~/.ssh/id_rsa.pub
```
## Crear AWS KMS Custom Managed Key (CMK) 
Se crea un KMS para el cluster de eks para usarlo al momento de generar y encriptar secretos en el cluster
```bash
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
```
Traer el nombre de recurso de CMK en el comando de creación del cluster
```bash
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
```
Añadimos el `MASTER_ARN` al bash_profile
```bash
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```
# Lanzado un cluster usando EKSCTL
Descargar el binario de [eksctl](https://eksctl.io/) 
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```
Verificar que eksctl fue correctamente instalado, la version de eksctl debe ser mayor a 0.24.0 para desplegar un cluster de EKS en la versión 1.17
```bash
eksctl version
```
Activar el autocompletar para eksctl
```bash
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

## Lanzando el cluster
Debemos crear el archivo de despliegue para crear el cluster de eks de nombre [eksworkshop.yaml](./eksworkshop.yaml)

```bash

```
Usamos el archivo previamente creado para generar el cluster
![CreatingEKS](./img/clusterready.png?raw=true)
```bash
eksctl create cluster -f eksworkshop.yaml
```
> la creaión del cluster toma alrededor de 15 minutos

## Testeando el cluster
Para confirmar los nodos:

```bash
kubectl get nodes -A -o wide
kubectl get pods -A -o wide
kubectl 
```
Si vemos nuestros 3 nodos sabremos que nos hemos autenticado correctamente
![TestingEKS](./img/getno.png?raw=true)

```bash
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```
![TestingEKS](./img/eksctlnodeg.png?raw=true)
* Se puede revisar en cualquier instante el rol usando el comando
```bash
aws sts get-caller-identity
```
## Beginner

```bash
export DASHBOARD_VERSION="v2.0.0"

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml
```
- lanzamos el dashboard en el puerto 8080 para poder acceder a el de manera rapida desde CD9
```bash
kubectl proxy --port=8080 --address=0.0.0.0 --disable-filter=true &

```
- Ponemos la dirección al final de la URl `/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/` 
```bash
aws eks get-token --cluster-name eksworkshop-eksctl | jq -r '.status.token'
```
### limpiamos el dashboard

```bash
# kill proxy
pkill -f 'kubectl proxy --port=8080'

# delete dashboard
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/${DASHBOARD_VERSION}/aio/deploy/recommended.yaml

unset DASHBOARD_VERSION
```
## Desplegando Microservicios
```bash
cd ecsdemo-nodejs/

kubectl apply -f kubernetes/

kubectl get deploy ecsdemo-nodejs

cd ~/environment/ecsdemo-crystal

kubectl apply -f kubernetes/d
```
Revisa si existe el rol o si no lo crea para poder desplegar el alb.
```bash
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"

```
- Revisar que se implemento de manera automatica un load balancer al definir de esta forma el servicio
```bash
kubectl get svc ecsdemo-frontend -o wide
```
- Revisar que el ELB esta despeglado

```bash
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')

curl -m3 -v $ELB
```
- Escalar los backend
```bash
kubectl get deployments
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
kubectl get deployments

#borrar los deployments
cd ~/environment/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```

## HELM
Helm is a package manager for Kubernetes that packages multiple Kubernetes resources into a single logical deployment unit called a Chart. Charts are easy to create, version, share, and publish.

Install HELM
```bash

```

```bash
```

```bash
helm completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
source <(helm completion bash)

```

```bash
```
## Helth Checks

```bash
```

```bash
```
- Revisar los logs de un contenedor que ya crasheo
```bash
kubectl logs liveness-app --previous
```

```bash
```

```bash
```

```bash
```
