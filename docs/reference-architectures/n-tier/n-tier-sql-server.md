---
title: Aplicación de n niveles con SQL Server
description: Implementación de una arquitectura de varios niveles en Azure para lograr una mayor disponibilidad, seguridad, escalabilidad y facilidad de uso.
author: MikeWasson
ms.date: 05/03/2018
pnp.series.title: Windows VM workloads
pnp.series.next: multi-region-application
pnp.series.prev: multi-vm
ms.openlocfilehash: 0f170f2fbcbbfeace53db199cb5d3949415b5546
ms.sourcegitcommit: a5e549c15a948f6fb5cec786dbddc8578af3be66
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 05/06/2018
ms.locfileid: "33673597"
---
# <a name="n-tier-application-with-sql-server"></a>Aplicación de n niveles con SQL Server

Esta arquitectura de referencia muestra cómo implementar máquinas virtuales y una red virtual configurada para una aplicación de n niveles, con SQL Server en Windows para la capa de datos. [**Implemente esta solución**.](#deploy-the-solution) 

![[0]][0]

*Descargue un [archivo Visio][visio-download] de esta arquitectura.*

## <a name="architecture"></a>Architecture 

La arquitectura consta de los siguientes componentes:

* **Grupo de recursos.** Los [grupos de recursos][resource-manager-overview] se utilizan para agrupar los recursos, para que puedan administrarse según su duración, su propietario u otros criterios.

* **Red virtual y subredes.** Cada máquina virtual de Azure se implementa en una red virtual que se puede dividir en varias subredes. Cree una subred independiente para cada nivel. 

* **Grupos de seguridad de red.** Use [grupos de seguridad de red][nsg] (NSG) para restringir el tráfico de red dentro de la red virtual. Por ejemplo, en la arquitectura de tres niveles que se muestra aquí, el nivel de base de datos no acepta el tráfico desde el front-end web, solo desde el nivel Business y la subred de administración.

* **Máquinas virtuales**. Para obtener recomendaciones sobre la configuración de máquinas virtuales, consulte [Ejecución de una VM con Windows en Azure](./windows-vm.md) y [Ejecución de una VM con Linux en Azure](./linux-vm.md).

* **Conjuntos de disponibilidad**. Cree un [conjunto de disponibilidad][azure-availability-sets] para cada nivel y aprovisione al menos dos máquinas virtuales en cada nivel. Esto hace que las máquinas virtuales sean aptas para un [Acuerdo de Nivel de Servicio (SLA)][vm-sla] mayor. 

* **Conjunto de escalado de máquinas virtuales** (no se muestra). Un [conjunto de escalado de máquinas virtuales][vmss] es una alternativa al uso de un conjunto de disponibilidad. Los conjuntos de escalado facilitan el escalado horizontal de las máquinas virtuales de un nivel, de forma manual o automática, y según reglas predefinidas.

* **Equilibradores de carga de Azure.** Los [equilibradores de carga][load-balancer] distribuyen las solicitudes entrantes de Internet a las instancias de máquina virtual. Use un [equilibrador de carga público][load-balancer-external] para distribuir el tráfico entrante de Internet al nivel Web y un [equilibrador de carga interno][load-balancer-internal] para distribuir el tráfico de red del nivel Web al nivel Business.

* **Dirección IP pública**. Se necesita una dirección IP pública para que el equilibrador de carga reciba tráfico de Internet.

* **JumpBox.** También se denomina [host bastión]. Se trata de una máquina virtual segura en la red que usan los administradores para conectarse al resto de máquinas virtuales. El Jumpbox tiene un NSG que solo permite el tráfico remoto que procede de direcciones IP públicas de una lista segura. El NSG debe permitir el tráfico de escritorio remoto (RDP).

* **Grupo de disponibilidad AlwaysOn de SQL Server**. Proporciona alta disponibilidad en el nivel de datos, al habilitar la replicación y la conmutación por error.

* **Servidores de Active Directory Domain Services (AD DS)**. Los grupos de disponibilidad Always On de SQL Server están unidos a un dominio para poder habilitar la tecnología de clúster de conmutación por error de Windows Server (WSFC) para la conmutación por error. 

* **Azure DNS**. [Azure DNS][azure-dns] es un servicio de hospedaje para dominios DNS que permite resolver nombres mediante la infraestructura de Microsoft Azure. Al hospedar dominios en Azure, puede administrar los registros DNS con las mismas credenciales, API, herramientas y facturación que con los demás servicios de Azure.

## <a name="recommendations"></a>Recomendaciones

Los requisitos pueden diferir de los de la arquitectura que se describe aquí. Use estas recomendaciones como punto inicial. 

### <a name="vnet--subnets"></a>Red virtual/subredes

Cuando cree la red virtual, determine cuántas direcciones IP requieren los recursos de cada subred. Especifique una máscara de subred y un intervalo de direcciones de la red virtual lo suficientemente grande para la dirección IP requerida con el uso de la notación [CIDR]. Use un espacio de direcciones que se encuentre dentro de los [bloques de direcciones IP privados][private-ip-space] estándar, que son 10.0.0.0/8, 172.16.0.0/12 y 192.168.0.0/16.

Elija un intervalo de direcciones que no se superponga con la red local, en caso de que necesite configurar una puerta de enlace entre la red virtual y la red local más adelante. Una vez creada la red virtual, no se puede cambiar el intervalo de direcciones.

Diseñe subredes teniendo en cuenta los requisitos de funcionalidad y seguridad. Todas las máquinas virtuales dentro del mismo nivel o rol deben incluirse en la misma subred, lo que puede servir como un límite de seguridad. Para obtener más información sobre el diseño de redes virtuales y subredes, vea [Planeación y diseño de redes virtuales de Azure][plan-network].

### <a name="load-balancers"></a>Equilibradores de carga

No exponga las máquinas virtuales directamente a Internet; en su lugar, asigne una dirección IP privada a cada máquina virtual. Los clientes se conectan con la dirección IP del equilibrador de carga público.

Defina reglas del equilibrador de carga para dirigir el tráfico de red a las máquinas virtuales. Por ejemplo, para habilitar el tráfico HTTP, cree una regla que asigne el puerto 80 de la configuración de front-end al puerto 80 del grupo de direcciones de back-end. Cuando un cliente envía una solicitud HTTP al puerto 80, el equilibrador de carga selecciona una dirección IP de back-end mediante un [algoritmo hash][load-balancer-hashing] que incluye la dirección IP de origen. De ese modo, las solicitudes de cliente se distribuyen entre todas las máquinas virtuales.

### <a name="network-security-groups"></a>Grupos de seguridad de red

Use reglas NSG para restringir el tráfico entre los niveles. Por ejemplo, en la arquitectura de tres niveles mostrada anteriormente, el nivel Web no se comunica directamente con el nivel de base de datos. Para exigir esto, el nivel de base de datos debe bloquear el tráfico entrante desde la subred del nivel Web.  

1. Deniegue todo el tráfico entrante de la red virtual. (Use la etiqueta `VIRTUAL_NETWORK` de la regla). 
2. Permita el tráfico entrante de la subred del nivel Business.  
3. Permita el tráfico entrante de la propia subred del nivel de la base de datos. Esta regla permite la comunicación entre las máquinas virtuales de la base de datos, lo cual es necesario para la replicación y la conmutación por error de esta.
4. Permita el tráfico RDP (puerto 3389) desde la subred de JumpBox. Esta regla permite a los administradores conectarse al nivel de base de datos desde JumpBox.

Cree las reglas 2 &ndash; 4 con una prioridad más alta que la primera regla para que puedan invalidarla.


### <a name="sql-server-always-on-availability-groups"></a>Grupos de disponibilidad AlwaysOn de SQL Server

Se recomiendan los [grupos de disponibilidad AlwaysOn][sql-alwayson] para alta disponibilidad de SQL Server. Antes de Windows Server 2016, los grupos de disponibilidad AlwaysOn requerían un controlador de dominio y todos los nodos del grupo de disponibilidad debían estar en el mismo dominio de AD.

Otros niveles se conectan a la base de datos a través de una [escucha de grupo de disponibilidad][sql-alwayson-listeners]. La escucha permite a un cliente SQL conectarse sin conocer el nombre de la instancia física de SQL Server. Las máquinas virtuales que acceden a la base de datos deben estar unidas al dominio. El cliente (en este caso, otro nivel) utiliza DNS para resolver el nombre de la red virtual de la escucha en direcciones IP.

Configure el grupo de disponibilidad AlwaysOn de SQL Server como sigue:

1. Cree un clúster de clústeres de conmutación por error de Windows Server (WSFC), un grupo de disponibilidad AlwaysOn de SQL Server y una réplica principal. Para más información, vea [Introducción a Grupos de disponibilidad AlwaysOn][sql-alwayson-getting-started]. 
2. Cree un equilibrador de carga interno con una dirección IP privada estática.
3. Cree una escucha de grupo de disponibilidad y asigne el nombre DNS de la escucha a la dirección IP del equilibrador de carga interno. 
4. Cree una regla del equilibrador de carga para el puerto de escucha de SQL Server (puerto TCP 1433 de forma predeterminada). La regla del equilibrador de carga debe habilitar la *IP flotante*, también denominada Direct Server Return. Esto causa que la máquina virtual responda directamente al cliente, lo que permite establecer una conexión directa con la réplica principal.
  
   > [!NOTE]
   > Cuando la IP flotante está habilitada, el número de puerto de front-end debe ser el mismo que el número de puerto de back-end en la regla del equilibrador de carga.
   > 
   > 

Cuando un cliente SQL intenta conectarse, el equilibrador de carga enruta la solicitud de conexión a la réplica principal. Si se produce una conmutación por error a otra réplica, el equilibrador de carga enruta automáticamente las solicitudes posteriores a una nueva réplica principal. Para más información, vea [Configuración de un equilibrador de carga para Grupos de disponibilidad AlwaysOn de SQL Server][sql-alwayson-ilb].

Durante una conmutación por error, se cierran las conexiones de cliente existentes. Una vez completada la conmutación por error, las conexiones nuevas se enrutarán a la nueva réplica principal.

Si la aplicación realiza significativamente más lecturas que escrituras, puede descargar algunas de las consultas de solo lectura en una réplica secundaria. Vea [Usar un agente de escucha para conectarse a una réplica secundaria de solo lectura (enrutamiento de solo lectura)][sql-alwayson-read-only-routing].

Pruebe la implementación mediante el [forzado de una conmutación por error manual][sql-alwayson-force-failover] del grupo de disponibilidad.

### <a name="jumpbox"></a>JumpBox

No permita el acceso RDP desde la red pública de Internet a las máquinas virtuales que ejecutan la carga de trabajo de la aplicación. En su lugar, todo el acceso RDP a estas máquinas virtuales debe realizarse a través de JumpBox. Un administrador inicia sesión en JumpBox y, después, en la otra máquina virtual desde JumpBox. JumpBox permite el tráfico RDP desde Internet, pero solo desde direcciones IP conocidas y seguras.

JumpBox tiene unos requisitos de rendimiento mínimos, por lo que puede seleccionar un pequeño tamaño de máquina virtual. Cree una [dirección IP pública] para JumpBox. Coloque JumpBox en la misma red virtual que las demás máquinas virtuales, pero en una subred de administración independiente.

Para proteger JumpBox, agregue una regla de grupo de seguridad de red que permita las conexiones RDP solo desde un conjunto seguro de direcciones IP públicas. Configure el NSG para las demás subredes, a fin de permitir el tráfico RDP de la subred de administración.

## <a name="scalability-considerations"></a>Consideraciones sobre escalabilidad

Los [conjuntos de escalado de máquinas virtuales][vmss] permiten implementar y administrar un conjunto de máquinas virtuales idénticas. Los conjuntos de escalado admiten el escalado automático en función de las métricas de rendimiento. A medida que aumenta la carga en las máquinas virtuales, se agregan más máquinas virtuales automáticamente al equilibrador de carga. Considere la posibilidad de usar conjuntos de escalado si necesita escalar horizontalmente las máquinas virtuales de inmediato o si necesita realizar el escalado automático.

Hay dos maneras básicas de configurar máquinas virtuales implementadas en un conjunto de escalado:

- Use extensiones para configurar la máquina virtual después de aprovisionarla. Con este método, las nuevas instancias de máquina virtual pueden tardar más en iniciarse que una máquina virtual sin extensiones.

- Implemente un [disco administrado](/azure/storage/storage-managed-disks-overview) con una imagen de disco personalizada. Esta opción puede ser más rápida de implementar. Sin embargo, requiere mantener la imagen actualizada.

Para obtener consideraciones adicionales, consulte [Consideraciones de diseño para conjuntos de escalado][vmss-design].

> [!TIP]
> Cuando utilice cualquier solución de escalado automático, pruébela con antelación con cargas de trabajo de nivel de producción.

Cada suscripción de Azure tiene límites predeterminados establecidos, incluido un número máximo de máquinas virtuales por región. Puede aumentar el límite si rellena una solicitud de soporte técnico. Para más información, consulte [Límites, cuotas y restricciones de suscripción y servicios de Microsoft Azure][subscription-limits].

## <a name="availability-considerations"></a>Consideraciones sobre disponibilidad

Si no va a usar conjuntos de escalado de máquinas virtuales, ponga estas en el mismo nivel en un conjunto de disponibilidad. Cree al menos dos máquinas virtuales en el conjunto de disponibilidad, para admitir el [SLA de disponibilidad para máquinas virtuales de Azure][vm-sla]. Para más información, consulte [Administración de la disponibilidad de las máquinas virtuales][availability-set]. 

El equilibrador de carga usa [sondeos de mantenimiento][health-probes] para supervisar la disponibilidad de las instancias de máquina virtual. Si un sondeo no puede conectar con una instancia durante un período de tiempo de espera, el equilibrador de carga deja de enviar tráfico a esa máquina virtual. Sin embargo, el equilibrador de carga continuará con el sondeo y, si la máquina virtual vuelve a estar disponible, el equilibrador de carga reanudará el envío del tráfico a esa máquina virtual.

A continuación, se presentan algunas recomendaciones sobre los sondeos de mantenimiento del equilibrador de carga:

* Los sondeos pueden probar los protocolos HTTP o TCP. Si las máquinas virtuales ejecutan un servidor HTTP, cree un sondeo HTTP. De lo contrario, cree un sondeo TCP.
* Para un sondeo HTTP, especifique la ruta de acceso a un punto de conexión HTTP. El sondeo comprueba si hay una respuesta HTTP 200 desde esta ruta de acceso. Puede ser la ruta de acceso raíz ("/") o un punto de conexión de supervisión de mantenimiento que implementa alguna lógica personalizada para comprobar el mantenimiento de la aplicación. El punto de conexión debe permitir solicitudes HTTP anónimas.
* El sondeo se envía desde una [dirección IP conocida][health-probe-ip], 168.63.129.16. Asegúrese de que no bloquear el tráfico hacia o desde esta dirección IP en las directivas de firewall o en las reglas de grupo de seguridad de red (NSG).
* Use [registros de sondeo de mantenimiento][health-probe-log] para ver el estado de los sondeos de mantenimiento. Habilite el registro en Azure Portal para cada equilibrador de carga. Los registros se escriben en Azure Blob Storage. Los registros muestran cuántas máquinas virtuales del back-end no reciben tráfico de red debido a las respuestas de sondeo con error.

Si necesita más disponibilidad de la que proporciona el [Acuerdo de Nivel de Servicio de Azure para máquinas virtuales][vm-sla], considere la replicación de la aplicación entre dos regiones y use Azure Traffic Manager para la conmutación por error. Para más información, consulte [Aplicación de n niveles para varias regiones para obtener alta disponibilidad][multi-dc].  

## <a name="security-considerations"></a>Consideraciones sobre la seguridad

Las redes virtuales son un límite de aislamiento del tráfico de Azure. Las máquinas virtuales de una red virtual no se pueden comunicar directamente con las máquinas virtuales de una red virtual diferente. Las máquinas virtuales que se encuentran en la misma red virtual se pueden comunicar entre sí, a menos que se creen [grupos de seguridad de red][nsg] (NSG) para restringir el tráfico. Para más información, consulte [Servicios en la nube de Microsoft y seguridad de red][network-security].

Para el tráfico entrante de Internet, las reglas del equilibrador de carga definen qué tráfico puede alcanzar el back-end. Sin embargo, las reglas del equilibrador de carga no son compatibles con las listas seguras de IP, por lo que si desea agregar determinadas direcciones IP públicas a una lista segura, agregue un NSG a la subred.

Considere la posibilidad de agregar una aplicación virtual de red (NVA) para crear una red perimetral entre la red de Internet y la red virtual de Azure. NVA es un término genérico para una aplicación virtual que puede realizar tareas relacionadas con la red, como firewall, inspección de paquetes, auditoría y enrutamiento personalizado. Para más información, vea [Implementación de una red perimetral entre Internet y Azure][dmz].

Cifre información confidencial en reposo y use [Azure Key Vault][azure-key-vault] para administrar las claves de cifrado de la base de datos. Key Vault puede almacenar las claves de cifrado en módulos de seguridad de hardware (HSM). Para más información, consulte [Configuración de la integración de Azure Key Vault para SQL Server en máquinas virtuales de Azure][sql-keyvault]. También se recomienda almacenar los secretos de aplicación como, por ejemplo, las cadenas de conexión de base de datos, en Key Vault.

## <a name="deploy-the-solution"></a>Implementación de la solución

Hay disponible una implementación de esta arquitectura de referencia en [GitHub][github-folder]. 

### <a name="prerequisites"></a>requisitos previos

1. Clone, bifurque o descargue el archivo ZIP del repositorio de GitHub de [arquitecturas de referencia][ref-arch-repo].

2. Asegúrese de que tiene la CLI de Azure 2.0 instalada en el equipo. Para instalar la CLI, siga las instrucciones de [Instalación de la CLI de Azure 2.0][azure-cli-2].

3. Instale el paquete de NPM de [Azure Building Blocks][azbb].

   ```bash
   npm install -g @mspnp/azure-building-blocks
   ```

4. Desde un símbolo del sistema, un símbolo del sistema de Bash o un símbolo del sistema de PowerShell, inicie sesión en la cuenta de Azure con alguno de los comandos siguientes y siga las indicaciones.

   ```bash
   az login
   ```

### <a name="deploy-the-solution-using-azbb"></a>Implementación de la solución con AZBB

Para implementar las máquinas virtuales de Windows en una arquitectura de referencia de la aplicación de N niveles, siga estos pasos:

1. Vaya a la carpeta `virtual-machines\n-tier-windows` del repositorio que clonó en el paso 1 donde se detallaron los requisitos previos.

2. El archivo de parámetros especifica un nombre de usuario administrador y una contraseña predeterminados para cada máquina virtual de la implementación. Debe cambiar estos datos antes de implementar la arquitectura de referencia. Abra el archivo `n-tier-windows.json` y reemplace los campos **adminUsername** y **adminPassword** con la nueva configuración.
  
   > [!NOTE]
   > En esta implementación existen varios scripts que se ejecutan tanto en los objetos **VirtualMachineExtension** como en la configuración de las **extensiones** de algunos objetos **VirtualMachine**. Algunos de estos scripts requieren el nombre de usuario administrador y la contraseña que acaba de cambiar. Le recomendamos que los revise para asegurarse de que especificó las credenciales correctas. Recuerde que es posible que se produzca un error en la implementación si no especifica las credenciales adecuadas.
   > 
   > 

Guarde el archivo.

3. Implemente la arquitectura de referencia mediante la herramienta de línea de comandos **azbb** tal y como se muestra a continuación.

   ```bash
   azbb -s <your subscription_id> -g <your resource_group_name> -l <azure region> -p n-tier-windows.json --deploy
   ```

Para obtener más información sobre la implementación de esta arquitectura de referencia de ejemplo mediante Azure Bulding Blocks, visite el [repositorio de GitHub][git].


<!-- links -->
[dmz]: ../dmz/secure-vnet-dmz.md
[multi-dc]: multi-region-sql-server.md
[n-tier]: n-tier.md
[azbb]: https://github.com/mspnp/template-building-blocks/wiki/Install-Azure-Building-Blocks
[azure-administration]: /azure/automation/automation-intro
[azure-availability-sets]: /azure/virtual-machines/virtual-machines-windows-manage-availability#configure-each-application-tier-into-separate-availability-sets
[azure-cli]: /azure/virtual-machines-command-line-tools
[azure-cli-2]: https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest
[azure-dns]: /azure/dns/dns-overview
[azure-key-vault]: https://azure.microsoft.com/services/key-vault
[host bastión]: https://en.wikipedia.org/wiki/Bastion_host
[CIDR]: https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
[chef]: https://www.chef.io/solutions/azure/
[git]: https://github.com/mspnp/template-building-blocks
[github-folder]: https://github.com/mspnp/reference-architectures/tree/master/virtual-machines/n-tier-windows
[lb-external-create]: /azure/load-balancer/load-balancer-get-started-internet-portal
[lb-internal-create]: /azure/load-balancer/load-balancer-get-started-ilb-arm-portal
[load-balancer-external]: /azure/load-balancer/load-balancer-internet-overview
[load-balancer-internal]: /azure/load-balancer/load-balancer-internal-overview
[nsg]: /azure/virtual-network/virtual-networks-nsg
[operations-management-suite]: https://www.microsoft.com/server-cloud/operations-management-suite/overview.aspx
[plan-network]: /azure/virtual-network/virtual-network-vnet-plan-design-arm
[private-ip-space]: https://en.wikipedia.org/wiki/Private_network#Private_IPv4_address_spaces
[dirección IP pública]: /azure/virtual-network/virtual-network-ip-addresses-overview-arm
[puppet]: https://puppetlabs.com/blog/managing-azure-virtual-machines-puppet
[ref-arch-repo]: https://github.com/mspnp/reference-architectures
[sql-alwayson]: https://msdn.microsoft.com/library/hh510230.aspx
[sql-alwayson-force-failover]: https://msdn.microsoft.com/library/ff877957.aspx
[sql-alwayson-getting-started]: https://msdn.microsoft.com/library/gg509118.aspx
[sql-alwayson-ilb]: /azure/virtual-machines/windows/sql/virtual-machines-windows-portal-sql-alwayson-int-listener
[sql-alwayson-listeners]: https://msdn.microsoft.com/library/hh213417.aspx
[sql-alwayson-read-only-routing]: https://technet.microsoft.com/library/hh213417.aspx#ConnectToSecondary
[sql-keyvault]: /azure/virtual-machines/virtual-machines-windows-ps-sql-keyvault
[vm-sla]: https://azure.microsoft.com/support/legal/sla/virtual-machines
[vnet faq]: /azure/virtual-network/virtual-networks-faq
[wsfc-whats-new]: https://technet.microsoft.com/windows-server-docs/failover-clustering/whats-new-in-failover-clustering
[Nagios]: https://www.nagios.org/
[Zabbix]: http://www.zabbix.com/
[Icinga]: http://www.icinga.org/
[visio-download]: https://archcenter.blob.core.windows.net/cdn/vm-reference-architectures.vsdx
[0]: ./images/n-tier-sql-server.png "Arquitectura de n niveles con Microsoft Azure"
[resource-manager-overview]: /azure/azure-resource-manager/resource-group-overview 
[vmss]: /azure/virtual-machine-scale-sets/virtual-machine-scale-sets-overview
[load-balancer]: /azure/load-balancer/load-balancer-get-started-internet-arm-cli
[load-balancer-hashing]: /azure/load-balancer/load-balancer-overview#load-balancer-features
[vmss-design]: /azure/virtual-machine-scale-sets/virtual-machine-scale-sets-design-overview
[subscription-limits]: /azure/azure-subscription-service-limits
[availability-set]: /azure/virtual-machines/virtual-machines-windows-manage-availability
[health-probes]: /azure/load-balancer/load-balancer-overview#load-balancer-features
[health-probe-log]: /azure/load-balancer/load-balancer-monitor-log
[health-probe-ip]: /azure/virtual-network/virtual-networks-nsg#special-rules
[network-security]: /azure/best-practices-network-security
