Análise de split-brain — Incidente C



1\. Identificação



\- Autor(es): Leiliane Flávia Travasso Oliveira; José Vinícius Cunha da Silva

\- Cenário: C — Split-brain após ataque de rede.

\- Caminho usado: Docker (docker compose), PostgreSQL 16, primário na porta 5442 e réplica na porta 5443, comandos executados pelo PowerShell.



2\. Resumo do incidente



Simulamos uma situação em que o primário fica inacessível, mas continua funcionando e aceitando gravações. Promovemos a réplica para assumir o papel de novo primário, porém o antigo também continuou aceitando escritas. Com isso, os dois bancos passaram a trabalhar de forma independente, causando split-brain. Inserimos dados diferentes nos dois lados, comparamos os resultados e, por fim, desligamos o antigo primário para manter apenas um banco principal ativo.



3\. Linha do tempo



Horário         | Ação                                                        | Resultado

\----------------|-------------------------------------------------------------|--------------------------------------------------------------

Antes de 13:20  | Verificação da replicação com pg\_stat\_replication           | Réplica conectada, estado streaming, modo async.

13:20:01        | Início da medição do RTO                                    | Horário inicial registrado no PowerShell.

13:20:02        | SELECT pg\_promote(); na réplica                             | Retornou t, confirmando a promoção.

Após a promoção | INSERT de 10 registros no antigo primário                   | INSERT 0 10.

Após a promoção | INSERT de 7 registros no novo primário                      | Foram inseridos 7 registros.

Durante a comparação | Consulta max(id) e count(\*) nos dois bancos            | Antigo: 10 e 10. Novo: 7 e 7.

Antes do desligamento | Consulta de amostras com ts > T\_PROMOCAO              | Foram encontradas versões diferentes dos registros nos dois bancos.

Após a comparação | docker compose stop primary                               | Antigo primário desligado.

13:22:43        | Verificação do novo primário                                | pg\_is\_in\_recovery() = false e count(\*) = 7.



4\. Medições



Métrica                         | Onde medi       | Comando ou método                                  | Valor

\--------------------------------|-----------------|----------------------------------------------------|-------------------------------

Tempo da promoção               | Réplica         | SELECT pg\_promote();                               | 0,78 segundo

RTO do procedimento             | PowerShell      | Diferença entre $fim e $inicio                     | 161,98 segundos

Registros no antigo primário    | Primário antigo | SELECT max(id), count(\*) FROM eventos;             | 10 registros

Registros no novo primário      | Novo primário   | SELECT max(id), count(\*) FROM eventos;             | 7 registros

Divergência entre versões       | Dois bancos     | Consulta dos registros após T\_PROMOCAO             | 7 versões conflitantes

Registros exclusivos do antigo  | Primário antigo | Comparação dos IDs                                 | 3 registros (IDs 8, 9 e 10)



RTO: o tempo total medido foi de 00:02:41.9774207, aproximadamente 2 minutos e 42 segundos. A medição começou às 13:20:01.313 e terminou às 13:22:43.291. Esse intervalo inclui a promoção, as gravações, a comparação dos dados, o desligamento do antigo primário e a verificação final. Portanto, representa o tempo do procedimento completo no laboratório, e não apenas o tempo de promoção.



RPO: foram identificadas 7 versões conflitantes dos registros com IDs de 1 a 7, pois os horários (ts) eram diferentes em cada banco. Além disso, os IDs 8, 9 e 10 existiam somente no antigo primário. Se a ramificação antiga fosse descartada sem reconciliação, essas diferenças seriam perdidas. Não foi possível medir um RPO em segundos, pois não registramos o instante de uma falha real nem o momento de cada gravação em relação a ela.



Failover: a promoção foi realizada com SELECT pg\_promote();, que retornou t. O tempo entre o início da chamada e seu retorno foi de aproximadamente 0,78 segundo. Depois, o antigo primário foi desligado e confirmamos que o novo continuava ativo.



5\. Causa raiz



A causa foi a promoção da réplica sem garantir que o antigo primário estivesse desligado ou isolado. Mesmo estando inacessível na situação simulada, o antigo banco continuou funcionando e aceitando gravações. Como a replicação era assíncrona, os dois bancos passaram a receber escritas independentes, causando divergência nos dados.



6\. Correção aplicada e prova



\- Correção: registramos as diferenças entre os bancos e desligamos o antigo primário com docker compose stop primary.

\- Antes: o antigo primário possuía 10 registros e o novo primário possuía 7. Os IDs de 1 a 7 apresentavam horários diferentes, e os IDs 8 a 10 existiam somente no antigo.

\- Depois: executamos SELECT pg\_is\_in\_recovery(); no novo primário e o resultado foi false, confirmando que ele continuava funcionando como primário.

\- Verificação final: SELECT max(id), count(\*) FROM eventos; retornou 7 | 7, mostrando que os registros do novo primário continuaram disponíveis após o desligamento do antigo.



Como regra de reconciliação, os dados do antigo primário devem ser preservados antes de qualquer descarte. Os registros conflitantes precisam ser analisados e comparados conforme uma regra definida pela aplicação. Não é seguro simplesmente substituir os dados de um banco pelos do outro.



7\. Comparação com serviço gerenciado



\- Em serviços gerenciados, recursos como Multi-AZ ajudam a manter a disponibilidade do banco por meio de instâncias em zonas diferentes e de um processo controlado de failover.

\- Antes de promover outra instância, o serviço precisa garantir que o antigo primário não continue aceitando gravações. Isso pode ser feito por mecanismos de isolamento, conhecidos como fencing.

\- O PITR (Point-in-Time Recovery) permite recuperar o banco para um momento específico utilizando backups e registros de transações. Porém, ele não substitui o isolamento do antigo primário e, sozinho, não impede split-brain.



8\. Prevenção e lições



\- Nunca promover uma réplica apenas porque o primário está inacessível. É necessário confirmar que o antigo primário foi desligado ou isolado.

\- Utilizar mecanismos de fencing para impedir que dois bancos atuem como primários ao mesmo tempo.

\- Utilizar quorum para que as decisões de promoção dependam da maioria dos nós.

\- Registrar e comparar os dados divergentes antes de descartar qualquer ramificação.

\- Lição: dois bancos funcionando não significam que os dados estejam consistentes. É necessário garantir que exista apenas um primário autorizado a receber gravações.



9\. Respostas às perguntas



1\. O que é fencing e por que um serviço gerenciado faz isso antes de promover? Fencing é o isolamento do antigo primário, impedindo que ele continue recebendo gravações. Isso garante que apenas um banco possa atuar como principal e evita que dois bancos aceitem escritas ao mesmo tempo.



2\. Por que um número ímpar de nós (quorum) evita split-brain? Um número ímpar facilita a formação de uma maioria. Por exemplo, em um ambiente com três nós, são necessários dois votos para formar quorum. Assim, apenas o lado que conseguir a maioria poderá tomar decisões, evitando que duas partes da rede se considerem principais ao mesmo tempo.



