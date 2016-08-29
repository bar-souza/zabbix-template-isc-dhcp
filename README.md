Tutorial criado por WERNECK COSTA e extra�do do site https://neckcosta.wordpress.com/2013/11/01/monitoramento-de-pools-isc-dhcp-com-zabbix/ em 29/08/2016
Template Zabbix criado por Bruno Souza e adaptado ao tutorial

O Zabbix n�o possui nativamente uma forma de monitoramento para Servidores DHCP, ent�o qual a solu��o? Resposta: fazer com que o Zabbix leia as informa��es de algum lugar no sistema operacional.

Cen�rio:

Servidor Zabbix Vers�o 3.0;
Servidor a ser monitorado: Debian 8 Jessie, zabbix-agent 3.0.4 ;
Aplica��o alvo do monitoramento: ISC-DHCP (apt-get install dhcp-server);
O usu�rio que roda o zabbix_agent precisa ter um shell v�lido (/bin/bash).
No meu caso, al�m da funcionalidade padr�o (servir configura��es de rede de forma autom�tica), meu DHCP server possui a fun��o de servir m�ltiplas faixas diferentes que chegam nele atrav�s de DHCP-Relay feitos em alguns Switchs.

Mas, o que � importante monitorar em um Servidor deste tipo?

Capacidade de IPs por pool DHCP;
Quantidade de IPs sendo utilizados em cada pool;
Quantidade de IPs �Tocados� em cada pool.
Estes IPs  �Tocados� representa a quantidade m�xima de M�quinas diferentes que j� foram endere�adas em cada pool. Servem, em linhas gerais, para ver o qu�o suficiente � este pool (ou seja, agora voc� tem como planejar!).

Agora, como fazer pra que o Zabbix leia estas informa��es no servidor DHCP?

Pesquisando uma forma de extrair os dados do DHCP Server, me deparei com esta alternativa.  O problema desta abordagem � que, para cada novo pool configurado, seria necess�rio, manualmente, fazer uma nova configura��o individual no SNMP� Ou seja, nada pr�tico. Continuando a pesquisa, achei a seguinte alternativa: http://dhcpd-pools.sourceforge.net/ A promessa era de ser mais r�pido que os �concorrentes� (inclusive do que minha primeira alternativa), pois o arquivo de Leases � bem grande e qualquer �cat + grep + cut� levaria um bom tempo e consumiria recursos da m�quina. Bem, s� d� pra saber, testando:

Passos:

Instala��o das depend�ncias:

apt-get install -y -b  xz-utils gnulib uthash-dev
Cria��o de um diret�rio para os fontes:

mkdir /usr/src/dhcpd-pools
Download e descompacta��o dos fonte do dhcpd-pools:

cd /usr/src/dhcpd-pools

wget http://sourceforge.net/projects/dhcpd-pools/files/dhcpd-pools-2.28.tar.xz/download -O dhcpd-pools-2.28.tar.xz

tar xJf dhcpd-pools-2.28.tar.xz
Compila��o e instala��o:

./configure && make && make install
Depois de instalado, um teste b�sico pode ser feito com o seguinte comando:

dhcpd-pools -c local/do/dhcpd.conf -l local/do/dhcpd.leases
No caso do setup utilizado nesta instala��o, o comando completo ficaria (procure os arquivos, conforme sua instala��o):

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases
A sa�da do comando, provavelmente ser� parecida com esta:

Ranges:
shared net name first ip last ip max cur percent touch t+c t+c perc
All networks 172.16.101.1 - 172.16.101.254 254 6 2.362 179 185 72.835

Shared networks:
name max cur percent touch t+c t+c perc

Sum of all ranges:
name max cur percent touch t+c t+c perc
All networks 254 6 2.362 179 185 72.835
Para cada instala��o e configura��o, a sa�da poder� ser diferente. Mas � poss�vel ter uma no��o atrav�s deste arquivo. Lendo o retorno do comando, temos o seguinte:

shared net name first ip last ip max cur percent touch t+c t+c perc
All networks 172.16.101.1 - 172.16.101.254 254 6 2.362 179 185 72.835
D� pra tirar da�, que minha rede possui uma capacidade m�xima de 254 endere�os (da forma que foi configurada), est� com 6 endere�os em uso atualmente, o que representa 2.362% e que dela j� foram utilizados 179 endere�os diferentes (al�m dos outros campos que n�o comentarei). Por�m se eu tiver v�rias redes preciso que, ao inv�s de ser mostrado �All networks�, seja mostrado o nome da rede. Para isso, ser� necess�rio executar altera��es nos arquivos de configura��o do Servi�o DHCP. As altera��es s�o pra �envolver� cada pool com um nome espec�fico, atrav�s do uso da diretiva �shared-network�. Em minha instala��o, editei o arquivo /etc/dhcp/dhcpd.conf que estava assim:

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
Depois de salvar e reiniciar o servi�o DHCPD, execute novamente o seguinte comando:

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases
Veja que em �Shared networks�, agora o retorno � diferente:

Ranges:
shared net name first ip last ip max cur percent touch t+c t+c perc
All networks 172.16.101.1 - 172.16.101.254 254 6 2.362 179 185 72.835

Shared networks:
name max cur percent touch t+c t+c perc
Minha-rede 254 11 4.331 178 189 74.409

Sum of all ranges:
name max cur percent touch t+c t+c perc
All networks 254 6 2.362 179 185 72.835
O dhcpd-pools oferece um help, no estilo man page do Linux, e nele � importante observar o seguinte (man dhcpd-pools):

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
Aqui � poss�vel ver que a utiliza��o do par�metro �-L�, limita a exibi��o conforme a tabela � cima, de acordo com a combina��o escolhida. Ent�o, se executarmos o comando incluindo o �-L 22�, ele retornar� somente os dados das �Shared networks�:

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22
Resultado:

Shared networks:
name max cur percent touch t+c t+c perc
Minha-rede 254 11 4.331 178 189 74.409
Voc� tamb�m poderia fazer algum malabarismo para separar o texto, pegar certa linha e etc mas, se o comando j� fornece isto, para que reinventar a roda?

Fazendo o zabbix capturar os dados

Sabemos que para obter dados que o Zabbix n�o consegue nativamente, uma das possibilidades � a utiliza��o de �UserParameters�. Estes, em resumo, s�o itens criados livremente pelo Administrador para que, quando consultados via Zabbix_agent, retornem algo �til, como o resultado de um comando, por exemplo. Para que isto funcione, � necess�rio editar o arquivo de configura��o do agente. Nele, procure a diretiva �Include� que, provavelmente estar� assim:

# Include=/usr/local/etc/zabbix_agentd.userparams.conf
Esta diretiva diz que, al�m do arquivo .conf padr�o, o zabbix (quando iniciar) ir� procurar outros arquivos. Nesta caso � poss�vel indicar um arquivo direto (como � cima), ou uma pasta que conter� v�rios outros arquivos. Para nosso tutorial, ficou da seguinte forma (Claro que este diret�rio precisa existir e estar acess�vel para leitura.):

Include=/usr/local/etc/zabbix_agentd.conf.d/
Neste diret�rio ser� necess�rio criar o arquivo �UserParameters_dhcpd.conf�, contendo os itens customizados e seus respectivos comandos. Copie e cole o conte�do � baixo no arquivo �UserParameters_dhcpd.conf� (antes de copiar, verifique a configura��o dos par�metros �-c� e �-l�):

UserParameter=dhcp.pool.all,dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22

UserParameter=dhcp.pool.max[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f2

UserParameter=dhcp.pool.use[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f3

UserParameter=dhcp.pool.percent[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f4

UserParameter=dhcp.pool.touch[*],dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22|grep -i $1|sed 's/ \+/;/g'|cut -d';' -f5
Depois de criar o arquivo (sem os espa�o entre as linhas), salve e reinicie o zabbix_agent. Cada linha de UserParameter possui dois elementos: o item e o comando. Exemplo:

Item

dhcp.pool.all
Comando

dhcpd-pools -c /etc/dhcp/dhcpd.conf -l /var/lib/dhcp/dhcpd.leases -L22
O item � o que ser� configurado na interface do Servidor Zabbix. Quando o agente receber a solicita��o do item (por exemplo, dhcp.pool.all), ele retornar� os dados resultantes do comando. Depois disso, no servidor zabbix execute o seguinte comando:

zabbix_get -s ip.do.servidor.dhcp -k dhcp.pool.all
Se tudo estiver correto, um retorno como este ser� exibido (caso exista mais de um pool configurado):

Shared networks:
name max cur percent touch t+c t+c perc
Minha-rede01 356 260 73.034 87 347 97.472
Minha-rede02 422 0 0.000 0 0 0.000
Minha-rede03 782 539 68.926 232 771 98.593
...
Minha-redeN 250 43 17.200 206 249 99.600
Lembre-se: Para cada pool (Minha-rede01, Minha-rede02, Minha-rede03), ser�o cadastrados todos os par�metros (max, use, percent e touch).

Outros testes pode ser feitos, utilizando os outros itens no UserParameter:

dhcp.pool.max[Minha-rede01]
Retorno (exemplo): 200

dhcp.pool.use[Minha-rede01]
Retorno (exemplo): 20
dhcp.pool.percent[Minha-rede01]
Retorno (exemplo): 12.4
dhcp.pool.touch[Minha-rede01]
Retorno (exemplo): 23
Pronto, par�metros configurados! Agora � hora de ir ao zabbix server (via interface web) e cadastrar os itens para o Servidor DHCP, como no exemplo � baixo:

Conf_itens_DHCP01

No exemplo dos pools dado anteriormente, � poss�vel ver o tipo de dado retornado por cada item. Cadastre um por um, e v� vendo em �Lastest data� se o resultado aparece. Depois, configure os gr�ficos a seu gosto e pronto, voc� tem os dados que queria!??

At� este ponto, � poss�vel monitorar os pools DHCP tranquilamente. Agora, se voc� j� trabalhou no zabbix com LLD (Low Level Discovery) e sabe que seria mais interessante pegar os pools automaticamente, siga pro pr�ximo par�grafo.

Utilizando LLD para buscar os pools DHCP:

Este LLD precisa de um Script rodando localmente para trazer os nomes dos pools e popular os 4 itens. Para isso, escolhi o diret�rio /usr/local/etc/zabbix/scripts/ para adicionar o script que retornar� os dados no formato JSON. L�, criei o script dhcppolls.sh com o seguinte conte�do:

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
Salve e saia do arquivo. No arquivo UserParameters_dhcpd.conf adicione o seguinte �UserParameter�:

UserParameter=dhcp.pool.discovery,/usr/local/etc/zabbix/scripts/dhcppolls.sh

Reinicie o agente zabbix no servidor DHCP.

Para teste, execute o seguinte no servidor Zabbix:

zabbix_get -s ip.do.servidor.dhcp -k dhcp.pool.discovery
Os dados ser�o parecidos com estes:

{ "data":[ {"{#DHCPPOOL}":"Minha-rede01"}, {"{#DHCPPOOL}":"Minha-rede02"}, {"{#DHCPPOOL}":"Minha-rede03"}]}
Se tudo correr como descrito, estaremos a um passo de concluir a configura��o. Se n�o, volte ao come�o do tutorial e revise as configura��es feitas.

Agora, na interface web, v� ao Host Servidor DHCP e cadastre um novo item do Tipo �Discovery rules�, da seguinte forma:

LLD_DHCP

Veja que o par�metro em �Macro� � o mesmo retornado na execu��o de �dhcp.pool.discovery�. Agora, ser� preciso criar os itens a serem populados com o dado trazido pelo LLD. Para isso, depois de criar a �Discovery Rule�, clique em �Item prototype� e crie o item como segue:

LLD_DHCP_item

Crie os 4 itens, conforme vimos anteriormente e, se desejar Triggers, gr�ficos e o que mais for necess�rio.

Pronto, fim do tutorial!??

Se precisar de algum suporte, � poss�vel me encontrar (e v�rios outros membros) na lista Brasileira de discuss�o sobre Zabbix, atrav�s deste link.

Abra�o!