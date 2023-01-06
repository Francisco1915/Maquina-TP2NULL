# Maquina-TP2NULL
Teste de Penetração à máquina fornecida pelo professor da unidade curricular Testes de Penetração e Hacking Ético

# Máquina 2 – TP2NULL

## Enumeração

### Enumeração dos serviços:

Nmap -A 192.168.57.4

![image](https://user-images.githubusercontent.com/83584144/211105283-bc0c311a-f80d-4fc7-9853-f84a1efb0ff1.png)

A partir deste scan ficamos a saber que temos 3 portas abertas nesta máquina, porta 80 a alojar um serviço de Apche httpd, porta 111 com o rpcbind e por fim a porta 777 com o ssh.

## Analise da máquina e Mapeamento de vulnerabilidades:

### Web servers:

#### Porta 80:

![image](https://user-images.githubusercontent.com/83584144/211105314-ca67e6e8-b51f-4484-8574-df8ee1a4b7d7.png)

Temos um html simples com uma imagem. Fiz download da imagem para poder analisá-la.

Usei a ferramenta exiftool para poder analisar os meta dados da imagem:

![image](https://user-images.githubusercontent.com/83584144/211105326-d07936fb-7e67-4f22-aac4-f7d616d2ea4a.png)

Podemos ver que no campo "comment" temos uma pista sobre qualquer coisa, por exemplo um password, mas não me parece pois até agora não temos informação sobre qualquer user ou então poderá ser um diretório da nossa página web.

### Nessus:

No nosso scan do nessus encontramos uma vulnerabilidade critica.

![image](https://user-images.githubusercontent.com/83584144/211105375-eaa34913-8942-4f8c-9dd8-09371af2a704.png)

![image](https://user-images.githubusercontent.com/83584144/211105363-846ea7bd-db38-4c74-b48b-bfe26dab0048.png)

### Nuclei:

Analise do serviço http da máquina:

#### Porta 80:

nuclei -u http://192.168.57.4:80/

![image](https://user-images.githubusercontent.com/83584144/211105408-ebb86857-e81f-4c28-ad2d-6300e9f03d05.png)

Sem informação relevante…

### Nikto:

Scan para tentar encontrar diretórios do serviço http.

#### Porta 80:

nikto-h http://192.168.57.4:80/

![image](https://user-images.githubusercontent.com/83584144/211105421-bf96e214-f5c8-426a-a040-816ff2e4c216.png)

Encontramos o diretório /phpmyadmin.

![image](https://user-images.githubusercontent.com/83584144/211105445-3fe52696-cec1-4503-8fe6-3d723d75d89a.png)

### Gobuster:

#### Porta 80:

Adicionei a pista encontrada nos meta dados da imagem à lista que usei para enumerar os diretórios, para verificar todos os diretórios existentes e se a pista é mesmo um diretório.

gobuster dir -u http://192.168.57.7:80 -w /opt/directory-list-lowercase-2.3-medium.txt

![image](https://user-images.githubusercontent.com/83584144/211105466-088a2206-95a4-4a62-9338-a5a3c06ff97b.png)

Muito bem, podemos ver que a pista é mesmo um diretório na página web.

## Mapeamento de vulnerabilidades e exploração de vulnerabilidades

### /kzMb5nVYJw/

![image](https://user-images.githubusercontent.com/83584144/211105486-be2300bb-0b96-48e2-b12c-8cb9cd262efd.png)

Ao analisar o código fonte podemos ver que também temos uma dica:

![image](https://user-images.githubusercontent.com/83584144/211105503-139dd25d-3631-4ee9-8a6d-8ffdc2d95e05.png)

Podemos ver que é um formulário http método post e que não esta conectado à base de dados e que a password não é complexa, ora bem se não é complexa podemos tentar um brute force.

hydra 192.168.57.7 http-form-post "/kzMb5nVYJw/index.php:key=^PASS^:invalid key" -l ignore -P /usr/share/wordlists/rockyou.txt -vV -t 64

Passo a explicar o comando acima:

- Em primeiro temos o nosso target: 192.168.57.7;
- Em segundo temos o método do ataque, neste caso é um pedido post do form da página.
- Argumento que específica o caminho do "login" do form, em que indica o campo da "key" usar o wordlist indicada e também indica a mensagem de erro ao inserir uma password incorreta.
- Neste caso so existe um campo no "login" ignoramos o user.
- Caminho para a wordlist que utilizamos, neste caso o rockyou.txt.
- Verbose a true.
- Usar o máximo de threads possível, neste caso 64.

![image](https://user-images.githubusercontent.com/83584144/211105527-d0071c24-649d-417b-bec6-c18c6eb4c5a7.png)

Ao fim de alguns segundos encontramos a password: elite.

![image](https://user-images.githubusercontent.com/83584144/211105559-6606cd58-db1d-404d-8cbe-e0074e746ebd.png)

Como até ao momento não se sabe de nenhum username, enviou se um pedido com o campo vazio e obteve se o seguinte resultado:

![image](https://user-images.githubusercontent.com/83584144/211105567-de2c546e-aa56-4a12-80a3-d8cd0dba3bd7.png)

Com o uso de uma ferramenta estudada nas aulas podemos testar se este pedido é vulnerável a SQL Injection.

###SQLMap:

Seguindo o exemplo que tem nos siles da unidade curricular.

Começamos por verificar se é vulnerável a Injections:

sqlmap 192.168.57.7/kzMb5nVYJw/420search.php?usrtosearch= --forms --batch –dbs

![image](https://user-images.githubusercontent.com/83584144/211105651-3da547a7-760c-45e3-9350-b8a0792a81ca.png)

Guardar o request com o uso da ferramenta burpsuite:

![image](https://user-images.githubusercontent.com/83584144/211105662-2f812385-f3f8-4376-9982-8c85b39de13d.png)

Guardei como request.txt.

sqlmap -r request.txt --dbs

![image](https://user-images.githubusercontent.com/83584144/211105671-2101bd3b-a304-45bf-a1a5-a08d1e498a57.png)

Analisando todas as DB só na "seth" é que obtivemos um resultado relevante para o nosso objetivo.

sqlmap -r request.txt -D seth –tables

![image](https://user-images.githubusercontent.com/83584144/211105691-fb533b86-5525-40ed-a5de-43310315b405.png)

Temos uma tabela denominada "users".

Vamos realizar o dump dessa tabela e analisar os dados fornecidos.

sqlmap -r request.txt -D seth -T users --dump

![image](https://user-images.githubusercontent.com/83584144/211105713-499e851b-04b7-47dd-bea9-d74364ce51ad.png)

Encontramos os users da máquina e as suas respetivas passwords em hash.

A partir de um site na internet consegui descobrir que a string que representa a pass esta em base64 e ao fazer o seu decode obtemos a hash da password:

![image](https://user-images.githubusercontent.com/83584144/211105722-72561875-b68f-478e-b301-280bc3c95947.png)

Temos a hash realizamos o decode da mesma também:

![image](https://user-images.githubusercontent.com/83584144/211105736-25cd3f69-5f35-4220-90f5-27c2d4dfcccc.png)

Obtemos a password omega.

Tentamos realizar uma ligação ssh então como user ramses e usamos a password encontrada:

ssh [ramses@192.168.57.7](mailto:ramses@192.168.57.7) -p 777

![image](https://user-images.githubusercontent.com/83584144/211105744-db49c4b4-0b92-4723-9fd0-b04d430d58ca.png)

Ps: Não esquecer que temos de indicar a porta do ssh pois o serviço não se encontra na porta default.

Obtemos com sucesso uma sessão ssh para a máquina.

## Escalada de privilégios:

Com a dica do enunciado, vamos ver analisar o "history" e tentar perceber o que o utilizador andou a fazer.

![image](https://user-images.githubusercontent.com/83584144/211105763-58f82b59-26b7-43a5-96b0-bb288f801bc7.png)

Podemos concluir que andou a mexer do diretório "/var/www/backup", sendo assim vamos dar uma vista de olhos nesse diretório.

![image](https://user-images.githubusercontent.com/83584144/211105777-d6c30b47-821c-4a4e-8191-517162a2049b.png)

Ao executar então o "./procwath" temos o seguinte resultado:

![image](https://user-images.githubusercontent.com/83584144/211105784-5851a3ff-d6a0-4ba1-9257-5a21483d2125.png)

Este resultado é muito parecido ao do comando "ps" que por acaso aparece no "history".

No "history" da linha 39 à linha 42 o utilizador dá dicas para poder abusar do programa existente no diretório pois já que ele é executado como root e usa outro programa, ao alterar o programa que usa para um que abra uma Shell, essa Shell será executada como root. Todos os programas seguem o "$PATH" default para conseguir ser executados de forma correta, o que podemos fazer, e o que o utilizador fez é criamos o programa "ps" para executar uma Shell, linha 40 do "history", mudamos o $PATH para o diretório onde foi criado o "ps", linha 41 do "history", isto faz com que o programa "./prowatch" em vez de utilizar o "ps" deafult do Linux vai usar o que criamos, sendo assim, ao executar o "./procwacth" temos um Shell como root;

![image](https://user-images.githubusercontent.com/83584144/211105819-247beb32-d792-477d-91fb-861c7d6fbcd6.png)

Como podemos ver estamos como root.

Fui a procura da flag no diretório /root/ como sempre e encontrei este resultado:

![image](https://user-images.githubusercontent.com/83584144/211105833-786b19a3-9a39-4451-a24a-c9c7c0ff041a.png)
