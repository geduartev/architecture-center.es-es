---
title: Principios de diseño para las aplicaciones de Azure
description: Principios de diseño para las aplicaciones de Azure
author: MikeWasson
ms.openlocfilehash: 462896098c668c0775464ca498925266cd73c6e1
ms.sourcegitcommit: 26b04f138a860979aea5d253ba7fecffc654841e
ms.translationtype: HT
ms.contentlocale: es-ES
ms.lasthandoff: 06/19/2018
ms.locfileid: "36206811"
---
# <a name="design-principles-for-azure-applications"></a>Principios de diseño para las aplicaciones de Azure

Siga estos principios de diseño para que la aplicación sea más escalable, resistente y administrable. 

**[Diseñe para la recuperación automática](self-healing.md)**. En un sistema distribuido, se producen errores. Diseñe la aplicación para que se recupere automáticamente cuando esto suceda.

**[Haga que todo sea redundante](redundancy.md)**. Cree redundancia en la aplicación, para evitar tener puntos únicos de error.
 
**[Minimice la coordinación](minimize-coordination.md)**. Minimice la coordinación entre los servicios de aplicación para lograr escalabilidad.
 
**[Diseñe el escalado horizontal](scale-out.md)**. Diseñe la aplicación para que pueda escalarse horizontalmente, agregando o quitando nuevas instancias a medida que se requiera.

**[Cree particiones alrededor de límites](partition.md)**. Use particiones para evitar los limites en la base de datos, la red y el proceso.

**[Diseñe para las operaciones](design-for-operations.md)**. Diseñe la aplicación para que el equipo de operaciones tenga las herramientas que necesita.

**[Use servicios administrados](managed-services.md)**. Cuando sea posible, use la plataforma como servicio (PaaS) en lugar de la infraestructura como servicio (IaaS).

**[Use el mejor almacén de datos para el trabajo](use-the-best-data-store.md)**. Elija la tecnología de almacenamiento que encaje mejor con sus datos y el modo en que se utilizarán. 
 
**[Diseñe para evolucionar](design-for-evolution.md)**. Todas las aplicaciones correctas cambian con el tiempo. Un diseño evolutivo es clave para una innovación continua.

**[Cree teniendo en cuenta las necesidades de la empresa](build-for-business.md)**. Cada decisión de diseño debe estar justificada por un requisito empresarial.

