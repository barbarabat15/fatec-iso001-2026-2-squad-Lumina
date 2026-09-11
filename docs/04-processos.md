# Aula 04 - Modelo de Processos da Solução
## Tema: Clínica ou Consultório Digital

## 1. Process Inventory

| Componente | Executa como | Iniciado por | Perfil | Recurso crítico | Se morrer... | Controle |
|---|---|---|---|---|---|---|
| API de Agendamento | serviço/processo | runtime/systemd/container | I/O-bound | rede/banco | pacientes não conseguem marcar/remarcar consulta | healthcheck + restart + logs |
| Serviço de Documentos e Anexos (prontuários, exames) | serviço/processo | runtime/systemd/container | I/O-bound | filesystem/disco | anexos não carregam nem ficam visíveis para o médico | backup + recuperação + auditoria |
| Banco de Dados (dados de pacientes) | serviço | serviço gerenciado/container | I/O-bound | disco/memória | transações falham, risco de perda de dados sensíveis | backup + monitoramento + restart |
| Serviço de Autenticação e Permissões | processo/serviço | runtime/systemd/container | misto (CPU/I/O) | CPU/memória | usuários não conseguem logar ou dados ficam expostos indevidamente | logs de auditoria + restart + alertas |
| Worker de Notificações (lembretes de consulta) | processo/job | scheduler/fila | misto (CPU/I/O) | rede | pacientes não recebem lembrete e faltam à consulta | retry + timeout + logs |

## 2. Ciclo de vida

**API de Agendamento**
- Inicia: junto com o deploy da aplicação (systemd/container).
- Saudável: responde ao endpoint de healthcheck (ex.: `/health`) em poucos milissegundos.
- Encerra normalmente: recebe sinal de shutdown, finaliza requisições em andamento e fecha conexões.
- Detecção de falha: healthcheck falhando ou erro 5xx recorrente nos logs.
- Reinício: orquestrador (systemd/container) reinicia automaticamente; se falhar 3x seguidas, alerta o time.
- Logs/métricas: latência por requisição, taxa de erro, agendamentos criados/cancelados por minuto.

**Serviço de Documentos e Anexos**
- Inicia: junto com a API, como processo separado por questão de isolamento.
- Saudável: consegue gravar e ler um arquivo de teste no storage.
- Encerra normalmente: termina uploads em andamento antes de desligar.
- Detecção de falha: erro ao gravar (disco cheio, permissão negada) ou timeout de upload.
- Reinício: automático, mas com alerta se o disco estiver cheio (reiniciar não resolve).
- Logs/métricas: uploads com sucesso/falha, espaço em disco disponível, tempo de resposta.

**Banco de Dados**
- Inicia: antes da API, como dependência.
- Saudável: aceita conexões e responde a uma query simples de teste.
- Encerra normalmente: aguarda transações abertas finalizarem (graceful shutdown).
- Detecção de falha: conexões recusadas ou queries travando.
- Reinício: automático via orquestrador + verificação de integridade dos dados após subir.
- Logs/métricas: conexões ativas, queries lentas, uso de disco/memória.

**Serviço de Autenticação e Permissões**
- Inicia: junto com a API.
- Saudável: emite e valida tokens corretamente no teste de healthcheck.
- Encerra normalmente: invalida sessões temporárias de forma controlada.
- Detecção de falha: falha ao validar token válido ou lentidão anormal no login.
- Reinício: automático, com log de auditoria detalhado (por lidar com dados sensíveis de acesso).
- Logs/métricas: tentativas de login (sucesso/falha), acessos negados, tempo de resposta.

**Worker de Notificações**
- Inicia: por agendamento (scheduler) ou disparado por evento (nova consulta marcada).
- Saudável: consegue consumir a fila e enviar mensagens de teste.
- Encerra normalmente: termina o processamento do item atual da fila antes de parar.
- Detecção de falha: mensagens acumulando na fila sem serem processadas.
- Reinício: automático com retry; após N tentativas, mensagem vai para fila de erro (dead-letter).
- Logs/métricas: notificações enviadas/falhadas, tamanho da fila, tempo médio de envio.

## 3. Hipótese de falha

Componente escolhido: **Serviço de Documentos e Anexos**

1. **Sintoma para o usuário:** o médico não consegue visualizar um exame anexado pelo paciente; o paciente recebe erro ao tentar anexar um novo documento.
2. **Evidência no SO/aplicação:** erro 500 no endpoint de upload; logs mostram falha de escrita em disco (ex.: `ENOSPC` - disco cheio, ou `EACCES` - permissão negada).
3. **Ação de recuperação:** verificar espaço em disco disponível; checar permissões da pasta de storage; reiniciar o serviço; se necessário, restaurar arquivos corrompidos a partir do backup.
4. **Risco de reiniciar incorretamente:** reiniciar sem resolver a causa raiz (por exemplo, disco cheio) faz o serviço cair de novo logo em seguida, gerando um loop de reinícios; também há risco de perder uploads que estavam em andamento no momento do restart.

## 4. Decisões da Sprint

- **Decisão 1:** o Serviço de Documentos terá controle de permissões próprio, separado do banco de dados principal, para reforçar a privacidade de dados sensíveis (prontuários, exames), alinhado à LGPD.
- **Decisão 2:** as notificações serão assíncronas via worker/fila, para não travar a API de agendamento enquanto o lembrete é enviado.
- **Dívida técnica:** rotina de auditoria de acesso (registrar quem acessou qual prontuário e quando) ainda não está implementada — entra no backlog da Sprint 3.