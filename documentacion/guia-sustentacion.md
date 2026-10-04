# Guía de sustentación

Cada diapositiva de la presentación tiene un guion en las **notas del orador** (en PowerPoint: Ver → Notas). Esta guía organiza quién presenta cada parte y prepara las preguntas más probables.

## Reparto sugerido (≈ 15 minutos)

| Diapositivas | Tema | Presenta | Tiempo |
|---|---|---|---|
| 1 a 5 | Portada, agenda, reto, historias de usuario, requisitos | Líder de presentación | 3 min |
| 6 a 11 | Visión general, modelo C4 y flujo de una atención | Arquitecto(a) de soluciones | 4 min |
| 12 y 14 | Arquitectura en Azure, alta disponibilidad y monitoreo | Ingeniero(a) cloud y DevOps | 2 min |
| 13 y 15 | Seguridad y decisiones técnicas | Especialista en seguridad | 2 min |
| 16 a 18 | Azure vs AWS, costos y optimización | Analista de costos | 2,5 min |
| 19 a 22 | Plan, riesgos, conclusiones y preguntas | Líder de presentación | 1,5 min |

## Ideas clave que hay que dejar claras

1. Cada historia de usuario es un microservicio independiente.
2. Las enfermeras se autentican contra el sistema de seguridad de la compañía con OpenID Connect; la aplicación nunca ve contraseñas.
3. Si cae una zona, el sistema sigue; si cae toda la región, en menos de una hora opera desde la otra.
4. El asistente de diagnóstico sugiere, pero la decisión clínica siempre es de la enfermera.
5. AWS sale más barato en precio de lista y lo reconocemos; elegimos Azure por la integración con la seguridad corporativa y los servicios de salud.

## Preguntas probables y respuestas

**¿Por qué microservicios y no una aplicación monolítica?**
Porque las historias de usuario tienen cargas muy distintas: el triaje y la consulta de historias se usan en cada llamada, mientras que la remisión solo en el 20% de los casos. Con microservicios cada uno escala por separado, se despliega sin afectar a los demás y una falla en un servicio no tumba toda la aplicación.

**¿Cómo se integra con el sistema de seguridad centralizado?**
Con OpenID Connect. Entra ID se federa con el proveedor de identidad corporativo, que sigue siendo la fuente de verdad. La enfermera inicia sesión allí, recibe un token JWT con su rol y API Management valida ese token en cada petición. Se usa el flujo Authorization Code con PKCE.

**¿Qué pasa si se cae Azure en la región principal?**
Front Door detecta la falla con sondas de salud y envía el tráfico a Central US, donde hay una réplica de la base de datos y un clúster mínimo que escala automáticamente. El objetivo es volver a operar en menos de una hora (RTO) perdiendo como máximo cinco minutos de datos (RPO).

**¿Por qué activo/pasivo y no activo/activo?**
Un activo/activo duplicaría el cómputo y obligaría a escribir en dos bases de datos a la vez, con riesgos de inconsistencia en datos clínicos. Activo/pasivo da la resiliencia necesaria a un costo razonable.

**¿Es seguro enviar datos de pacientes a un modelo de lenguaje?**
No se envían datos que identifiquen al paciente: el anonimizador elimina nombre, documento y dirección antes de la consulta. Azure OpenAI se usa con endpoint privado, dentro de la misma región, y Microsoft no usa esos datos para entrenar modelos. Además, el motor de reglas clínicas detecta los signos de alarma de forma determinista y la enfermera valida siempre.

**¿Por qué Azure si AWS es más barato?**
AWS es cerca de 35% más barato en precio de lista, sobre todo por API Gateway, mensajería y logs. Pero en la matriz ponderada Azure gana (4,60 vs 4,30) por la integración nativa con la identidad corporativa, los servicios de salud (FHIR) y de IA con red privada, y la compatibilidad con el ecosistema Microsoft. Con reservas la brecha se reduce.

**¿De dónde salen los precios?**
De la API oficial de precios minoristas de Azure y de los archivos oficiales de precios de AWS (AWS Price List), consultados en octubre de 2026. Cada fila del Excel tiene el enlace a la página de precios del servicio.

**¿Cómo se calculó el volumen?**
Está en la hoja Supuestos del Excel: 250 enfermeras, unas 24 llamadas por enfermera al día (≈ 180.000 al mes), 20% de remisiones, 80 peticiones API por llamada. Si el consejo cambia un supuesto, todo se recalcula.

**¿Cómo funciona en Windows, Linux y macOS?**
La aplicación es una SPA responsiva en React que corre en cualquier navegador moderno. Para quien necesite una aplicación instalada, la misma SPA se empaqueta con Electron para los tres sistemas operativos.

**¿Por qué FHIR?**
Es el estándar internacional de interoperabilidad en salud (HL7). Permite conectarse con los sistemas del hospital y con otros prestadores sin integraciones a la medida.

**¿Cómo se entera el equipo de que algo falla?**
Azure Monitor tiene alertas para caída de región, latencia mayor a 2 segundos, errores mayores al 2%, remisiones que no se pudieron enviar, alertas de seguridad de Defender y gasto mayor al 90% del presupuesto.

**¿Cómo se habilita después el canal de pacientes?**
Se agrega Entra External ID para el registro de pacientes y una nueva aplicación cliente (web o móvil). Los microservicios existentes no cambian; solo se exponen nuevas APIs en API Management con permisos propios para pacientes.

**¿Qué pasa si el centro médico no recibe el aviso?**
La remisión se publica en Service Bus. El procesador de notificaciones reintenta con espera exponencial y, si sigue fallando, el mensaje pasa a la cola de mensajes fallidos, lo que dispara una alerta para que la supervisora contacte al centro.
