# compass
Instalando e configurando um servidor NFS (Network File System) para subir o Apache e validar se o serviço está online ou offline a cada 5 minutos no Amazon Linux 2 utilizando a AWS e o Windows PowerShell - passo a passo

Criando e configurando tudo necessário para rodar a instância:

Toda essa parte é realizada no console AWS.

Criar o Ec2:

Para criar a instância Ec2, basta pesquisar pelo serviço Ec2, executar instância, configurar de acordo com o necessário, dê um nome (foi utlizado Amazon Linux 2 com a família t3.small e um volume de 16 GB SSD) escolha seu par de chave, criar novo grupo de segurança e permita o tráfego no seu IP.

Criar VPC, sub-rede e gateway da internet:

Pesquise por VPC e crie o seu VPC e vincule ele a sua instância Ec2, depois crie um gateway de internet vinculado a VPC e depois crie uma sub-rede vinculado a VPC.

Criar um Elastic IP:

Gere 1 elastic IP e anexe à instância EC2.

Criar tabela de rotas e editar grupo de segurança:

Crie uma tabela de rotas e vincule a VPC, edite as rotas e adicione uma com destino 0.0.0.0/0 e o alvo no seu gateway de internet (a rota com alvo local permanece inalterada)
Em grupos de segurança, selecione o grupo de segurança utilizado pela sua Ec2 e adicione as seguintes rotas de entrada:
Tipo de tráfego: SSH (22/TCP)
Tipo de tráfego: Custom TCP (111/TCP)
Tipo de tráfego: Custom UDP (111/UDP)
Tipo de tráfego: Custom TCP (2049/TCP)
Tipo de tráfego: Custom UDP (2049/UDP)
Tipo de tráfego: HTTP (80/TCP)
Tipo de tráfego: HTTPS (443/TCP)
Certifique-se de definir a fonte como "Anywhere" (Qualquer lugar) ou "0.0.0.0/0" para permitir o acesso público.

Após a criação, configuração e execução da Instância Ec2, VPC, sub-rede, gateways de internet, IP elástico e grupos de segurança, é hora de conectar a Instância, vamos utilizar o modo Cliente SSH com o Windows PowerShell.

Lembrar de estar com a instância Ec2 no estado Running (Executando).
Sempre é uma boa prática interromper a execução da instância quando não estiver utilizando!

Configurando no Linux

A partir daqui, foi utilizado o Windows PowerShell como cliente SSH

Conecte-se à Instância Ec2:

Abra o Windows PowerShell em modo de administrador e execute o seguinte comando para se conectar à instância:
ssh -i “/caminho/para/sua-chave-privada.pem” ec2-user@seu-endereco-ip-publico
Ex: ssh -i "C:\Users\zeca\Downloads\minhachave.pem" ec2-user@77.77.777.0

Atualize o sistema:   

Para atualizar os repositórios de pacotes, execute o seguinte comando:
sudo yum update -y
   
Instale o Servidor NFS:

Para instalar o servidor NFS, é necessário o pacote `nfs-utils`. Execute o seguinte comando para instalar o pacote:
sudo yum install nfs-utils -y
   
Habilitar e Iniciar o Serviço NFS:

Para habilitar o serviço NFS, execute o seguinte comando:
sudo systemctl enable nfs-server
  
   Para iniciar o serviço NFS, execute o seguinte comando:
sudo systemctl start nfs-server

É importante verificar se o NFS está ativo, execute o seguinte comando:
sudo systemctl status nfs-server
   
Configurar as Exportações NFS:
A próxima etapa envolve a definição das configurações NFS, que especificam quais pastas ou sistemas de arquivos serão disponibilizados via NFS. Para realizar essa configuração, é necessário editar o arquivo denominado /etc/exports. Vamos utilizar o editor de texto Nano para fazer as alterações no arquivo, execute o seguinte comando:
sudo nano /etc/exports

É possível adicionar a linha a seguir ao arquivo /etc/exports para disponibilizar o diretório /var/nfs_share com permissões de leitura e escrita para todos os clientes NFS. No entanto, é importante observar que essa configuração não é recomendada para ambientes de produção, execute o seguinte comando:
/var/nfs_share *(rw,sync,no_root_squash)

Configure as exportações do NFS no arquivo /etc/exports. Adicione uma linha para compartilhar o diretório que você deseja:
/caminho/do/seu/diretorio *(rw,sync,no_root_squash)
Após adicionar a linha, pressione CTRL+O e ENTER para salvar e depois CTRL+X para sair do arquivo.

Criar um diretório dentro do filesystem do NFS com seu nome:

Use o comando mkdir para criar um diretório dentro do filesystem do NFS, execute o seguinte comando:
sudo mkdir /caminho/do/seu/diretorio/seu_nome


Habilitar o Serviço Portmapper:

O serviço Portmapper é necessário para o funcionamento do NFS. Certifique-se de que ele esteja habilitado e em execução com os seguintes comandos:
sudo systemctl enable rpcbind
sudo systemctl start rpcbin

Aplicar as Configurações de Exportação:

Para aplicar as novas configurações de exportação execute o seguinte comando:
sudo exportfs -a
 

Instalando o servidor web Apache no Amazon Linux 2

Atualize novamente o Sistema:

É importante manter os pacotes do sistema atualizados, então execute o comando:
sudo yum update -y
  
Instale o Apache 2:

Execute o seguinte comando para iniciar a instalação:
sudo yum install httpd -y

Inicie o Serviço Apache:

Após a instalação bem-sucedida, inicie o serviço Apache2 com o seguinte comando:
sudo systemctl start httpd
 
Habilite o Apache para Inicialização Automática:

Para assegurar que o Apache seja automaticamente inicializado toda vez que a instância for reiniciada, execute o comando a seguir:
sudo systemctl enable httpd 

Verifique o Apache:

Para verificar o status do Apache no servidor, execute o seguinte comando:
sudo systemctl status httpd
Certifique-se de que o status seja "ativo (running)". 

Criar um script de validação para o serviço Apache:

Agora vamos criar um script de validação para verificar o funcionamento do serviço Apache e enviar o resultado para um diretório NFS. Crie um diretório onde você deseja armazenar o script e os relatórios com o seguinte comando:
sudo mkdir -p /caminho/do/seu/diretorio/nfs

Crie um arquivo de script, por exemplo, `check_apache.sh`, dentro desse diretório utilize um editor nano para criar o arquivo “.sh” por exemplo:
sudo nano check_apache.sh

Adicione o seguinte código ao script `check_apache.sh` criado:

#!/bin/bash
# Diretório onde os relatórios serão salvos
relatorio_dir="/caminho/do/seu/diretorio/nfs"
# Nome do serviço
servico="Apache"
# Verifica se o Apache está ativo
if systemctl is-active --quiet httpd; then
    status="Online"
    arquivo_saida="$relatorio_dir/servico_online.txt"
else
    status="Offline"
    arquivo_saida="$relatorio_dir/servico_offline.txt"
fi
# Obtém a data e hora atual
data_hora=$(date '+%Y-%m-%d %H:%M:%S')
# Cria a mensagem completa
mensagem_completa="$data_hora - $servico - Status: $status"
# Escreve a mensagem no arquivo de saída
echo "$mensagem_completa" >> "$arquivo_saida"


 
   Certifique-se de substituir `/caminho/do/seu/diretorio/nfs` pelo caminho real do diretório NFS onde você deseja armazenar os relatórios, como por exemplo: /home/ec2-user/nome/nfs

Torne o script executável:

Para tornar o script executável, execute o seguinte comando:
sudo chmod +x /caminho/do/seu/diretorio/nfs/check_apache.sh
   
Dê permissão aos documentos criados para bash/script ter acesso e edição continua do documento criado, execute os seguintes comandos:
sudo chmod 777 servico_offline.txt
sudo chmod 777 servico_online.txt

Preparar a execução automatizada do script a cada 5 minutos:

Para abrir o cron, execute o seguinte comando:
sudo crontab -e
   
Adicionar execução no cron

Adicione a seguinte linha ao seu arquivo crontab para executar o script a cada 5 minutos:
*/5 * * * * /bin/sh /home/check_apache.sh
Certifique-se de substituir `/home/` pelo seu verdadeiro caminho  onde está localizado o seu script `check_apache.sh`
Para digitar no arquivo cron, pressione a tecla A, após terminar a edição, pressione ESC e depois dê : e digite wq e pressione ENTER


Agora, o script denominado check_apache.sh será executado de forma automatizada a cada intervalo de 5 minutos. Ele realizará uma verificação do status do serviço Apache e registrará o resultado em dois arquivos: servico_online.txt e servico_offline.txt, localizados no diretório NFS definido anteriormente. Antes de executar o script em um ambiente de produção, é essencial garantir que tanto o serviço Apache quanto o diretório NFS estejam configurados de maneira adequada.

Para conferir se o script está funcionando, dentro do diretório onde estão localizados os arquivos ‘servico_online.txt’ e ‘servico_offline.txt’, execute o seguinte comando:
sudo cat servico_online.txt // mesmo caso para o offline
É esperado que mostre DATA HORA + nome do serviço + Status + mensagem! Se no offline não mostrar nada, significa que o serviço não ficou a nenhum momento offline!
Para checar se o serviço está sendo verificado a cada 5 minutos, espere e execute novamente o comando após o intervalo ter passado!

Mudando a TimeZone

Para mudar o fuso horário da Instância Amazon Linux 2, execute o seguinte comando:
sudo timedatectl set-timezone America/Fortaleza
O local você coloca de acordo com a zona que você quer, meu caso foi a de Fortaleza
