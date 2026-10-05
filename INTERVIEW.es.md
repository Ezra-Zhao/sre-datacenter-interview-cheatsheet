# Guía de Entrevistas SRE / Operaciones de Centro de Datos

> Hoja de referencia rápida para entrevistas de SRE, DevOps e ingenieros de infraestructura de centros de datos.
> Formato: **Pregunta → respuesta breve de referencia → preguntas para profundizar.**
> Todo el contenido es original, redactado desde cero por el autor.

---

## 🐧 Linux: núcleo y sistema

### P1. ¿Cuál es la diferencia entre un proceso y un hilo?
**Respuesta:** Un proceso tiene su propio espacio de memoria virtual, descriptores de archivo y tabla de señales; los hilos de un mismo proceso comparten memoria, descriptores y la mayoría de los recursos, pero cada uno tiene su propia pila y registros. Crear/destruir un hilo es mucho más barato que un proceso.
**Para profundizar:** ¿Cuándo preferirías multiproceso sobre multihilo? (Aislamiento ante fallos, evitar el GIL en Python, seguridad.)

### P2. ¿Qué hace el OOM killer y cómo decides qué proceso mata?
**Respuesta:** Cuando el sistema se queda sin memoria (física + swap), el núcleo invoca al OOM killer, que puntúa cada proceso con `oom_score` (basado en memoria usada, tiempo de ejecución, `oom_score_adj`, si es root, etc.) y mata al de mayor puntuación para liberar memoria y evitar un cuelgue total.
**Para profundizar:** ¿Cómo protegerías un proceso crítico del OOM killer? (`echo -1000 > /proc/<pid>/oom_score_adj`.) ¿Qué ves en los logs? (`dmesg | grep -i oom`.)

### P3. ¿Qué es un descriptor de archivo y qué significa el error "too many open files"?
**Respuesta:** Todo en Linux (archivos, sockets, pipes) se accede mediante un descriptor de archivo, un entero que indexa la tabla del proceso. Cada proceso y el sistema tienen límites (`ulimit -n`, `/proc/sys/fs/file-max`). El error aparece cuando un proceso agota su límite, típico en servicios con muchas conexiones concurrentes.
**Para profundizar:** ¿Cómo lo diagnosticas en caliente? (`lsof -p <pid> | wc -l`, `/proc/<pid>/fd`.) ¿Solución temporal vs definitiva? (Subir `LimitNOFILE` en systemd + revisar fugas de conexiones.)

### P4. `df` dice que hay espacio libre pero `du` dice que el disco está lleno (o al revés). ¿Por qué?
**Respuesta:** Causa clásica: un archivo borrado (`rm`) que sigue abierto por un proceso; `df` cuenta bloques del sistema de archivos (el espacio no se libera hasta cerrar el fd), `du` suma lo que ve en el árbol de directorios. También puede ser inodos agotados (`df -i`): disco con espacio pero sin inodos libres.
**Para profundizar:** ¿Cómo encuentras al culpable? (`lsof | grep deleted`.) ¿Cómo liberas espacio sin reiniciar el servicio? (`: > /proc/<pid>/fd/<n>`.)

### P5. ¿Diferencia entre SIGTERM y SIGKILL? ¿Cómo harías un apagado elegante de un servicio?
**Respuesta:** SIGTERM (15) pide al proceso que termine y puede ser capturado para limpiar recursos (cerrar conexiones, vaciar buffers); SIGKILL (9) lo mata de inmediato sin darle oportunidad, no se puede capturar ni ignorar. Apagado elegante: enviar SIGTERM, esperar un timeout razonable, y solo entonces SIGKILL.
**Para profundizar:** ¿Qué pasa si un contenedor ignora SIGTERM? (Kubernetes espera `terminationGracePeriodSeconds` y luego hace SIGKILL.)

### P6. ¿Qué significan los tres números del `load average`?
**Respuesta:** Promedio de procesos en estado ejecutable o en espera ininterrumpida (I/O) durante 1, 5 y 15 minutos. En un servidor de N núcleos, un load sostenido muy por encima de N indica saturación. Ojo: un load alto con CPU ociosa suele ser espera de disco/red.
**Para profundizar:** ¿Cómo distingues si el load viene de CPU o de I/O? (`top`: `%wa`; `iostat -x`.)

### P7. ¿Qué es un proceso zombi y cómo lo eliminas?
**Respuesta:** Un proceso que terminó pero cuyo padre no leyó su estado de salida con `wait()`; queda como entrada en la tabla de procesos consumiendo solo un slot. No se mata con señales: hay que matar o arreglar al proceso padre para que lo "recoja"; si el padre es PID 1 y no lo recoge, toca reiniciar el servicio o el nodo.
**Para profundizar:** ¿Qué herramienta lo confirma? (`ps aux | grep 'Z'`, estado `Z+`.)

### P8. ¿Qué aportan los cgroups y los namespaces a los contenedores?
**Respuesta:** Los namespaces aíslan lo que un proceso *ve* (PID, red, montajes, usuarios, hostname); los cgroups limitan y miden lo que *usa* (CPU, memoria, I/O, pids). Juntos son la base técnica de Docker/Kubernetes sin necesidad de un hipervisor.
**Para profundizar:** ¿Qué pasa si un contenedor no tiene límite de memoria? (Puede disparar el OOM killer del nodo y matar otros pods.)

---

## 🌐 Redes

### P9. Describe el three-way handshake de TCP y el cierre en cuatro pasos. ¿Por qué el cierre necesita un paso más?
**Respuesta:** Apertura: SYN → SYN+ACK → ACK (3 pasos sincronizan números de secuencia en ambas direcciones). Cierre: FIN → ACK → FIN → ACK (4 pasos) porque cada lado cierra su dirección de envío de forma independiente; tras recibir un FIN, un lado puede aún tener datos que enviar antes de mandar su propio FIN.
**Para profundizar:** ¿Qué es un SYN flood y cómo se mitiga? (SYN cookies, `net.ipv4.tcp_syncookies`.)

### P10. El servidor acumula miles de conexiones en TIME_WAIT. ¿Es un problema y cómo lo manejas?
**Respuesta:** TIME_WAIT (típicamente 60 s) existe para que los últimos ACK lleguen y evitar que paquetes viejos contaminen conexiones nuevas. Miles de TIME_WAIT en un servidor con mucho tráfico saliente (p. ej. un proxy) pueden agotar puertos efímeros. Mitigación: reutilizar conexiones (keep-alive, pools), ajustar `tcp_tw_reuse` (seguro en cliente), nunca `tcp_tw_recycle` (eliminado del kernel por romper NAT).
**Para profundizar:** ¿TIME_WAIT del lado servidor con keep-alive es normal? (Sí, si el cliente cierra primero.)

### P11. Describe el flujo completo de una resolución DNS.
**Respuesta:** 1) Caché local (navegador/SO) → 2) resolver recursivo (p. ej. 8.8.8.8) → 3) si no está en su caché, consulta iterativa: servidor raíz → TLD (`.com`) → autoritativo del dominio → 4) el recursivo cachea según el TTL y responde al cliente. Cada nivel puede tener caché con su propio TTL.
**Para profundizar:** ¿Cómo depuras un fallo DNS? (`dig +trace`, `nslookup`, revisar `/etc/resolv.conf`, comprobar si es fallo del autoritativo o del recursivo.)

### P12. Compara algoritmos de balanceo de carga: round-robin, least-connections y consistent hashing.
**Respuesta:** Round-robin reparte por turnos (simple, asume backends homogéneos); weighted round-robin pondera por capacidad; least-connections envía al backend con menos conexiones activas (mejor con sesiones de duración variable); consistent hashing mapea claves a nodos en un anillo para que al añadir/quitar un nodo solo se reubique una fracción de las claves (ideal para cachés).
**Para profundizar:** ¿Balanceo en capa 4 o capa 7? (L4: rápido, por IP/puerto; L7: enruta por URL/cabecera/cookie, permite sticky sessions y terminación TLS.)

### P13. ¿Qué significan 502, 503 y 504? ¿Cómo los distingues al depurar?
**Respuesta:** 502 Bad Gateway: el proxy recibió una respuesta inválida del upstream (proceso caído, cabeceras corruptas). 503 Service Unavailable: el upstream responde que no puede atender (sobrecargado, en despliegue, circuito abierto). 504 Gateway Timeout: el upstream no respondió a tiempo. Clave: 502 = respuesta mala, 503 = "no puedo", 504 = "no contestó".
**Para profundizar:** ¿Dónde miras primero? (Logs del proxy con `upstream_response_time` / `upstream_status` en Nginx.)

### P14. `traceroute` muestra asteriscos (`* * *`) en varios saltos. ¿Significa que hay un problema?
**Respuesta:** No necesariamente: muchos routers limitan o ignoran los ICMP/UDP de sonda por política, así que no responden aunque reenvíen el tráfico perfectamente. Solo es problema si el destino final tampoco responde o hay pérdida sostenida a partir de un salto concreto.
**Para profundizar:** ¿Alternativas? (`mtr` combina ping + traceroute; mirar pérdida *acumulada* por salto, no puntual.)

### P15. ¿Qué es el MTU y qué síntomas da un problema de MTU mal configurado?
**Respuesta:** Maximum Transmission Unit: tamaño máximo de paquete en un enlace (típico 1500 en Ethernet). Si un paquete grande no puede fragmentarse (bit DF activo) y no hay Path MTU Discovery (ICMP bloqueado), la conexión se "cuelga": el handshake funciona pero la transferencia de datos se queda congelada (síntoma clásico tras un túnel VPN/GRE mal configurado).
**Para profundizar:** ¿Cómo lo confirmas? (`ping -M do -s 1472 <destino>` para probar el PMTU.)

---

## 🏢 Centro de datos: energía, refrigeración y hardware

### P16. ¿Qué es el PUE y qué valor se considera bueno?
**Respuesta:** Power Usage Effectiveness = energía total del centro de datos / energía que llega a los equipos de TI. Un PUE de 1.0 sería perfecto; la industria moderna anda entre 1.2 y 1.5; por encima de 2.0 hay mucho margen de mejora (refrigeración ineficiente, UPS sobredimensionados).
**Para profundizar:** ¿Qué medidas bajan el PUE? (Contención de pasillos, free cooling, subir la temperatura de consigna según ASHRAE, variadores en ventiladores.)

### P17. ¿Cómo está diseñada la energía de un rack crítico? (Doble acometida, UPS, generador)
**Respuesta:** Doble acometida eléctrica (A y B) desde fuentes independientes, cada servidor con doble fuente (una a cada acometida); UPS en línea que filtra y sostiene la carga ante microcortes; generadores diésel con arranque automático y autonomía de combustible para cortes largos. Todo el camino es redundante N+1 o 2N según el Tier.
**Para profundizar:** ¿Qué pasa si una PDU falla? (La otra acometida asume el 100%; por eso cada acometida se dimensiona para la carga total.)

### P18. Explica el concepto de pasillo frío / pasillo caliente.
**Respuesta:** Los racks se orientan alternando la toma de aire frío (frontal) y la expulsión de aire caliente (trasera), formando pasillos fríos (impulsión) y calientes (retorno). Evita que un servidor aspire el aire caliente del vecino, mejora la eficiencia del CRAC/CRAH y permite subir la consigna general.
**Para profundizar:** ¿Qué temperatura/humedad recomienda ASHRAE? (18–27 °C, humedad relativa 20–80%, punto de rocío controlado para evitar condensación y descargas.)

### P19. Compara RAID 5, RAID 6 y RAID 10. ¿Cuándo usarías cada uno?
**Respuesta:** RAID 5: paridad distribuida, tolera 1 disco fallido, buena capacidad útil, pero reconstrucciones largas y arriesgadas en discos grandes. RAID 6: doble paridad, tolera 2 fallos, más seguro para discos de alta capacidad, penaliza escritura. RAID 10 (1+0): espejo + striping, tolera fallos múltiples según distribución, el mejor rendimiento de escritura/lectura, pero solo ~50% de capacidad útil.
**Para profundizar:** ¿Por qué RAID 5 está desaconsejado con discos de 8 TB+? (Probabilidad de un segundo fallo o error irrecuperable —URE— durante una reconstrucción de muchas horas.)

### P20. Un servidor se reinicia solo de forma aleatoria. ¿Cuál es tu flujo de diagnóstico de hardware?
**Respuesta:** 1) Revisar logs del SO y del BMC/IPMI (`ipmitool sel list`) buscando errores de hardware; 2) comprobar temperaturas y ventiladores; 3) test de memoria (ECC corrige 1 bit y registra; múltiples errores corregibles = DIMM sospechoso); 4) SMART de discos; 5) fuentes de alimentación y PDU; 6) aislar por sustitución (mover la carga, cambiar DIMM de slot). Documentar cada paso.
**Para profundizar:** ¿Qué significa un SEL lleno de "Memory ECC correctable"? (El DIMM está degradándose: planificar reemplazo antes de que un error incorregible tumbe el nodo.)

### P21. ¿Qué verificarías antes de dar por bueno un servidor recién instalado en rack?
**Respuesta:** Checklist: firmware/BIOS/BMC actualizados; RAID configurado y verificado; test de memoria y de estrés de CPU; red (ambas acometidas, velocidad negociada, VLAN correctas); etiquetado de cables y registro en el CMDB/DCIM; sensores de temperatura funcionando; prueba de las dos fuentes desenchufando una.
**Para profundizar:** ¿Por qué probar desenchufando una fuente? (Es la única forma de verificar que la redundancia A/B es real y no solo "enchufada al mismo circuito".)

---

## ☸️ Kubernetes

### P22. ¿Qué relación hay entre Pod, Deployment y Service?
**Respuesta:** El Pod es la unidad mínima desplegable (uno o más contenedores que comparten red y almacenamiento). El Deployment gestiona un conjunto de Pods idénticos: define la plantilla, el número de réplicas y la estrategia de actualización. El Service da un punto de acceso estable (IP virtual + DNS) a ese conjunto cambiante de Pods mediante selectores de etiquetas.
**Para profundizar:** ¿Qué pasa si borras los Pods de un Deployment a mano? (El ReplicaSet los recrea: el estado deseado manda.)

### P23. Diferencias entre ClusterIP, NodePort y LoadBalancer.
**Respuesta:** ClusterIP: IP virtual solo accesible dentro del clúster (comunicación interna). NodePort: expone el Service en un puerto alto de cada nodo (acceso externo rudimentario). LoadBalancer: aprovisiona un balanceador externo en la nube y le asigna IP pública (el modo estándar para exponer servicios en cloud).
**Para profundizar:** ¿Y el Ingress? (Un único punto de entrada L7 que enruta por host/ruta a múltiples Services ClusterIP.)

### P24. ¿Cómo decide el scheduler en qué nodo poner un Pod?
**Respuesta:** Dos fases: filtrado (descarta nodos que no cumplen: recursos insuficientes, taints sin tolerancia, nodeSelector/affinity no coincidente, puertos en uso) y puntuación (elige el mejor entre los candidatos según recursos disponibles, afinidad, dispersión). Luego el kubelet del nodo lo ejecuta.
**Para profundizar:** ¿Diferencia entre nodeSelector, nodeAffinity y taints/tolerations? (nodeSelector: coincidencia simple; affinity: reglas ricas con preferencia/obligatoriedad; taints: el nodo *repele*, el Pod debe *tolerar*.)

### P25. Un Pod está en CrashLoopBackOff. ¿Cuál es tu rutina de diagnóstico con kubectl?
**Respuesta:** 1) `kubectl describe pod` (eventos: ¿OOMKilled? ¿liveness fallando? ¿imagen no encontrada?); 2) `kubectl logs --previous` (logs del contenedor que murió); 3) revisar probes y límites de recursos en el manifiesto; 4) `kubectl get events --sort-by=.lastTimestamp`; 5) si es la app: reproducir el comando de arranque a mano.
**Para profundizar:** ¿Qué significa exit code 137? (SIGKILL: casi siempre OOM o `kubectl delete`/evicción. Exit 1 = error de la app.)

### P26. ¿Diferencia entre readinessProbe y livenessProbe? ¿Qué pasa si las confundes?
**Respuesta:** readiness dice "estoy listo para recibir tráfico" (si falla, el Pod sale de los endpoints del Service pero sigue vivo); liveness dice "sigo sano" (si falla, kubelet reinicia el contenedor). Si usas liveness donde tocaba readiness, un pico de carga puede provocar reinicios en cascada que empeoran la caída.
**Para profundizar:** ¿Por qué no poner la misma comprobación pesada en ambas? (Una query lenta a la BD como liveness convierte lentitud temporal en reinicios.)

### P27. ¿Cómo funciona el HPA (Horizontal Pod Autoscaler)?
**Respuesta:** Un controlador que consulta periódicamente las métricas (CPU/memoria por defecto, métricas custom con adaptador) y ajusta el número de réplicas del Deployment/ReplicaSet entre un mínimo y un máximo para mantener el valor objetivo. Actúa con retardos de estabilización para evitar oscilaciones.
**Para profundizar:** ¿Por qué el HPA a veces "no reacciona" a un pico súbito? (Ventana de medición + cooldown: está diseñado para tendencias, no para picos instantáneos; para eso, sobredimensiona el mínimo o usa KEDA con métricas externas.)

### P28. ¿Cómo le pasas configuración y secretos a un Pod? ¿Qué no debes hacer nunca con un Secret?
**Respuesta:** ConfigMap para configuración no sensible y Secret para datos sensibles, inyectados como variables de entorno o montados como archivos. Nunca: subirlos al repo en claro, pasarlos por línea de comandos (quedan en el historial), ni loguearlos. En producción, integrarlos con un gestor de secretos externo (Vault, cloud KMS) con rotación.
**Para profundizar:** ¿Los Secrets de Kubernetes están cifrados por defecto? (No: solo base64. Hay que activar cifrado en reposo en etcd.)

---

## 🚨 On-call y respuesta a incidentes

### P29. ¿Cómo defines los niveles de severidad de un incidente (P1–P4)?
**Respuesta:** P1/crítico: servicio caído o datos en riesgo, afecta a muchos usuarios → respuesta inmediata, se despierta a quien sea. P2/alto: degradación importante con workaround. P3/medio: impacto limitado, se atiende en horario. P4/bajo: cosmético o mejora. La clave: criterios objetivos y escritos, no "lo que parezca grave a las 3 AM".
**Para profundizar:** ¿Quién puede declarar un P1? (Cualquiera que detecte los criterios: mejor un falso P1 que un P1 tardío.)

### P30. ¿Qué debe contener un postmortem sin culpa (blameless)?
**Respuesta:** Cronología objetiva, impacto medido (duración, usuarios, SLO quemado), causa raíz técnica *y* sistémica, qué funcionó bien en la respuesta, y acciones correctivas con dueño y fecha. "Sin culpa": se analiza el sistema y los procesos, no a las personas; si alguien teme contarlo todo, el postmortem no sirve.
**Para profundizar:** ¿Cómo evitas que las acciones se queden en un cajón? (Cada acción es un ticket con responsable y fecha de revisión en la próxima revisión de postmortems.)

### P31. Explica SLI, SLO y SLA, y qué es el "error budget".
**Respuesta:** SLI (indicador): métrica medida, p. ej. "99,9% de peticiones < 200 ms en 30 días". SLO (objetivo): el valor que te comprometes internamente. SLA (acuerdo): el compromiso contractual con penalización. Error budget = 1 − SLO: el "presupuesto de fallos" permitido; si se agota, se congelan lanzamientos y se prioriza fiabilidad.
**Para profundizar:** ¿Qué haces cuando el error budget se quema en la primera semana del mes? (Moratoria de deploys no críticos + foco en las causas top.)

### P32. Te pasan el turno de on-call. ¿Qué revisas en el handover?
**Respuesta:** Incidentes abiertos y su estado, cambios recientes en producción (deploys, migraciones), alertas conocidas/false positives, trabajos programados (mantenimientos, backups), y "dónde están los runbooks". Un buen handover se escribe, no se cuenta de palabra a las prisas.
**Para profundizar:** ¿Qué haces si heredas 200 alertas sin reconocer? (Triaje: silenciar duplicados, priorizar por impacto al usuario, y abrir deuda de alertas al día siguiente.)

### P33. En pleno incidente, ¿qué va primero: mitigar o encontrar la causa raíz?
**Respuesta:** Mitigar siempre primero: devolver el servicio a los usuarios (rollback, failover, escalar capacidad, feature flag). La causa raíz se investiga con el servicio ya estable. Perseguir la causa con el servicio caído alarga el impacto y lleva a cambios arriesgados bajo presión.
**Para profundizar:** Frase de referencia: "stop the bleeding first". ¿Cuándo declaras mitigado? (SLI vuelve a verde de forma sostenida, no un pico.)

---

## 📊 Monitoreo

### P34. ¿Cuáles son los cuatro tipos de métricas de Prometheus y cuándo usarías cada uno?
**Respuesta:** Counter: solo sube (peticiones totales, errores totales → para ratios y tasas). Gauge: sube y baja (conexiones activas, temperatura). Histogram: distribuye observaciones en buckets (latencias → percentiles calculables). Summary: percentiles precalculados en el cliente (similar pero no agregable entre instancias).
**Para profundizar:** ¿Por qué un Counter que "baja" rompe tus alertas? (Los reinicios del proceso lo ponen a cero: usa `rate()`/`increase()`, que lo manejan.)

### P35. ¿Cómo diseñas una regla de alerta que no genere fatiga?
**Respuesta:** Alertar sobre síntomas que ve el usuario (tasa de errores, latencia p95, saturación con previsión de impacto), no sobre cada causa interna; umbrales con ventana de tiempo (">1% de errores durante 5 min", no un pico de 10 s); severidad acorde a acción requerida; toda alerta debe tener runbook enlazado. Revisar periódicamente las alertas que nadie atiende y eliminarlas o degradarlas.
**Para profundizar:** ¿Qué es el "alert fatigue" y su riesgo? (Tantas alertas que se ignoran todas: el día del P1 real nadie mira el canal.)

### P36. ¿Qué miden los métodos RED y USE?
**Respuesta:** RED (para servicios): Rate (peticiones/s), Errors (tasa de fallos), Duration (latencia). USE (para recursos): Utilization (% en uso), Saturation (cola/espera), Errors. RED dice "¿el servicio sufre?"; USE dice "¿qué recurso está al límite?".
**Para profundizar:** ¿Cuál usas para un disco? (USE: % lleno, latencia de I/O —saturación—, errores SMART.)

### P37. ¿Diferencia entre monitoreo de caja blanca y caja negra?
**Respuesta:** Caja blanca: métricas/logs/traces desde dentro del sistema (qué está pasando por dentro). Caja negra: sondas externas que verifican el comportamiento visible por el usuario (¿responde la web? ¿el login funciona?). Necesitas ambos: la caja negra te dice *que* algo falla para el usuario; la blanca te dice *por qué*.
**Para profundizar:** Ejemplo de caja negra: (un probe que hace login completo cada minuto desde fuera del centro de datos.)

---

## 🔥 Escenarios reales de troubleshooting

> Cadena de diagnóstico paso a paso. En la entrevista, narra lo que *harías*, en orden, y qué descartas en cada paso.

### E1. El servicio no arranca: "no space left on device", pero `df` muestra espacio libre
**Cadena:** 1) `df -i`: casi seguro son inodos agotados (millones de ficheros pequeños: sesiones, cachés, colas de mail). 2) Localizar: `find /var -xdev | head`, o por directorio con `du --inodes`. 3) Mitigar: rotar/limpiar el directorio culpable, nunca `rm -rf` a ciegas. 4) Si no son inodos: `lsof | grep deleted` (fichero borrado aún abierto) → truncar vía `/proc/<pid>/fd`.
**Cierre:** Acción definitiva: logrotate + monitoreo de inodos (no solo de espacio).

### E2. De repente, miles de conexiones en TIME_WAIT y "cannot assign requested address"
**Cadena:** 1) Confirmar: `ss -s`, `netstat -an | grep TIME_WAIT | wc -l`. 2) Identificar quién abre/cierra tanto (un cliente sin keep-alive o sin pool de conexiones). 3) Mitigar: activar keep-alive y pools en el cliente; `tcp_tw_reuse=1` si el agotamiento es de puertos efímeros en el lado que *inicia* conexiones. 4) Subir `ip_local_port_range` como parche temporal.
**Cierre:** La causa raíz casi siempre está en el cliente, no en el servidor.

### E3. La web va lenta solo para algunos usuarios: sospecha de DNS
**Cadena:** 1) `dig` + `dig +trace` desde una máquina afectada vs una sana: ¿dónde difieren los tiempos? 2) Comparar resolvers (`/etc/resolv.conf`): ¿usan el mismo recursivo? 3) Mirar TTLs: un TTL muy bajo multiplica consultas; un registro envenenado/cambiado tarda en propagar. 4) Comprobar DNSSEC si hay validación fallando.
**Cierre:** Medir antes y después; el DNS es inocente hasta que `dig` demuestre lo contrario.

### E4. Un proceso muere cada noche a la misma hora sin dejar logs: OOM
**Cadena:** 1) `dmesg -T | grep -i "oom\|killed process"`: confirmar que el OOM killer actuó. 2) Correlacionar la hora con un cron (backup, batch, reporte). 3) Revisar el pico con el monitoreo histórico (memoria del proceso creciendo = fuga; pico puntual = carga). 4) Mitigar: mover el batch a un nodo con más RAM o limitarlo; ajustar `oom_score_adj` solo si sabes lo que haces.
**Cierre:** Si es fuga lenta, un reinicio programado es parche; el fix es en el código.

### E5. La VM en la nube va lenta aunque "la CPU está al 30%": steal time
**Cadena:** 1) `top`: mirar `%st` (steal). Un `%st` alto y sostenido = el hipervisor no te da la CPU que pides porque los vecinos la acaparan (noisy neighbor) o el host está sobrevendido. 2) Confirmar con el proveedor/métricas del hipervisor. 3) Mitigar: cambiar a instancia dedicada o a otro host/zona; a largo plazo, dimensionar con margen.
**Cierre:** No es tu culpa ni tu código: es contención del host.

### E6. Un Pod entra en CrashLoopBackOff tras un despliegue
**Cadena:** 1) `kubectl describe pod`: eventos (¿ImagePullBackOff? ¿OOMKilled 137? ¿probe fallando?). 2) `kubectl logs --previous`: el último mensaje antes de morir suele delatarlo (puerto en uso, variable faltante, migración fallida). 3) Comparar con el manifiesto anterior (`kubectl rollout history`). 4) Mitigar: `kubectl rollout undo` para estabilizar; investigar sin presión.
**Cierre:** Rollback primero, root cause después (regla de oro de incidentes).

### E7. Nginx devuelve 502 de forma intermitente
**Cadena:** 1) Logs de Nginx con `upstream_status` y `upstream_response_time`: ¿el upstream responde mal o no responde? 2) Si 502 con tiempo ~0: el backend murió o rechazó la conexión (mirar sus logs, ¿OOM? ¿reinicios?). 3) Si intermitente bajo carga: pool de conexiones del backend agotado o keepalive mal configurado entre proxy y upstream. 4) Revisar `proxy_next_upstream` y health checks.
**Cierre:** El 502 casi siempre es el backend, no el proxy: sigue la cadena aguas arriba.

### E8. Un disco de la RAID parpadea en ámbar: degradación del array
**Cadena:** 1) No entrar en pánico: con RAID 6/10 el array sigue funcionando. 2) Identificar el disco (`megacli`/`storcli`, o la controladora): ¿fallo real o predictive failure por SMART? 3) Verificar que hay hot-spare o pedir el reemplazo YA. 4) Sustituir en caliente y vigilar la reconstrucción (es el momento de más riesgo: un segundo fallo puede ser fatal en RAID 5).
**Cierre:** Nunca reconstruir sin backup verificado si el array ya está degradado.

### E9. A las 00:00 todo deja de funcionar: certificado TLS expirado
**Cadena:** 1) `echo | openssl s_client -connect host:443 -servername host | openssl x509 -noout -dates`: confirmar expiración. 2) Mitigar: renovar/emergencia con el CA (Let's Encrypt: `certbot renew --force-renewal`). 3) Revisar por qué falló la renovación automática (cron caído, desafío HTTP bloqueado, rate limits).
**Cierre:** Acción definitiva: monitoreo de expiración con alerta a 30/14/7 días. Los certificados no deberían sorprender a nadie.

### E10. La base de datos va lenta y la app agota su pool de conexiones
**Cadena:** 1) ¿Es la BD o la app? `SHOW PROCESSLIST` / `pg_stat_activity`: ¿muchas queries en `Waiting` o una sola query monstruosa bloqueando? 2) Si es una query: `EXPLAIN`, índices faltantes, un despliegue que cambió el plan de ejecución. 3) Mitigar: matar la query bloqueante, escalar réplicas de lectura, o activar el circuit breaker en la app. 4) Revisar el tamaño del pool: un pool mayor no arregla una BD lenta, solo esconde el problema.
**Cierre:** El pool agotado es síntoma, no causa: sigue la cadena hasta la query.

---

*¿Te sirvió? Dale una ⭐ y comparte con quien esté preparando entrevistas de SRE/DevOps.*
