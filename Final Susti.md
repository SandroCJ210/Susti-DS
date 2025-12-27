### FINAL SUSTI
**Nombres y Apellidos:** Sandro Alfredo Carrillo Jordán
**Código:** 20221679K

## 1

Pues, por lo que veo en el contexto, el sistema debe de estar bien protegido y controlado. Se trata de un sistema local que, si no funciona bien, ya sea haciendo leak de los datos de los clientes o simplemente dando mala información, puede traer problemas fuertes. Entonces, IaC es bastante importante en este contexto, ya que da las facilidades para tener un buen software en el que se minimicen los errores. Con respecto a la reproducibilidad, el poder tener un entorno idéntico, con ayuda de terraform o make, 
aporta bastante, ya que se evita el típico error de que funciona en cierta máquina y en otra no. Aparte con la ayuda de la trazabilidad podríamos tener las condiciones exactas de cada entorno por el cual el sistema ha pasado, esto facilita en caso se quieran probar versiones pasadas en mismas condiciones para asegurar distintas pruebas. Luego, con el uso de SBOM, que es, como dicen las lecturas, como una lista de ingredientes del software, con esto podemos tener en cuenta por ejemplo si es que hay vulnerabilidades con las dependencias. Por último, las firmas, hechas con cosign, y provenance que es un metadato, sirven para saber quién hizo qué cambios, de donde vienen, ¿es original?, y más información relacionada al artefacto, ayudan a saber de que el software no ha sido modificado por externos. Estos tres últimos forman parte de la supply chain, por lo que forman la creación y consumo del código, además de hacerlo de forma segura. Como ejemplo tengamos, lo que mencioné al principio, el sistema es acusado de dar mal la información, digamos que se acusa de que el sistema dice que un paciente no tiene una condición, cuando en realidad sí. Entonces, lo que pasaría es, teniendo en cuenta de que el código del sistema funciona correctamente, pueden decir que ha estado corriendo en un entorno diferente al que debería, para esto se puede usar SBOM y tener más información sobre las dependencias y componentes en general del software en ese preciso momento, al cumplir con reproducibilidad se podría comprobar de que el sistema funcionó correctamente. O quizá pueden decir que se usó una versión que aún estaba con muchos bugs, se puede usar provenance (de nuevo también usando SBOM para tener esta información) y asegurarse de que la versión usada es la correcta.

## 2 

El qué se desplegó. Aunque no haya tag, el haber usado cosign para firmar el artefacto(como la imagen con su digest, la firma, etc. ) puede darnos idea de lo que se ha desplegado. Como se mencionan clústeres, en kubernetes el id único es el digest, entonces también podemos usar esto para saber qué es lo que se ha desplegado. 

Cuándo se desplegó. Para esto se pueden usar tres cosas, el tiempo del build, que para verificar esto se puede usar cosign o fijarse directamente cuándo se ejecutó el pipeline; el tiempo en el que se hizo el release con su tag firmado, esto se puede verificar con un git log para ver la hora del commit y por último el tiempo de despliegue en el cluster. 

Por quién. Se revisa fácilmente con los PRs hechos, además de comprobar las builds usando la firma que se hizo en los pipelines. Todo esto se puede verificar en GitHub y usando cosign. 

Con qué dependencias. Para esto se puede usar SBOM, por las razones que mencioné en la pregunta pasada. Se usaría la herramienta syft para generar el SBOM e incluso si se quieren verificar vulnerabilidades se puede usar Grype y escanear el SBOM.  

Entonces, para que lo descubierto no haga problemas, se tiene que asegurar uso de firmas, PRs, trazabilidad en Git, aprovechar los digest de kubernetes, provenance, y SBOM.

## 3

- Todo commit debe de estar firmado por el que lo hizo. Esto debe de ser un paso obligatorio, sin esto no debería de poder hacerse merge. 
- Debe de activarse que los PRs necesiten dos aprobaciones en la configuración de ramas. Además de usar branch protection para evitar merge --force que puedan romper el programa. También se puede usar CODEOWNERS para esto, ya que da el requerimiento de que los PRs sean revisados obligatoriamente por ciertos equipos.
- Con respecto a la seguridad, primero creo que sería bueno usar los análisis más comunes, los de linteo y formateo, con poner ruff y black en el pipeline basta. También hay que hacer uso de políticas como OPA y Conftest para manejar secretos. Nuevamente, el uso de SBOM junto con Grype para su análisis es bastante importante. También para evitar drift siempre se puede hacer uso de terraform plan para verificar la diferencia entre el estado real y lo que se tiene en la infraestructura, dry run puede servir también para hacer pruebas con python. 
- Con respecto a los secretos, es bastante importante que se haga uso de Kubernetes Vault y/o Secrets. Además de que se puede usar el 3er factor y tener los secretos inyectados al código desde variables del enviroment. Nuevamente, el uso de OPA/Conftest es bastante importante para manejar secretos. También es necesario un TTL corto para reducir la ventana de ataques. Que las claves se roten cada ciclo de sincronización exitosa. Hacer uso de hooks pre-commit con GitGuardian o TruffleHog.
- Creo que el uso de snake case estaría bien para esto, ya que HCL no es como otros lenguajes, para esto no sentiría cómodo el uso de CamelCase por sus mayúsculas, aparte es más común creo. Además se debe de hacer uso de terraform_validate, además de aplicar retención. 

Un antipatrón silencioso sería un TTL largo, como dije, es recomendado un TTL corto para que haya menos riesgo de ataques. Que sea largo solo hace que los datos de los pacientes estén más expuestos, por lo que agranda la posibilidad de que haya sabotaje o alteración de datos. 

Otro antipatrón que es bastante común sería el de la modificación manual, digamos que se hace un cambio manual importante que arregla cierta situación, esto va a generar drift, a la hora de usar el sistema en producción, este cambio no se verá expuesto, por lo que la situación o bug arreglado va a seguir ocurriendo.

## 4

Builder:

Se tendría que hacer un ImageBuilder que obtenga todos los datos necesarios para hacer un dockerfile además de usar firmas y scaneo por SBOM para mantener la seguridad en este entorno crítico. También haría el hardening por defecto. Sería algo como

```
ImageBuilder("python-slim").workdir("app").install_requirements().port(8080).scan_sbom().sign()
```

Esto desde ya ayuda a evitar errores humanos y tiempo, ya que está automatizado, hace el hardening por sí mismo, junto con la firma, esto evita que el developer olvide estos pasos. Además, al tener toda la estructura de las imágenes en el builder, no habría necesidad de revisar las imágenes una por una, por lo que se reduce el costo de auditoría. Con respecto al tiempo de recuperación, si hay algún error simplemente se puede modificar el builder y generar las imágenes de nuevo.

Prototype:

Para esto se puede tener el prototipo base. Cómo funcionaría el clonado? Pues se usaría para el cambio de entornos. Es decir, al crear entornos nuevos clonamos el prototipo y ajustamos parámetros, así podemos hacer varias pruebas con el uso de parametryze. 

Evita tener errores humanos al tener una plantilla de entorno para cada prototipo, así no nos saltamos ninguna configuración importante y solo cambiamos los parámetros que tengamos que cambiar. Además, similar al caso del builder, el costo de auditoría disminuye debido a que se tiene la plantilla de los entornos en el prototipo base, de ahí solo tendrían que revisarse pequeños cambios en los clones de este. Para el tiempo de recuperación se tiene lo que se mencionó, se pueden recrear entornos rápidamente, entonces con esto podemos reproducir bugs, arreglarlos y luego eliminar el entorno, se acelera el proceso.

Composite:

Podemos tomar cada una de las métricas de todo el sistema y colocarlas en un árbol, en las hojas estarían métricas puntuales, como latencia, logs, etc. Cada microservicio estaría en el árbol, si alguna métrica de estos microservicios da error, entonces el estado del nodo del microservicio cambiaría a "Warning" o algo similar que indique que no está sano al 100% y se tendrían logs para añadir trazabilidad de estos errores. 

Con respecto a los errores humanos, esto evita que se pase por alto algún fallo en alguna métrica o servicio del sistema, propagándo estos fallos hacia arriba en el árbol y haciendo mejor la visibilización. El costo de auditoría también se reduce, debido a que, con ayuda de los logs y su trazabilidad, encontrar los errores sería bastante sencillo. El tiempo de recuperación disminuye gracias a que es bastante sencillo encontrar la raíz del error para así poder arreglarlo lo más pronto posible. 

## 5 

Se puede usar un mediator, como ClinicManager, que conecte los tres microservicios, adqusition-service, inference-service y audit-service, para así no sea necesario que se conozcan. Así incluso se puede implementar DI e IoC por los datos que se pueden pasar entre sí. Además, para evitar que ClinicManager dependa de inference-service, usamos una interfaz IDiagnosticModelEngine que implemente inference-service para que así, si es que se cambia el modelo de IA, no se tenga que recompilar ningún código. Además todos los cambios que se hagan van a ser versionados, tan solo un pequeño cambio obligará a crear otra versión del programa para que así haya mejor trazabilidad y no se rompa la historia.

Una invariante sería un identificador para la traza, para así saber exáctamente cómo es que se sacan los diagnósticos, un traceId. Tampoco el nombre o id del paciente no debería de cambiar para un solo paciente. 

Cada microservicio cumple exactamente con lo que debe de cumplir. El audit sería responsable de la auditoría, firmas, horas, etc. El inference sería el encargado de toda la parte del modelo, no estaría conectado a ninguna base de datos ni nada, o sea, stateless. Acquisition estaría únicamente encargado de la señal, este servicio no sabe nada del paciente. 

## 6

- seguridad: Shift-left, hacer la seguridad antes de mandarlo a producción. Uso de linteo y formateo, terraform-validate, checkov, también uso de TruffleHog y GitGuardián así como expliqué en respuestas pasadas. Usar terraform plan y dry run para minimizar drift y testeos. 

- Pruebas rollback con datos sensibles: Se puede desplegar usando blue-green para más seguridad y usar datos "sensibles". Si no funciona simplemente no se pasa el tráfico a green y se hace rollback para arreglar los errores. 

- E2E: Levantar un entorno, levantar el sistema en este entorno y mandar una señal para así verificar que todos los microservicios (junto con el ClinicManager) conectan correctamente y de ahí verificar que audit hizo la firma y el diagnóstico esperado. Para el teardown hay que usar terraform destroy y de ahí verificar que todos los volúmenes y datos fueron borrados correctamente. 

- Políticas OPA: Audit service no debe de estar conectado a internet, es algo bastante importante para mantener la seguridad de los pacientes. Rechazar cualquier imagen que no haya sido firmada correctamente y que no tenga SBOM. Negar db sin cifrado, y conecciones deprecated. 

Defectos detectables:
- IP pública.
- Filtración de secretos
- Drift
- Imagen vulnerable que tiene CVE crítico. 
- Errores de linteo o formateo

No detectables:
- Que se usen términos médicos incorrectamente, para esto debe de haber revisión por parte de especialistas.
- Ya que el plan es de IaC, esto no asegura de que la IA funcione correctamente, así que no detectaría si hay falsos positivos.
- Actos maliciosos manuales de parte de alguien del equipo, el plan no sabría detectar esto ya que está siendo modificado manualmente.

## 7

- Docker

Usar User app, User root trae problemas según las lecturas del curso debido a la compartición de kernel. También aplicar cap_drop=["ALL"] para eliminar privilegios heredados del kernel. Hacer que readOnlyRootFileSystem sea true, así el contenedor no puede modificar su config en tiempo de ejecución. No usar latest porque este para cambiando. 

- Kubernetes

Hacer uso de un controlador de admisión como Kyverno u OPA para bloquear cargas de trabajo inseguras. Habilitar Encryption at Rest en vez de usar base64 que se puede decodificar fácilmente. Cada servicio debe de tener su propia ServiceAccountl, no se debe de usar la cuenta default y deshabilitar el montaje automático de tokens. 

- NetworkPolicies

Se usa Default Deny All para que ningún pod hable con nadie ni se conecte a internet. Políticas: acquisition-service  solo se conecta a ClinicalKernel. ClinicalKernel se conecta hacia audit-service. Audit-service no se conecta a nadie.

- Estrategia de actualización

Como ya mencioné en preguntas pasadas, lo mejor sería usar blue/green, ya que el sistema trata de salud, necesitamos de que todos los resultados sean los mejores, por eso no es pertinente usar el Canary, ya que, aunque sea solo un pequeño porcentaje, es muy riesgoso que ese pequeño porcentaje sea la prueba. En blue green, en cambio, los testeos no son hechos por los usuarios, sino por la gente de QA o testers, así no se corre el riesgo de diagnosticar mal a alguien, además de que si hay algún error hay un roll instantáneo.

- Alertas clínicas

Definir alertas con respecto a superación de algún umbral, a que no se hizo una auditoría completa, a rechazo de firmas, cantidad masiva de diagnóstico de patologías con respecto al promedio, etc.

- Evidencia de no repudio

Para esto se pueden usar las firmas, el metadato provenance que esté firmado y tenga información del commit, hash de la imagen y logs del pipeline CI/CD, el registo de los clúster o dockerfiles, uso de digest, trazado del sistema.

