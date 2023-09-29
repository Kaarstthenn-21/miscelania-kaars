# Ejercicio de Well-Architected Discovery Automatizado usando la herramienta Open Source: Prowler

**Nota: Esta página/gist estará disponible solo por UN TIEMPO LIMITADO. Por favor, descargue el contenido de esta página si lo encuentra útil.**

# Introducción
Este ejercicio proporciona una rápida introducción sobre cómo analizar automáticamente el estado de seguridad de una cuenta de AWS. Se utilizará la herramienta de auditoría de seguridad de código abierto [Prowler](https://duckduckgo.com) en este ejercicio. (¡Un GRAN agradecimiento a Tony y a los colaboradores de Prowler por construir una herramienta tan maravillosa...)

![Prowler](https://user-images.githubusercontent.com/3985464/113734260-7ba06900-96fb-11eb-82bc-d4f68a1e2710.png)

# Advertencia/Descargo de responsabilidad
- Participar en este ejercicio es **BAJO SU PROPIO RIESGO**. Existe la posibilidad de que esto active sistemas internos de detección de intrusiones o anomalías. Ejecute esto solo si se siente cómodo con lo que hará (es decir, configurar un Rol, ejecutar código desde un repositorio público de código abierto).
- Úselo solo en una cuenta de prueba o desarrollo. No ejecute esto en una cuenta de producción a menos que lo haya probado completamente.

## Resumen del ejercicio:

Preparación
- Abrir la Consola de AWS con privilegios de administrador o PowerUser.
- Iniciar un "Cloud Shell".
- Crear un nuevo Rol de IAM llamado "ProwlerExecRole" con permisos restringidos utilizando comandos de AWS CLI.

Demo
- Cambiar al nuevo Rol de IAM "ProwlerExecRole" en la Consola.
- Crear un nuevo [Environment Cloud9](https://docs.aws.amazon.com/es_es/cloud9/latest/user-guide/tutorial-create-environment.html) tipo **Ubuntu**.
- Instale la herramienta Prowler.
- Ejecutar la herramienta de descubrimiento automatizado de Prowler y luego descargar el informe resultante.

Limpieza
- Vuelva a su usuario/rol IAM normal.
- Limpie eliminando el rol que se creó.

## Detalle del ejercicio:

### Preparación
- Abra la Consola de AWS con privilegios de administrador o PowerUser.
  - Seleccione una región en la que tenga cargas de trabajo de prueba o desarrollo en ejecución para obtener los mejores resultados.

- Inicie un "Cloud Shell".
- Configure un nuevo Rol de IAM con permisos restringidos ejecutando los siguientes comandos.
  - Nota: Estos comandos configurarán un nuevo rol que permitirá que su usuario ingrese a un entorno con permisos restringidos en el que puede ejecutar el código de Prowler sin temor a romper o exponer nada.
```
CURRENTAWSUSER=$(aws sts get-caller-identity --query Arn --output text); \
curl --silent -O https://gist.githubusercontent.com/lechu77/58c8d9b455c750d45c69e07fc105b6b5/raw/8abf70f29513b932957f2e154f726d27757125b3/prowler-exec-role.yaml; \
aws cloudformation create-stack \
  --capabilities CAPABILITY_IAM --capabilities CAPABILITY_NAMED_IAM \
  --template-body "file://./prowler-exec-role.yaml" \
  --stack-name "ProwlerExecRole" \
  --parameters "ParameterKey=AuthorisedARN,ParameterValue=${CURRENTAWSUSER}"; \
echo Esperando hasta que se cree el rol de IAM...; \
aws cloudformation wait stack-create-complete --stack-name ProwlerExecRole; \
echo ¡Creación del Rol de IAM finalizada! ;\
echo Usuario actual = ${CURRENTAWSUSER}
```

### Demo
- Cambie al nuevo Rol de IAM "ProwlerExecRole" en la Consola. Esto se puede hacer de dos maneras: directamente en Cloud Shell o cambiando toda la Consola web.
  - Use el menú en la parte superior derecha de la Consola para copiar su ID de cuenta.
  - Luego, use nuevamente el menú en la parte superior derecha y seleccione "Cambiar Rol".
  - Se le pedirá que complete tres campos. Pegue el ID de cuenta en el campo superior.
  - Pegue el nombre del Rol "ProwlerExecRole" en el segundo campo. (El tercer campo no es necesario.)

**NOTA:** Asegúrese de crear un ambiente tipo **Ubuntu** en [Cloud9](https://aws.amazon.com/es/cloud9/) tal y como se detalla a continuación:

- Ingrese a [Cloud9](https://console.aws.amazon.com/cloud9control/home) para luego [crear un nuevo Environment](https://docs.aws.amazon.com/es_es/cloud9/latest/user-guide/tutorial-create-environment.html) tipo **Ubuntu**
	- Elija el botón Create environment (Crear entorno) grande en una de las ubicaciones mostradas. Si aún no tiene entornos de AWS Cloud9, el botón se muestra en una página de bienvenida
	![Create_environment01](https://docs.aws.amazon.com/es_es/cloud9/latest/user-guide/images/create_welcome_env_new_UX.png)
	
	- Si ya tiene entornos de AWS Cloud9, el botón se muestra como se indica a continuación
	![Create_environment02](https://docs.aws.amazon.com/es_es/cloud9/latest/user-guide/images/console_create_env_new_UX.png)

	- Indique un Nombre y luego asegúrese de seleccionar **Ubuntu** en *Platform* , luego en *Network settings* seleccione **Secure Shell (SSH)** (El VPC debe ser el default) y deje todas las otras opciones por default. 
	![Cloud9_Create](https://d2908q01vomqb2.cloudfront.net/fc074d501302eb2b93e2554793fcaf50b3bf7291/2023/01/19/Figure-3.-Create-an-AWS-Cloud9-environment.png)
	- Elija Create (Crear) para crear su entorno y, a continuación, se le redirigirá a la página de inicio. Si la cuenta se ha creado correctamente, aparecerá una barra parpadeante verde en la parte superior de la consola de AWS Cloud9. Puede seleccionar el nuevo entorno y elegir Open (Abrir) para lanzar el IDE.
	![Cloud9_Open](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2023/07/14/image3.png)
	- Si todo se ha ejecutado correctamente, deberá encontrarse dentro del IDE
	![Cloud9](https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/09/22/AWS-Cloud9-Network-Figure-4.jpg)


- Configure la herramienta Prowler.
  - Instalar Prowler y sus dependencias de software.
```
CURRENTAWSREGION=$(aws configure get region)
sudo dd if=/dev/zero of=/swapfile bs=1024k count=512
sudo mkswap /swapfile
sudo chmod 0644 /swapfile
sudo swapon /swapfile
pip install prowler
prowler -v  # Verifique el número de versión d Prowler
```
### Ejecutar inventario rápido con Prowler
```
CURRENTAWSREGION=$(aws configure get region)
prowler aws -f ${CURRENTAWSREGION} --quick-inventory
```
- Si todo funciona correctamente, debería comenzar a ver un escaneo de seguridad en curso, como este:
![Prowler_quick_inventory](https://raw.githubusercontent.com/prowler-cloud/prowler/master/docs/img/quick-inventory.jpg)

### Ejemplos de ejecución de Prowler
- Ejecutar Prowler sólo para los recursos de la región actual:
```
CURRENTAWSREGION=$(aws configure get region)
echo $CURRENTAWSREGION 
prowler aws -f ${CURRENTAWSREGION}
```
- Liste las comprobaciones disponibles que Prowler puede ejecutar:
```
prowler aws --list-checks -f ${CURRENTAWSREGION}
```
- Liste los servicios disponibles que Prowler puede ejecutar:
```	
prowler aws --list-services -f ${CURRENTAWSREGION}
```
- Liste los Marcos de Cumplimiento (Compliance Frameworks) disponibles que Prowler puede ejecutar:
```	
prowler aws --list-compliance -f ${CURRENTAWSREGION}
```

- Ejecutar Prowler con comprobaciones centradas en recursos expuestos a Internet:
```
# rm ~/prowler/output/* ~/prowler-*.zip -f
prowler aws --categories internet-exposed \
  -f ${CURRENTAWSREGION} -M json json-asff csv html
ls -lh output/prowler-*
```
- Ejecutar Prowler sobre los recursos con TAGs específicos:
```
prowler aws --resource-tags Environment=Prod Project=MyNewAPP
```
- Ejecutar Prowler sobre los recursos con ARNs específicos:
```
prowler aws --resource-arn arn:aws:iam::012345678910:user/test arn:aws:ec2:us-east-1:123456789012:vpc/vpc-12345678
```

- Aquí hay un ejemplo de cómo se verá el informe HTML:
![Prowler](https://user-images.githubusercontent.com/3985464/141443976-41d32cc2-533d-405a-92cb-affc3995d6ec.png)

### Limpieza
- Elimine el ambiente de [Cloud9](https://console.aws.amazon.com/cloud9control/home) creado.
- Cambie de nuevo a su usuario/rol IAM normal.
  - Utilice "Switch back" en el menú de la esquina superior derecha de la consola.
- Limpie eliminando el rol que se creó.
```
aws cloudformation delete-stack --stack-name ProwlerExecRole; \
echo Esperando hasta que se elimine la pila de CloudFormation...; \
aws cloudformation wait stack-delete-complete --stack-name ProwlerExecRole; \
rm ./prowler-exec-role.yaml; \
echo ¡La pila de CloudFormation ha sido eliminada!
``` 