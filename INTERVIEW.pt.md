# Guia de Entrevistas SRE / Operações de Data Center

> Cola rápida para entrevistas de SRE, DevOps e engenheiros de infraestrutura de data centers.
> Formato: **Pergunta → resposta curta de referência → perguntas para aprofundar.**
> Todo o conteúdo é original, escrito do zero pelo autor.

---

## 🐧 Linux: kernel e sistema

### P1. Qual a diferença entre processo e thread?
**Resposta:** Processo tem seu próprio espaço de memória virtual, descritores de arquivo e tabela de sinais; threads do mesmo processo compartilham memória, descritores e quase todos os recursos — cada uma tem apenas sua pilha e registradores. Criar/destruir thread é muito mais barato que processo.
**Para aprofundar:** Quando você preferiria multiprocesso a multithread? (Isolamento contra falhas, fugir do GIL em Python, segurança.)

### P2. O que faz o OOM killer e como ele escolhe qual processo matar?
**Resposta:** Quando acaba a memória (física + swap), o kernel chama o OOM killer, que pontua cada processo com o `oom_score` (memória usada, tempo de execução, `oom_score_adj`, se é root etc.) e mata o de maior pontuação para liberar memória e evitar o travamento total.
**Para aprofundar:** Como proteger um processo crítico? (`echo -1000 > /proc/<pid>/oom_score_adj`.) Onde aparece no log? (`dmesg | grep -i oom`.)

### P3. O que é um file descriptor e o que significa "too many open files"?
**Resposta:** No Linux, tudo (arquivos, sockets, pipes) é acessado por um file descriptor — um inteiro que indexa a tabela do processo. Processo e sistema têm limites (`ulimit -n`, `/proc/sys/fs/file-max`). O erro aparece quando o processo estoura o limite — clássico em serviços com muitas conexões concorrentes.
**Para aprofundar:** Como diagnostica em produção? (`lsof -p <pid> | wc -l`, `/proc/<pid>/fd`.) Solução paliativa vs definitiva? (Subir `LimitNOFILE` no systemd + caçar vazamento de conexões.)

### P4. O `df` diz que há espaço livre, mas o `du` diz que o disco está cheio (ou o inverso). Por quê?
**Resposta:** Causa clássica: arquivo deletado (`rm`) ainda aberto por um processo — o `df` conta blocos do filesystem (o espaço só libera ao fechar o fd), o `du` soma o que vê na árvore de diretórios. Também pode ser inodes esgotados (`df -i`): disco com espaço, mas sem inodes livres.
**Para aprofundar:** Como acha o culpado? (`lsof | grep deleted`.) Como libera espaço sem reiniciar o serviço? (`: > /proc/<pid>/fd/<n>`.)

### P5. Diferença entre SIGTERM e SIGKILL? Como fazer um shutdown gracioso?
**Resposta:** SIGTERM (15) pede ao processo para terminar e pode ser capturado para limpeza (fechar conexões, descarregar buffers); SIGKILL (9) mata na hora, sem chance de reação — não pode ser capturado nem ignorado. Shutdown gracioso: manda SIGTERM, espera um timeout razoável e só então SIGKILL.
**Para aprofundar:** E se um container ignorar o SIGTERM? (O Kubernetes espera `terminationGracePeriodSeconds` e depois manda SIGKILL.)

### P6. O que significam os três números do `load average`?
**Resposta:** Média de processos em estado executável ou em espera ininterrupta (I/O) nos últimos 1, 5 e 15 minutos. Num servidor de N núcleos, load sustentado bem acima de N indica saturação. Atenção: load alto com CPU ociosa geralmente é espera de disco/rede.
**Para aprofundar:** Como distinguir se o load vem de CPU ou de I/O? (`top`: `%wa`; `iostat -x`.)

### P7. O que é um processo zumbi e como eliminá-lo?
**Resposta:** Processo que terminou, mas cujo pai não leu seu status de saída com `wait()` — fica como entrada na tabela de processos, ocupando só um slot. Não morre com sinal: é preciso matar ou corrigir o processo pai para ele "recolher" o filho; se o pai for o PID 1 e não recolher, só reiniciando o serviço ou o nó.
**Para aprofundar:** Como confirma? (`ps aux | grep 'Z'`, estado `Z+`.)

### P8. O que cgroups e namespaces trazem para os containers?
**Resposta:** Namespaces isolam o que o processo *enxerga* (PID, rede, montagens, usuários, hostname); cgroups limitam e medem o que ele *usa* (CPU, memória, I/O, pids). Juntos, são a base técnica de Docker/Kubernetes sem precisar de hipervisor.
**Para aprofundar:** E se um container não tiver limite de memória? (Pode disparar o OOM killer do nó e matar outros pods.)

---

## 🌐 Redes

### P9. Descreva o three-way handshake do TCP e o encerramento em quatro vias. Por que o encerramento precisa de um passo a mais?
**Resposta:** Abertura: SYN → SYN+ACK → ACK (3 passos sincronizam os números de sequência nas duas direções). Encerramento: FIN → ACK → FIN → ACK (4 passos) porque cada lado fecha sua direção de envio de forma independente — ao receber um FIN, um lado ainda pode ter dados para enviar antes de mandar seu próprio FIN.
**Para aprofundar:** O que é um SYN flood e como mitigar? (SYN cookies, `net.ipv4.tcp_syncookies`.)

### P10. O servidor acumula milhares de conexões em TIME_WAIT. É problema? Como trata?
**Resposta:** TIME_WAIT (tipicamente 60 s) existe para garantir a chegada dos últimos ACKs e evitar que pacotes antigos contaminem conexões novas. Milhares de TIME_WAIT num servidor com muito tráfego de saída (ex.: um proxy) podem esgotar as portas efêmeras. Mitigação: reusar conexões (keep-alive, pools), ajustar `tcp_tw_reuse` (seguro no lado cliente), nunca `tcp_tw_recycle` (removido do kernel por quebrar NAT).
**Para aprofundar:** TIME_WAIT no servidor com keep-alive é normal? (Sim, se o cliente fecha primeiro.)

### P11. Descreva o fluxo completo de uma resolução DNS.
**Resposta:** 1) Cache local (navegador/SO) → 2) resolvedor recursivo (ex.: 8.8.8.8) → 3) se não estiver no cache dele, consulta iterativa: servidor raiz → TLD (`.com`) → autoritativo do domínio → 4) o recursivo guarda no cache conforme o TTL e responde ao cliente. Cada nível pode ter cache com seu próprio TTL.
**Para aprofundar:** Como depura uma falha de DNS? (`dig +trace`, `nslookup`, conferir `/etc/resolv.conf`, descobrir se falhou o autoritativo ou o recursivo.)

### P12. Compare algoritmos de load balancing: round-robin, least-connections e consistent hashing.
**Resposta:** Round-robin distribui por turnos (simples, assume backends homogêneos); weighted round-robin pondera pela capacidade; least-connections manda para o backend com menos conexões ativas (melhor com sessões de duração variável); consistent hashing mapeia chaves para nós num anel, de modo que ao adicionar/remover um nó só uma fração das chaves muda de lugar (ideal para caches).
**Para aprofundar:** Balanceamento em camada 4 ou 7? (L4: rápido, por IP/porta; L7: roteia por URL/cabeçalho/cookie, permite sticky sessions e terminação TLS.)

### P13. O que significam 502, 503 e 504? Como distingui-los depurando?
**Resposta:** 502 Bad Gateway: o proxy recebeu resposta inválida do upstream (processo morto, cabeçalhos corrompidos). 503 Service Unavailable: o upstream responde que não pode atender (sobrecarregado, em deploy, circuito aberto). 504 Gateway Timeout: o upstream não respondeu a tempo. Chave: 502 = resposta ruim, 503 = "não posso", 504 = "não respondeu".
**Para aprofundar:** Onde olha primeiro? (Logs do proxy com `upstream_response_time` / `upstream_status` no Nginx.)

### P14. O `traceroute` mostra asteriscos (`* * *`) em vários saltos. Significa problema?
**Resposta:** Não necessariamente: muitos roteadores limitam ou ignoram os pacotes de sonda ICMP/UDP por política — não respondem, mas encaminham o tráfego normalmente. Só é problema se o destino final também não responder ou houver perda sustentada a partir de um salto específico.
**Para aprofundar:** Alternativas? (`mtr` combina ping + traceroute; olhar perda *acumulada* por salto, não pontual.)

### P15. O que é MTU e quais sintomas dá um MTU mal configurado?
**Resposta:** Maximum Transmission Unit: tamanho máximo de pacote num enlace (típico 1500 no Ethernet). Se um pacote grande não pode ser fragmentado (bit DF ativo) e não há Path MTU Discovery (ICMP bloqueado), a conexão "trava": o handshake funciona, mas a transferência de dados congela — sintoma clássico após túnel VPN/GRE mal configurado.
**Para aprofundar:** Como confirma? (`ping -M do -s 1472 <destino>` para testar o PMTU.)

---

## 🏢 Data center: energia, refrigeração e hardware

### P16. O que é PUE e qual valor é considerado bom?
**Resposta:** Power Usage Effectiveness = energia total do data center / energia que chega aos equipamentos de TI. PUE 1.0 seria perfeito; a indústria moderna fica entre 1.2 e 1.5; acima de 2.0 há muita margem de melhoria (refrigeração ineficiente, UPS superdimensionados).
**Para aprofundar:** O que baixa o PUE? (Confinamento de corredores, free cooling, elevar o setpoint conforme a ASHRAE, inversores nos ventiladores.)

### P17. Como é desenhada a energia de um rack crítico? (Dupla alimentação, UPS, gerador)
**Resposta:** Dupla alimentação elétrica (A e B) de fontes independentes, cada servidor com fonte dupla (uma em cada alimentação); UPS online que filtra e sustenta a carga em microcortes; geradores a diesel com partida automática e autonomia de combustível para cortes longos. Todo o caminho é redundante N+1 ou 2N conforme o Tier.
**Para aprofundar:** E se uma PDU falhar? (A outra alimentação assume 100% — por isso cada uma é dimensionada para a carga total.)

### P18. Explique o conceito de corredor frio / corredor quente.
**Resposta:** Os racks são orientados alternando a tomada de ar frio (frontal) e a exaustão de ar quente (traseira), formando corredores frios (insuflamento) e quentes (retorno). Evita que um servidor aspire o ar quente do vizinho, melhora a eficiência do CRAC/CRAH e permite elevar o setpoint geral.
**Para aprofundar:** Que temperatura/umidade a ASHRAE recomenda? (18–27 °C, umidade relativa 20–80%, ponto de orvalho controlado para evitar condensação e descargas.)

### P19. Compare RAID 5, RAID 6 e RAID 10. Quando usar cada um?
**Resposta:** RAID 5: paridade distribuída, tolera 1 disco com falha, bom aproveitamento de capacidade, mas rebuilds longos e arriscados em discos grandes. RAID 6: paridade dupla, tolera 2 falhas, mais seguro para discos de alta capacidade, penaliza escrita. RAID 10 (1+0): espelho + striping, tolera múltiplas falhas conforme a distribuição, melhor desempenho de leitura/escrita, mas só ~50% de capacidade útil.
**Para aprofundar:** Por que RAID 5 é desaconselhado com discos de 8 TB+? (Chance de uma segunda falha ou erro irrecuperável — URE — durante um rebuild de muitas horas.)

### P20. Um servidor reinicia sozinho de forma aleatória. Qual seu fluxo de diagnóstico de hardware?
**Resposta:** 1) Revisar logs do SO e do BMC/IPMI (`ipmitool sel list`) buscando erros de hardware; 2) conferir temperaturas e ventiladores; 3) teste de memória (ECC corrige 1 bit e registra; múltiplos erros corrigíveis = DIMM suspeito); 4) SMART dos discos; 5) fontes de alimentação e PDU; 6) isolar por substituição (mover a carga, trocar o DIMM de slot). Documentar cada passo.
**Para aprofundar:** O que significa um SEL cheio de "Memory ECC correctable"? (O DIMM está se degradando: planejar a troca antes que um erro incorrigível derrube o nó.)

### P21. O que você verificaria antes de aprovar um servidor recém-instalado no rack?
**Resposta:** Checklist: firmware/BIOS/BMC atualizados; RAID configurado e verificado; teste de memória e de estresse de CPU; rede (as duas alimentações, velocidade negociada, VLANs corretas); etiquetagem de cabos e registro no CMDB/DCIM; sensores de temperatura funcionando; teste das duas fontes desplugando uma.
**Para aprofundar:** Por que testar desplugando uma fonte? (É a única forma de provar que a redundância A/B é real, e não "as duas no mesmo circuito".)

---

## ☸️ Kubernetes

### P22. Qual a relação entre Pod, Deployment e Service?
**Resposta:** Pod é a menor unidade implantável (um ou mais containers compartilhando rede e armazenamento). Deployment gerencia um conjunto de Pods idênticos: define o template, o número de réplicas e a estratégia de atualização. Service dá um ponto de acesso estável (IP virtual + DNS) a esse conjunto mutável de Pods via seletores de labels.
**Para aprofundar:** E se você deletar na mão os Pods de um Deployment? (O ReplicaSet os recria: o estado desejado manda.)

### P23. Diferenças entre ClusterIP, NodePort e LoadBalancer.
**Resposta:** ClusterIP: IP virtual acessível só dentro do cluster (comunicação interna). NodePort: expõe o Service numa porta alta de cada nó (acesso externo rudimentar). LoadBalancer: provisiona um balanceador externo na nuvem com IP público (o modo padrão para expor serviços em cloud).
**Para aprofundar:** E o Ingress? (Um único ponto de entrada L7 que roteia por host/caminho para múltiplos Services ClusterIP.)

### P24. Como o scheduler decide em qual nó colocar um Pod?
**Resposta:** Duas fases: filtragem (descarta nós que não atendem: recursos insuficientes, taints sem tolerância, nodeSelector/affinity sem match, portas em uso) e pontuação (escolhe o melhor entre os candidatos por recursos livres, afinidade, dispersão). Depois o kubelet do nó executa.
**Para aprofundar:** Diferença entre nodeSelector, nodeAffinity e taints/tolerations? (nodeSelector: match simples; affinity: regras ricas com preferência/obrigatoriedade; taints: o nó *repele*, o Pod precisa *tolerar*.)

### P25. Um Pod está em CrashLoopBackOff. Qual sua rotina de diagnóstico com kubectl?
**Resposta:** 1) `kubectl describe pod` (eventos: OOMKilled? liveness falhando? imagem não encontrada?); 2) `kubectl logs --previous` (logs do container que morreu); 3) revisar probes e limites de recursos no manifesto; 4) `kubectl get events --sort-by=.lastTimestamp`; 5) se for a app: reproduzir o comando de start na mão.
**Para aprofundar:** O que significa exit code 137? (SIGKILL: quase sempre OOM ou `kubectl delete`/evicção. Exit 1 = erro da app.)

### P26. Diferença entre readinessProbe e livenessProbe? O que acontece se confundir?
**Resposta:** readiness diz "estou pronto para receber tráfego" (se falhar, o Pod sai dos endpoints do Service, mas continua vivo); liveness diz "continuo saudável" (se falhar, o kubelet reinicia o container). Se usar liveness onde cabia readiness, um pico de carga pode causar restarts em cascata que pioram a queda.
**Para aprofundar:** Por que não colocar a mesma checagem pesada nas duas? (Uma query lenta no banco como liveness transforma lentidão temporária em restarts.)

### P27. Como funciona o HPA (Horizontal Pod Autoscaler)?
**Resposta:** Um controlador que consulta periodicamente as métricas (CPU/memória por padrão, métricas custom com adaptador) e ajusta o número de réplicas do Deployment/ReplicaSet entre mínimo e máximo para manter o valor alvo. Atua com janelas de estabilização para evitar oscilação.
**Para aprofundar:** Por que o HPA às vezes "não reage" a um pico súbito? (Janela de medição + cooldown: foi feito para tendências, não para picos instantâneos — para isso, dimensione o mínimo com folga ou use KEDA com métricas externas.)

### P28. Como passar configuração e segredos para um Pod? O que nunca fazer com um Secret?
**Resposta:** ConfigMap para configuração não sensível e Secret para dados sensíveis, injetados como variáveis de ambiente ou montados como arquivos. Nunca: subir ao repo em claro, passar por linha de comando (fica no histórico) nem logar. Em produção, integrar com um gerenciador externo (Vault, KMS da nuvem) com rotação.
**Para aprofundar:** Secrets do Kubernetes são criptografados por padrão? (Não: só base64. É preciso ativar criptografia em repouso no etcd.)

---

## 🚨 On-call e resposta a incidentes

### P29. Como você define os níveis de severidade de um incidente (P1–P4)?
**Resposta:** P1/crítico: serviço fora do ar ou dados em risco, afeta muitos usuários → resposta imediata, acorda quem for preciso. P2/alto: degradação importante com workaround. P3/médio: impacto limitado, trata em horário comercial. P4/baixo: cosmético ou melhoria. A chave: critérios objetivos e escritos — não "o que parecer grave às 3 da manhã".
**Para aprofundar:** Quem pode declarar um P1? (Qualquer um que identificar os critérios: melhor um P1 falso do que um P1 tardio.)

### P30. O que um postmortem blameless deve conter?
**Resposta:** Cronologia objetiva, impacto medido (duração, usuários, SLO queimado), causa raiz técnica *e* sistêmica, o que funcionou bem na resposta e ações corretivas com dono e data. "Blameless": analisa-se o sistema e os processos, não as pessoas — se alguém tem medo de contar tudo, o postmortem não serve.
**Para aprofundar:** Como evitar que as ações morram na gaveta? (Cada ação vira um ticket com responsável e data de revisão na próxima reunião de postmortems.)

### P31. Explique SLI, SLO e SLA, e o que é "error budget".
**Resposta:** SLI (indicador): métrica medida, ex. "99,9% das requisições < 200 ms em 30 dias". SLO (objetivo): o valor com que você se compromete internamente. SLA (acordo): o compromisso contratual com penalidade. Error budget = 1 − SLO: a "cota de falhas" permitida; se esgota, congelam-se lançamentos e prioriza-se confiabilidade.
**Para aprofundar:** E quando o error budget queima na primeira semana do mês? (Moratória de deploys não críticos + foco nas top causas.)

### P32. Receberam seu turno de on-call. O que você revisa no handover?
**Resposta:** Incidentes abertos e seu estado, mudanças recentes em produção (deploys, migrações), alertas conhecidos/falsos positivos, trabalhos agendados (manutenções, backups) e "onde estão os runbooks". Um bom handover é escrito, não contado de boca na correria.
**Para aprofundar:** E se você herdar 200 alertas sem acknowledge? (Triagem: silenciar duplicados, priorizar por impacto no usuário e abrir a dívida de alertas no dia seguinte.)

### P33. No meio do incidente, o que vem primeiro: mitigar ou achar a causa raiz?
**Resposta:** Mitigar sempre primeiro: devolver o serviço aos usuários (rollback, failover, escalar capacidade, feature flag). A causa raiz se investiga com o serviço já estável. Caçar a causa com o serviço fora do ar prolonga o impacto e leva a mudanças arriscadas sob pressão.
**Para aprofundar:** Frase de referência: "stop the bleeding first". Quando declara mitigado? (SLI volta ao verde de forma sustentada — não um pico.)

---

## 📊 Monitoramento

### P34. Quais são os quatro tipos de métricas do Prometheus e quando usar cada um?
**Resposta:** Counter: só sobe (requisições totais, erros totais → para razões e taxas). Gauge: sobe e desce (conexões ativas, temperatura). Histogram: distribui observações em buckets (latências → percentis calculáveis). Summary: percentis pré-calculados no cliente (parecido, mas não agregável entre instâncias).
**Para aprofundar:** Por que um Counter que "desce" quebra seus alertas? (Reinícios do processo o zeram: use `rate()`/`increase()`, que tratam isso.)

### P35. Como desenhar uma regra de alerta que não gere fadiga?
**Resposta:** Alertar sobre sintomas que o usuário vê (taxa de erros, latência p95, saturação com previsão de impacto) — não sobre cada causa interna; limiares com janela de tempo (">1% de erros por 5 min", não um pico de 10 s); severidade condizente com a ação exigida; todo alerta com runbook linkado. Revisar periodicamente os alertas que ninguém atende e removê-los ou rebaixá-los.
**Para aprofundar:** O que é "alert fatigue" e seu risco? (Tantos alertas que todos são ignorados: no dia do P1 de verdade, ninguém olha o canal.)

### P36. O que medem os métodos RED e USE?
**Resposta:** RED (para serviços): Rate (req/s), Errors (taxa de falhas), Duration (latência). USE (para recursos): Utilization (% em uso), Saturation (fila/espera), Errors. RED diz "o serviço está sofrendo?"; USE diz "qual recurso está no limite?".
**Para aprofundar:** Qual usa para um disco? (USE: % cheio, latência de I/O — saturação —, erros SMART.)

### P37. Diferença entre monitoramento caixa-branca e caixa-preta?
**Resposta:** Caixa-branca: métricas/logs/traces de dentro do sistema (o que acontece por dentro). Caixa-preta: sondas externas que verificam o comportamento visível ao usuário (o site responde? o login funciona?). Você precisa dos dois: a caixa-preta diz *que* algo falha para o usuário; a branca diz *por quê*.
**Para aprofundar:** Exemplo de caixa-preta: (uma sonda que faz login completo a cada minuto de fora do data center.)

---

## 🔥 Cenários reais de troubleshooting

> Cadeia de diagnóstico passo a passo. Na entrevista, narre o que você *faria*, em ordem, e o que descarta em cada passo.

### E1. O serviço não sobe: "no space left on device", mas o `df` mostra espaço livre
**Cadeia:** 1) `df -i`: quase certo que são inodes esgotados (milhões de arquivinhos: sessões, caches, filas de e-mail). 2) Localizar: `find /var -xdev | head`, ou por diretório com `du --inodes`. 3) Mitigar: rodar/limpar o diretório culpado — nunca `rm -rf` no escuro. 4) Se não forem inodes: `lsof | grep deleted` (arquivo deletado ainda aberto) → truncar via `/proc/<pid>/fd`.
**Fechamento:** Ação definitiva: logrotate + monitoramento de inodes (não só de espaço).

### E2. De repente, milhares de conexões em TIME_WAIT e "cannot assign requested address"
**Cadeia:** 1) Confirmar: `ss -s`, `netstat -an | grep TIME_WAIT | wc -l`. 2) Identificar quem abre/fecha tanto (um cliente sem keep-alive ou sem pool de conexões). 3) Mitigar: ativar keep-alive e pools no cliente; `tcp_tw_reuse=1` se o esgotamento for de portas efêmeras no lado que *inicia* conexões. 4) Subir `ip_local_port_range` como paliativo.
**Fechamento:** A causa raiz quase sempre está no cliente, não no servidor.

### E3. O site fica lento só para alguns usuários: suspeita de DNS
**Cadeia:** 1) `dig` + `dig +trace` de uma máquina afetada vs uma saudável: onde os tempos divergem? 2) Comparar resolvedores (`/etc/resolv.conf`): usam o mesmo recursivo? 3) Olhar os TTLs: TTL muito baixo multiplica consultas; um registro envenenado/trocado demora a propagar. 4) Checar DNSSEC se houver validação falhando.
**Fechamento:** Meça antes e depois; o DNS é inocente até o `dig` provar o contrário.

### E4. Um processo morre toda noite no mesmo horário sem deixar logs: OOM
**Cadeia:** 1) `dmesg -T | grep -i "oom\|killed process"`: confirmar que o OOM killer agiu. 2) Correlacionar o horário com um cron (backup, batch, relatório). 3) Ver o pico no monitoramento histórico (memória do processo crescendo = vazamento; pico pontual = carga). 4) Mitigar: mover o batch para um nó com mais RAM ou limitá-lo; ajustar `oom_score_adj` só sabendo o que faz.
**Fechamento:** Se for vazamento lento, restart agendado é paliativo; o fix é no código.

### E5. A VM na nuvem está lenta embora "a CPU esteja em 30%": steal time
**Cadeia:** 1) `top`: olhar o `%st` (steal). Um `%st` alto e sustentado = o hipervisor não entrega a CPU pedida porque os vizinhos a monopolizam (noisy neighbor) ou o host está com overselling. 2) Confirmar com o provedor/métricas do hipervisor. 3) Mitigar: trocar para instância dedicada ou outro host/zona; no longo prazo, dimensionar com margem.
**Fechamento:** Não é sua culpa nem seu código: é contenção do host.

### E6. Um Pod entra em CrashLoopBackOff após um deploy
**Cadeia:** 1) `kubectl describe pod`: eventos (ImagePullBackOff? OOMKilled 137? probe falhando?). 2) `kubectl logs --previous`: a última mensagem antes de morrer geralmente entrega (porta em uso, variável faltando, migração que falhou). 3) Comparar com o manifesto anterior (`kubectl rollout history`). 4) Mitigar: `kubectl rollout undo` para estabilizar; investigar sem pressão.
**Fechamento:** Rollback primeiro, root cause depois (regra de ouro de incidentes).

### E7. O Nginx retorna 502 de forma intermitente
**Cadeia:** 1) Logs do Nginx com `upstream_status` e `upstream_response_time`: o upstream responde mal ou não responde? 2) Se 502 com tempo ~0: o backend morreu ou recusou a conexão (olhar os logs dele — OOM? restarts?). 3) Se intermitente sob carga: pool de conexões do backend esgotado ou keepalive mal configurado entre proxy e upstream. 4) Revisar `proxy_next_upstream` e health checks.
**Fechamento:** O 502 quase sempre é o backend, não o proxy: siga a cadeia rio acima.

### E8. Um disco do RAID pisca em âmbar: degradação do array
**Cadeia:** 1) Sem pânico: com RAID 6/10 o array continua funcionando. 2) Identificar o disco (`megacli`/`storcli` ou a controladora): falha real ou predictive failure pelo SMART? 3) Verificar se há hot-spare ou pedir a reposição JÁ. 4) Trocar a quente e vigiar o rebuild (é o momento de maior risco: uma segunda falha pode ser fatal no RAID 5).
**Fechamento:** Nunca reconstruir sem backup verificado se o array já está degradado.

### E9. À meia-noite tudo para de funcionar: certificado TLS expirado
**Cadeia:** 1) `echo | openssl s_client -connect host:443 -servername host | openssl x509 -noout -dates`: confirmar a expiração. 2) Mitigar: renovar/emergência com a CA (Let's Encrypt: `certbot renew --force-renewal`). 3) Descobrir por que a renovação automática falhou (cron caído, desafio HTTP bloqueado, rate limits).
**Fechamento:** Ação definitiva: monitoramento de expiração com alerta a 30/14/7 dias. Certificados não deveriam surpreender ninguém.

### E10. O banco está lento e a app esgota seu pool de conexões
**Cadeia:** 1) É o banco ou a app? `SHOW PROCESSLIST` / `pg_stat_activity`: muitas queries em `Waiting` ou uma única query monstra bloqueando? 2) Se for uma query: `EXPLAIN`, índices faltando, um deploy que mudou o plano de execução. 3) Mitigar: matar a query bloqueante, escalar réplicas de leitura ou ativar o circuit breaker na app. 4) Revisar o tamanho do pool: um pool maior não conserta um banco lento — só esconde o problema.
**Fechamento:** Pool esgotado é sintoma, não causa: siga a cadeia até a query.

---

*Foi útil? Deixe uma ⭐ e compartilhe com quem está se preparando para entrevistas de SRE/DevOps.*
