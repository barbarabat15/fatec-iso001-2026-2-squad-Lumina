# Aula 03 - Fundamentos e Arquitetura Operacional

## Baseline do Ambiente
1. Kernel: Linux 6.8.0-1052-azure
   Distribuição: Ubuntu 24.04.4 LTS (Noble Numbat)
2. Memória RAM disponível: 5.7 GiB disponíveis de um total de 7.8 GiB
   Espaço em disco disponível: 20 GB livres de 32 GB (34% usado)
3. Processos que chamaram atenção: bash, docker-init (processos padrão do ambiente)
4. Recurso mais crítico: armazenamento e disponibilidade, pois o sistema lida com prontuários e exames de pacientes que não podem ser perdidos, e o agendamento precisa estar sempre disponível para não impactar o atendimento.

## Processo e /proc
5. Componente que executa como processo/serviço: API de agendamento de consultas
6. Se esse processo morrer: pacientes não conseguem marcar, remarcar ou cancelar consultas
7. Decisão: o processo deve reiniciar automaticamente e gerar alerta em caso de falha

## Permissões e Syscalls
8. Quem negou o acesso: o kernel, que valida as permissões antes de liberar a leitura do arquivo
9. Operação esperada no trace: uma chamada openat ao tentar abrir o arquivo
10. Por que a aplicação não pode ignorar a permissão: o controle de acesso é feito pelo sistema operacional, não pela aplicação — ela depende do kernel autorizar
11. Arquivo/diretório de risco equivalente: prontuários e exames de pacientes, que só podem ser acessados por usuários autorizados

## Mapeamento para Sistemas Operacionais

| Componente | Executa como | Recurso/abstração do SO | Risco operacional | Controle/Evidência |
|---|---|---|---|---|
| API de agendamento | processo/serviço | CPU, memória, socket, logs | queda impede marcação de consulta | health check, restart automático |
| Banco de prontuários | processo + arquivos | memória, filesystem, I/O | perda ou vazamento de dados sensíveis | backup, permissões restritas |
| Storage de exames/laudos | arquivos/diretórios | filesystem, permissões | acesso indevido a dados de paciente | permissões, restore testado |
| Serviço de notificação | processo/job | CPU, memória, logs | lembrete não enviado, paciente falta | retry, logs de envio |