# lab-cibersantander-DIO
Repositório feito para a conclusão do desafio DIO " Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux"

O primeiro passo do Desafio é a configuração de duas máquinas virtuais, uma maquina com metasploitable2 para servir de alvo e uma máquina com o kali linux para realizar o ataque

baixei o virtualbox para emular as duas máquinas através do site:  vmware.com/productes/desktop-hypervisor/workstation-and-fusion
depois baixei a máquina virtual do metasploitable2 do site: sourceforge.net/projectes/metasplitable/
e o kali linux do site: https://www.kali.org/get-kali/#kali-virtual-machines  (baixada a opção vmware)

após ter as máquinas baixadas e a vmware workstation baixada e instalada usei o comando Ctrl+O para roda a maquina virtual clicando no arquivo referente a máquina virtual (identificado pela extenção .vmx)
para configurar ambas as máquinas clicar com o botão direito do mouse no nome da máquina e ir em configurações, clicar onde diz network adapter e na aba network conection setar a opção Host-only para ambas as máquinas.

feito as configuraçoes inicializamos as máquinas por padrao o login e senha do kali linux é kali:kali e da máquina metasploitable é msfadmin:msfadmin.
passar um comando (sem as aspas)
"ip a" 
na máquina metasploitable2 para pegar o ip do alvo.
192.168.134.128 (no meu caso)

confirmar que as máquinas estão se comunicando com o comando (sem as aspas):
"ping -c3 192.168.134.128"
o comando ping serve para enviar pacotes para o alvo para saber se ele está recebendo esses pacotes
o comando -c3 serve para passar a quantidade de pacotes que serão enviados. Se estiver funcionando o comando retornará algum valor diferente de 0 em received (vai depender da quantidade de pacotes enviados).


ATAQUE BRUTE FORCE EM FTP

antes de começar o ataque de brute force precisamos descobrir quais os serviços estão rodando na máquina alvo, a versão desse serviço e a porta.
Para isso utilizamos o Nmap. O Nmap serve para fazer essa enumeraçao de serviços, portas e até exploraçao.

O comando usado no Nmap para enumeração de portas e versão foi o:
(sem aspas)
"nmap -v -sV 192.168.134.128 >varredura.txt"

onde:
-v significa verbose que é para visualizar as informações enquanto a aplicação está rodando.
-sV significa service version
>varredura.txt é para pegar o resultado da enumeraçao e salvar num arquivo chamado varredura.txt

feito isso encontramos 2 serviços FTP rodando nas portas 21(vsftpd 2.3.4) e 2121(ProFTPD 1.3.1)
atacamos a porta 21 usando uma lista de usuarios e outra de senhas mais comuns utilizadas em serviços FTP
com as listas criadas usamos o serviço do MEDUSA para realizar o ataque de brute force com o seguinte comando:
(sem aspas)
"medusa -h 192.168.134.128 -U login.txt -P senha.txt -M ftp >bruteforce.txt"
onde:
-h é para host
-U para o arquivo que contem os usuarios
-P para o aquivo que contém as senhas
-M é o serviço que vai ser atacado
>bruteforce.txt para salvar o resultado do bruteforce num arquivo chamado bruteforce.txt

Para facilitar a leitura de saída visto que a lista de logins ficou um pouco grande utilizei um comando linux para filtrar por "ACCOUNT FOUND"
comando linux:
grep "ACCOUNT FOUND" bruteforce.txt
vemos que trouxe vários resultados
optamos pelo login que possuia o nome msfadmin por se tratar de uma credencial que faz referencia a administrador, então podemos ver se temos acesso total.
para confirmar se realmente funcionou vamos testar no próprio serviço FTP
com o comando:
(sem aspas)
ftp 192.168.134.128
(por padrao o ftp se nao passar a porta ele vai para a porta 21, se por acaso tiver em outra porta usar o comando -P mais o numero da porta correspodente)
com o serviço conectado ele vai pedir por login e logo em seguida pela senha, usando as credenciais obtidas retorna a seguinte mensagem:
230 Login sucessful.
Como o intuito do desafio era apenas a técnica de bruteforce nao seguimos com a exploraçao do serviço.

No arquvo mitigaçao-bruteforce estão as recomendações para mitigação desse tipo de ataque.


ATAQUE AUTOMAÇAO DE TENTATIVAS EM FORMULÁRIO WEB (DVWA)

A segunda etapa do desafio consistia em realizar um ataque de automaçao de tentativas em formulário web em DVWA
para acessar o site usamos a seguinte url:
http://192.168.134.128/dvwa/login.php
(a url vai variar de acordo com o ip)
o site remete a uma tela de login simples com username e password e um botão login
ao fazer uma requisiçao inválida: teste:teste
a aplicação retorna a seguinte mensagem: Login failed

Para atacar esse tipo de serviço utilizei a ferramenta Burpsuit Community

utilizando a aba proxy com o intercept on
interceptei a chamada de uma tentativa de login e enviei essa intercepção para o intruder para poder realizar o ataque
selecionamos a query que passamos como login e usando o botao "Add§" a query fica assim:
username=§teste§&password=§teste§&Login=Login
entre os "§" é o que o burp vai substituir para realizar o ataque
usando a opção de ataque cluster bomb passamos uma lista em cada posiçao entre os "§"
alternamos entre as posiçoes na aba "payloads" no seguimento "payload position". No primeiro passamos a lista de usuarios e no segundo passamos a lista de senhas podemos importar as listas e usar a mesma que usamos no
bruteforce de FTP.

Na aba settings no seguimento "grep-match" vamos limpar a lista no botao clear e adicionar a frase que a aplicação web retornou quando fizemos a tentativa de login inválida
(sem aspas)
"Login failed"
Ainda na aba settings no seguimento "redirections" passamos o follow redirections para opção "always"

feito isso basta iniciar o ataque
o burp irá fazer as tentativas de login com a lista que foi passada, testando todas as senhas para cada login e vai apresentar o filtro "Login failed", sempre que um login falhar ele vai trazer o número "1"
nesse filtro, quando o login for válido ficará vazio. podemos ver também que o tamanho da "length" se altera, indicando que temos um resultado positivo. feito isso só basta logar e confirmar as credenciais.

o arquivo mitigaçao-automaçao-formulario-web contém informações para mitigação desse ataque.


ATAQUE PASSWORD SPRAYING EM PROTOCOLO SMB COM ENUMERAÇAO DE USUÁRIOS

A ultima parte do desafio pede uma enumeraçao de seriço smb com um ataque password spraying

para realizar a enumeração em smb usamos a ferramenta enum4linux
com o comando:
enum4linux 192.168.134.128 >enumsmb.txt
salvamos em um arquivo para facilitar a captura dos nomes de usuários com o comando linux
comando:
grep "user" enumsmb.txt
obtemos mais informações além dos nomes de usuários, infos como o workgroup e até nomes de diretórios
para pegar apenas o nome dos usuarios e salvar numa lista para realizar o ataque, vamos utilizar o comando:
grep -oP 'user:\[\K[^\]]+' enumsmb.txt > usuarios.txt

-o = mostra apenas a parte que casa com o padrão
-P = usa expressões regulares Perl (mais poderosas)
'user:\[\K[^\]]+' = padrão que busca:
user:\[ = encontra "user:["
\K = "keep" - descarta tudo que foi casado até agora
[^\]]+ = captura tudo que não seja "]" (o nome do usuário)

resumindo: o comando limpa tudo que vem antes/inclusive de "[" e depois/inclusive de "]"

agora pegamos uma lista de senhas e rodamos no MEDUSA
como o comando:
medusa -h 192.168.134.128 -U usuarios.txt -P senha.txt -M smbnt  -t 3 | grep "ACCOUNT FOUND"

-h host
-U lista de usuarios
-P lista de senhas
-M serviço atacado
-t 3 é a quantidade de threads em simultâneo
usamos o grep para filtrar a saída por account found

para verificar se as credenciais são realmente válidas utilizamos o serviço smbclient com o comando:
smbclient -L //192.168.134.128 -U msfadmin

-L serve para listar 
para passar o ip alvo usamos o //
-U é usuário
vai pedir a senha, após digitarmos o serviço vai mostrar os diretórios e o workgroup.
como o pedido do desafio era realizar um password spraying não demos seguimento na exploração.

no arquivo mitigaçao ataque-enumeraçao-spraying-smb estão as recomendações para mitigação dessa vulnerabilidade.
