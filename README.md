# Automatização de Backup utilizando Chaves Publicas, Cron e Rsync
Neste tutorial vamos aprender a como fazer uma automatização de backup entre duas maquinas, denomidas de:
- Maquina A - (Servidor de Backup);
- Maquina B - (Servidor de Arquivos);

# Falando um pouco do que vamos usar
Chaves publicas: Criptografia de chave pública, também conhecida como criptografia assimétrica, é uma classe de protocolos de criptografia baseados em algoritmos que requerem duas chaves, uma delas sendo secreta (ou privada) e a outra delas sendo pública. Isso permiti com que você possa configurar duas máquinas para que elas se confiem entre si, sem ter a necessidade de digitar senhas a um acesso via ssh ou scp.
Cron: é um agendador de tarefas do linux, no qual com ele você agendar tarefas diárias, ou a cada uma hora para que você não precise se preocupar com isso. O roda automaticamente toda vez que o sistema do linux reinicia. O arquivo do cron geralmente fica dentro da pasta /etc. Mas a frente vamos aprender a como configurar ele.
Rsync: rsync é um utilitário amplamente usado para manter cópias de um arquivo em dois sistemas de computadores ao mesmo tempo. É normalmente encontrado em sistemas do tipo Unix e em funções como um programa de sincronização de arquivos e transferência de arquivos. O algoritmo rsync, um tipo de codificação delta, é usado para minimizar o uso da rede.

# Vamos ao que interessa
Primeiramente precisamos fazer com as as duas máquinas (Maquina-A e Maquina-B) se confiem, e para isso vamos utilizar as chaves públicas.
No meu caso eu precisei fazer com que a Maquina-A confiasse na Maquina-B para poder receber os arquivos que eu estou enviando para fazer os backups. Então vamos lá.
Na Maquina-B (Servidor de Arquivos) vamos gerar uma chave privada dessa máquina. 

	$ ssh-keygen -t rsa

Aperte enter em todos os passos até aparecer “The key fingerprint : (chavegerada)”.
A chave privada está em:

    $ /home/server/.ssh/id_rsa.pub
    
>Lembrete: O nome de usuario da minha máquina é "server".

Para que seja possível uma conexão SSH sem senha entre Maquina-B e Maquina-A, é preciso divulgar para à Maquina-A a chave pública gerada pela Maquina-B. 
É possível fazer esta cópia remotamente executando os seguintes comandos: 

    $ scp /home/server/.ssh/id_rsa.pub usuario@ipdoservidor:/home/usuario
Nesse caso ainda irá aparecer para vocês digitar a senha para ter acesso a Maquina-A
então digite a senha do seu servidor.
O próximo passo vamos fazer um acesso via SSH na Máquina-A.
    
    ssh usuario@ipdoservidor
    
Novamente digite a senha da Maquina-A.
Agora vamos fazer a cópia do arquivo "id_rsa.pub" da Maquina-B dentro da Máquina-A.

    cat /home/server/.ssh/id_rsa.pub >> /home/server/.ssh/authorized_keys 
    
> /home/server/.ssh/id_rsa.pub é referente a minha Maquina-B.
E /home/server/.ssh/authorized_keys é referente a minha Maquina-A.

Agora para que funcione precisamos reinicializar o nosso httpd da Maquina-A.

    ssh root@servidor '/etc/init.d/httpd restart' 
    
Feito isso, pronto, a sua Maquina-A já confia na Maquina-B. 
Para fazer o teste, na sua Maquina-B digite:
    
    ssh usuario@ipdoservidor

Você vai ver que ele não pede mas senha para acessar o servidor, caso isso não ocorra, reveja todos os passos novamente e refaça o procedimento, se der tudo certo siga para o próximo passo.
# Baixando o Rsync
Caso você não tenha o rsync instalado em seu servidor, é possível fazer a instalação pelo codigo abaixo:
    
    $ sudo apt-get install rsync
Depois de instalado siga para a configuração do Cron.

# Configurando o Cron para fazer automatização
Para fazer os backups precisamos criar a pasta que irá receber todos os arquivos do da nossa Maquina-B (Servidor de Arquivos). 
Na Maquina-A faça vamos criar essa nova pasta dentro da pasta /var/backup. No meu caso utilizei o nome "BkpRemoto"
	
	$ mkdir /var/backups/BkpRemoto
	
Depois de criada a pasta, iremos precisar escrever arquivos dentro dessa pasta, para isso precisamos dá permissão a pasta:

	$ sudo chmod -R 777 /var/backups/BkpRemoto

Feito isso, vamos voltar para nossa Maquina-B (Servidor de Arquivos), agora iremos criar um script chamado "executar.sh". Dentro desse script vamos colocar o que o Rsync vai fazer e qual pasta ele vai mandar para a nossa Maquina-A.
Na Maquina-B eu coloquei o script na pasta principal do meu usuario.

    $ cd /home/server/
    Comparecendo na pasta principal vamos criar o arquivo executar.sh
    $ >> executar.sh

Depois de ter criado o arquivo, precisamos editar ele, eu utilizei o nano por ser um editor bem simples de se utilizar, mas podem ser utilizado muitos outros como o vi e kedit.
Então digitamos

    nano executar.sh
    
No editor, eu digitei os comandos do rsync seguindo as pastas no qual eu estou mandando para fazer os backups na Maquina-A. 

    $ rsync -Cravzp /home/server/bkpteste/  usuario@ipdoservidor:/var/backups/BkpRemoto

>Neste tutorial foi criado a pasta bkpteste dentro da pasta do meu usuario /home/server.

Terminado de digitar, agora temos que transformar esse arquivo "executar.sh" em um arquivo executável, para isso precisamos digitar.
	
	chmod +x /home/server/executar.sh
Agora chegou a vez do Cron, vamos configurar ele digite: 

    $ crontab -e 
    
Irá abrir o próprio editor nele que é baseado no nano o mesmo que utilizamos para fazer o script. A sintaxe do cron é uma sintaxe bem simples, sempre uma linha de rociocínio de mm hh dd mm ss comando onde:
- mm = minuto(0-59)
- hh = hora(0-23)
- dd = dia(1-31)
- MM = mes(1-12)
- ss = dia_da_semana(0-6)
- comando = comando a ser executado.

Eu editei no meu para que o cron executasse todos os dias as 22 horas. você vai editar de acordo com os seus gostos, a linha abaixo:

    00 22 * * * /home/server/executar.sh

> 00 = É os minutos da hora.
22 = Hora do dia em
*= Dias (0-30). o asterisco servar para marcar todos. no caso aqui todos os dias.
*= Meses, marquei como todos os meses.
*= Dias da semana, 0 para domingo e 6 para sabado, preferi marcar todos os dias.
/home/server/executar.sh = aqui eu mandei ele executar o meu script executar.sh que contem os dados do rsync.

Com isso vai fazer com que você torne suas automatizações mais simples e fácil.
Era um pouco do que eu tinha para repassar a vocês, e qualquer duvidas mandar email para lclpedro@gmail.com assim que eu visualizar irei responder, grande abraço, e até a próxima.


Criado por: Pedro Lucas C Lima
Email: lclpedro@gmail.com
Twitter: @pedrolucasliima
Facebook: fb.com/PedroLucasLiima

>Frase:
"Se não queres que ninguém saiba, não o faças."
Provérbio Chinês
