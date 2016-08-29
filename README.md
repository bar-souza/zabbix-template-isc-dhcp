Tutorial criado por WERNECK COSTA e extraído do site https://neckcosta.wordpress.com/2013/11/01/monitoramento-de-pools-isc-dhcp-com-zabbix/ em 29/08/2016
Template Zabbix criado por Bruno Souza e adaptado ao tutorial

O Zabbix não possui nativamente uma forma de monitoramento para Servidores DHCP, então qual a solução? Resposta: fazer com que o Zabbix leia as informações de algum lugar no sistema operacional.

Cenário:

Servidor Zabbix Versão 3.0;
Servidor a ser monitorado: Debian 8 Jessie, zabbix-agent 3.0.4 ;
Aplicação alvo do monitoramento: ISC-DHCP (apt-get install dhcp-server);
O usuário que roda o zabbix_agent precisa ter um shell válido (/bin/bash).
No meu caso, além da funcionalidade padrão (servir configurações de rede de forma automática), meu DHCP server possui a função de servir múltiplas faixas diferentes que chegam nele através de DHCP-Relay feitos em alguns Switchs.

Mas, o que é importante monitorar em um Servidor deste tipo?

Capacidade de IPs por pool DHCP;
Quantidade de IPs sendo utilizados em cada pool;
Quantidade de IPs “Tocados” em cada pool.
Estes IPs  “Tocados” representa a quantidade máxima de Máquinas diferentes que já foram endereçadas em cada pool. Servem, em linhas gerais, para ver o quão suficiente é este pool (ou seja, agora você tem como planejar!).

Agora, como fazer pra que o Zabbix leia estas informações no servidor DHCP?

Pesquisando uma forma de extrair os dados do DHCP Server, me deparei com esta alternativa.  O problema desta abordagem é que, para cada novo pool configurado, seria necessário, manualmente, fazer uma nova configuração individual no SNMP… Ou seja, nada prático. Continuando a pesquisa, achei a seguinte alternativa: http://dhcpd-pools.sourceforge.net/ A promessa era de ser mais rápido que os “concorrentes” (inclusive do que minha primeira alternativa), pois o arquivo de Leases é bem grande e qualquer “cat + grep + cut” levaria um bom tempo e consumiria recursos da máquina. Bem, só dá pra saber, testando:

Passos:

Instalação das dependências:

apt-get install -y -b  xz-utils gnulib uthash-dev
Criação de um diretório para os fontes:

mkdir /usr/src/dhcpd-pools
Download e descompactação dos fonte do dhcpd-pools:

cd /usr/src/dhcpd-pools

wget http://sourceforge.net/projects/dhcpd-pools/files/dhcpd-pools-2.28.tar.xz/download -O dhcpd-pools-2.28.tar.xz

tar xJf dhcpd-pools-2.28.tar.xz
Compilação e instalação:

./configure && make && make install
Depois de instalado, um teste básico pode ser feito com o seguinte comando:

dhcpd-pools -c local/do/dhcpd.conf -l local/do/dhcpd.leases
No caso do setup utilizado nesta instalação, o comando completo ficaria (procure os arquivos, conforme sua instalação):

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases
A saída do comando, provavelmente será parecida com esta:

Ranges:
shared net name first ip last ip max cur percent touch t+c t+c perc
All networks 172.16.101.1 - 172.16.101.254 254 6 2.362 179 185 72.835

Shared networks:
name max cur percent touch t+c t+c perc

Sum of all ranges:
name max cur percent touch t+c t+c perc
All networks 254 6 2.362 179 185 72.835
Para cada instalação e configuração, a saída poderá ser diferente. Mas é possível ter uma noção através deste arquivo. Lendo o retorno do comando, temos o seguinte:

shared net name first ip last ip max cur percent touch t+c t+c perc
All networks 172.16.101.1 - 172.16.101.254 254 6 2.362 179 185 72.835
Dá pra tirar daí, que minha rede possui uma capacidade máxima de 254 endereços (da forma que foi configurada), está com 6 endereços em uso atualmente, o que representa 2.362% e que dela já foram utilizados 179 endereços diferentes (além dos outros campos que não comentarei). Porém se eu tiver várias redes preciso que, ao invés de ser mostrado “All networks”, seja mostrado o nome da rede. Para isso, será necessário executar alterações nos arquivos de configuração do Serviço DHCP. As alterações são pra “envolver” cada pool com um nome específico, através do uso da diretiva “shared-network”. Em minha instalação, editei o arquivo /etc/dhcp/dhcpd.conf que estava assim:

subnet 172.16.100.0 netmask 255.255.253.0 {
 default-lease-time 7200;
 max-lease-time 10800;
 range 172.16.101.1 172.16.101.254;
 option routers 172.16.100.15;
 option domain-name-servers 172.16.100.15;
 option broadcast-address 172.16.101.255;
 option domain-name "dominio.local";
}
Deixando-o asim:

shared-network Minha-rede {
subnet 172.16.100.0 netmask 255.255.253.0 {
 default-lease-time 7200;
 max-lease-time 10800;
 range 172.16.101.1 172.16.101.254;
 option routers 172.16.100.15;
 option domain-name-servers 172.16.100.15;
 option broadcast-address 172.16.101.255;
 option domain-name "dominio.local";
}
}
Depois de salvar e reiniciar o serviço DHCPD, execute novamente o seguinte comando:

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases
Veja que em “Shared networks”, agora o retorno é diferente:

Ranges:
shared net name first ip last ip max cur percent touch t+c t+c perc
All networks 172.16.101.1 - 172.16.101.254 254 6 2.362 179 185 72.835

Shared networks:
name max cur percent touch t+c t+c perc
Minha-rede 254 11 4.331 178 189 74.409

Sum of all ranges:
name max cur percent touch t+c t+c perc
All networks 254 6 2.362 179 185 72.835
O dhcpd-pools oferece um help, no estilo man page do Linux, e nele é importante observar o seguinte (man dhcpd-pools):

-L, --limit=NR
The NR will limit what will be printed. Syntax is similar to chmod(1) permission string. The NR limit string uses two digits which vary between
0 to 7. The first digit determines which headers to display, and the second digit determines which numeric analysis tables to include in the output.
The following values are "OR'd" together to create the desired output. The default is 77.

01 Print ranges
02 Print shared networks
04 Print total summary
10 Print range header
20 Print shared network header
40 Print total summary header
Aqui é possível ver que a utilização do parâmetro “-L”, limita a exibição conforme a tabela à cima, de acordo com a combinação escolhida. Então, se executarmos o comando incluindo o “-L 22”, ele retornará somente os dados das “Shared networks”:

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22
Resultado:

Shared networks:
name max cur percent touch t+c t+c perc
Minha-rede 254 11 4.331 178 189 74.409
Você também poderia fazer algum malabarismo para separar o texto, pegar certa linha e etc mas, se o comando já fornece isto, para que reinventar a roda?

Fazendo o zabbix capturar os dados

Sabemos que para obter dados que o Zabbix não consegue nativamente, uma das possibilidades é a utilização de “UserParameters”. Estes, em resumo, são itens criados livremente pelo Administrador para que, quando consultados via Zabbix_agent, retornem algo útil, como o resultado de um comando, por exemplo. Para que isto funcione, é necessário editar o arquivo de configuração do agente. Nele, procure a diretiva “Include” que, provavelmente estará assim:

# Include=/usr/local/etc/zabbix_agentd.userparams.conf
Esta diretiva diz que, além do arquivo .conf padrão, o zabbix (quando iniciar) irá procurar outros arquivos. Nesta caso é possível indicar um arquivo direto (como à cima), ou uma pasta que conterá vários outros arquivos. Para nosso tutorial, ficou da seguinte forma (Claro que este diretório precisa existir e estar acessível para leitura.):

Include=/usr/local/etc/zabbix_agentd.conf.d/
Neste diretório será necessário criar o arquivo “UserParameters_dhcpd.conf”, contendo os itens customizados e seus respectivos comandos. Copie e cole o conteúdo à baixo no arquivo “UserParameters_dhcpd.conf” (antes de copiar, verifique a configuração dos parâmetros “-c” e “-l“):

UserParameter=dhcp.pool.all,dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22

UserParameter=dhcp.pool.max[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f2

UserParameter=dhcp.pool.use[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f3

UserParameter=dhcp.pool.percent[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f4

UserParameter=dhcp.pool.touch[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f5
Depois de criar o arquivo (sem os espaço entre as linhas), salve e reinicie o zabbix_agent. Cada linha de UserParameter possui dois elementos: o item e o comando. Exemplo:

Item

dhcp.pool.all
Comando

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22
O item é o que será configurado na interface do Servidor Zabbix. Quando o agente receber a solicitação do item (por exemplo, dhcp.pool.all), ele retornará os dados resultantes do comando. Depois disso, no servidor zabbix execute o seguinte comando:

zabbix_get -s ip.do.servidor.dhcp -k dhcp.pool.all
Se tudo estiver correto, um retorno como este será exibido (caso exista mais de um pool configurado):

Shared networks:
name max cur percent touch t+c t+c perc
Minha-rede01 356 260 73.034 87 347 97.472
Minha-rede02 422 0 0.000 0 0 0.000
Minha-rede03 782 539 68.926 232 771 98.593
...
Minha-redeN 250 43 17.200 206 249 99.600
Lembre-se: Para cada pool (Minha-rede01, Minha-rede02, Minha-rede03), serão cadastrados todos os parâmetros (max, use, percent e touch).

Outros testes pode ser feitos, utilizando os outros itens no UserParameter:

dhcp.pool.max[Minha-rede01]
Retorno (exemplo): 200

dhcp.pool.use[Minha-rede01]
Retorno (exemplo): 20
dhcp.pool.percent[Minha-rede01]
Retorno (exemplo): 12.4
dhcp.pool.touch[Minha-rede01]
Retorno (exemplo): 23
Pronto, parâmetros configurados! Agora é hora de ir ao zabbix server (via interface web) e cadastrar os itens para o Servidor DHCP, como no exemplo à baixo:

Conf_itens_DHCP01

No exemplo dos pools dado anteriormente, é possível ver o tipo de dado retornado por cada item. Cadastre um por um, e vá vendo em “Lastest data” se o resultado aparece. Depois, configure os gráficos a seu gosto e pronto, você tem os dados que queria!??

Até este ponto, é possível monitorar os pools DHCP tranquilamente. Agora, se você já trabalhou no zabbix com LLD (Low Level Discovery) e sabe que seria mais interessante pegar os pools automaticamente, siga pro próximo parágrafo.

Utilizando LLD para buscar os pools DHCP:

Este LLD precisa de um Script rodando localmente para trazer os nomes dos pools e popular os 4 itens. Para isso, escolhi o diretório /usr/local/etc/zabbix/scripts/ para adicionar o script que retornará os dados no formato JSON. Lá, criei o script dhcppolls.sh com o seguinte conteúdo:

#!/bin/bash
pools_all=`dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L02|egrep -io "^[a-z]*(\-|\_)?[a-z]*"`
pools_qtd=`echo $pools_all|wc -w`
retorno=`echo -e "{\n\t\t"\"data\"":["`
for p in $pools_all
do
 if [ "$pools_qtd" -le "1" ]
 then
 retorno=$retorno`echo -e "\n\t\t{\"{#DHCPPOOL}\":\"$p\"}"`
 else
 retorno=$retorno`echo -e "\n\t\t{\"{#DHCPPOOL}\":\"$p\"},"`
 fi
 pools_qtd=$(($pools_qtd - 1))
done
retorno=$retorno`echo -e "]}"`
echo -e $retorno
Salve e saia do arquivo. No arquivo UserParameters_dhcpd.conf adicione o seguinte “UserParameter”:

UserParameter=dhcp.pool.discovery,/usr/local/etc/zabbix/scripts/dhcppolls.sh

Reinicie o agente zabbix no servidor DHCP.

Para teste, execute o seguinte no servidor Zabbix:

zabbix_get -s ip.do.servidor.dhcp -k dhcp.pool.discovery
Os dados serão parecidos com estes:

{ "data":[ {"{#DHCPPOOL}":"Minha-rede01"}, {"{#DHCPPOOL}":"Minha-rede02"}, {"{#DHCPPOOL}":"Minha-rede03"}]}
Se tudo correr como descrito, estaremos a um passo de concluir a configuração. Se não, volte ao começo do tutorial e revise as configurações feitas.

Agora, na interface web, vá ao Host Servidor DHCP e cadastre um novo item do Tipo “Discovery rules”, da seguinte forma:

LLD_DHCP

Veja que o parâmetro em “Macro” é o mesmo retornado na execução de “dhcp.pool.discovery”. Agora, será preciso criar os itens a serem populados com o dado trazido pelo LLD. Para isso, depois de criar a “Discovery Rule”, clique em “Item prototype” e crie o item como segue:

LLD_DHCP_item

Crie os 4 itens, conforme vimos anteriormente e, se desejar Triggers, gráficos e o que mais for necessário.

Pronto, fim do tutorial!??

Se precisar de algum suporte, é possível me encontrar (e vários outros membros) na lista Brasileira de discussão sobre Zabbix, através deste link.

Abraço!