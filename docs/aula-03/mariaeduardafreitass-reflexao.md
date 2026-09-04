# Aula 03 - Reflexão individual

## 1. Ambiente
Kernel observado: 6.8.0-1052-azure
Memória disponível: 5.7Gi (de um total de 7.8Gi)
Uma informação que me chamou atenção: o ambiente é um container Linux (Ubuntu 24.04) rodando sobre Azure, e não uma máquina física — mesmo assim ele expõe informações reais de kernel, memória e processos como se fosse um sistema completo.

## 2. Processo
PID observado: 15880
PPID observado: 4136
Explique com suas palavras a diferença entre programa e processo: programa é o código parado, guardado em disco, como uma receita. Processo é uma instância desse programa em execução, com identidade própria (PID), ligada a um processo pai (PPID) e com um estado (STAT) controlado pelo Sistema Operacional enquanto ela existe na memória.

## 3. Proteção
Quem negou a leitura do arquivo e por quê? O kernel negou. Ao rodar `chmod 000` no arquivo, retirei todas as permissões (leitura, escrita e execução) para qualquer usuário, incluindo o dono. Quando tentei ler com `cat`, o processo pediu acesso ao arquivo e o kernel verificou as permissões antes de liberar os dados — como não havia nenhuma permissão concedida, ele recusou o pedido com "Permission denied". Isso mostra que a proteção de arquivos não é decidida pelo disco, e sim mediada pelo Sistema Operacional a cada tentativa de acesso.

## 4. Projeto da squad
Componente escolhido: Agendamento de consultas

Esse componente rodaria como quê? Como um serviço (processo em execução contínua), do tipo API, que fica esperando requisições de pacientes e da equipe da clínica para marcar, remarcar ou cancelar consultas.

Se ele falhar, qual impacto de negócio aparece? Pacientes não conseguiriam marcar ou confirmar consultas, a recepção ficaria sem visibilidade da agenda em tempo real, e poderia gerar conflitos de horário (dois pacientes marcados no mesmo horário) ou perda de atendimentos, afetando diretamente a receita e a confiança dos pacientes na clínica.

Qual controle deveria existir? Um healthcheck monitorando se o serviço está no ar, com restart automático em caso de queda, além de logs de cada operação de agendamento (pra rastrear falhas) e permissões de acesso controladas, já que dados de agendamento envolvem informações sensíveis dos pacientes.