
# Relatório Técnico — Achados & Perdidos

Disciplina: Computação em Nuvem

## 1. Objetivo

O objetivo desta atividade foi configurar um ambiente Docker para executar uma aplicação de achados e perdidos com banco de dados PostgreSQL.

A aplicação utiliza Python e Flask, enquanto o Docker Compose gerencia os contêineres, a rede e os volumes.

A principal finalidade é permitir que todo o ambiente seja iniciado com um único comando e que os dados permaneçam armazenados mesmo após a recriação dos contêineres.

## 2. Dockerfile da aplicação

O arquivo app/Dockerfile foi criado para construir a imagem do serviço web.

### FROM python:3.12-slim

Define a imagem base utilizada na construção.

Foi escolhida a versão Slim do Python 3.12 porque contém os componentes necessários para executar a aplicação, utilizando uma base mais enxuta.

### WORKDIR /app

Define o diretório /app como pasta de trabalho dentro do contêiner.

Os próximos comandos de cópia e execução utilizam esse diretório como referência.

### COPY requirements.txt .

Copia o arquivo de dependências Python para dentro da imagem.

Essa instrução aparece antes da cópia do app.py para aproveitar o cache de camadas do Docker.

### RUN pip install --no-cache-dir -r requirements.txt

Instala as bibliotecas Flask, Psycopg e Gunicorn.

O parâmetro --no-cache-dir evita guardar o cache de downloads do pip na imagem.

### COPY app.py .

Copia o código da aplicação Flask para o diretório /app.

Como as dependências foram instaladas antes, alterações somente no app.py não exigem necessariamente uma nova instalação das bibliotecas.

### RUN useradd, mkdir e chown

Cria o usuário appuser, prepara o diretório /app/uploads e define esse usuário como proprietário da pasta.

O diretório será utilizado para armazenar as fotos enviadas pelos usuários.

### USER appuser

Define que a aplicação será executada com o usuário appuser, sem utilizar root.

Essa configuração atende ao requisito de segurança RNF04.

### EXPOSE 5000

Documenta que a aplicação utiliza a porta 5000 dentro do contêiner.

A porta 5000 é utilizada internamente. O acesso externo à aplicação ocorre por meio do Nginx, configurado no Docker Compose.

### CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]

Define o comando utilizado para iniciar a aplicação.

O Gunicorn executa o Flask e escuta na porta 5000, aceitando conexões nas interfaces de rede do contêiner.

O comando foi escrito na forma exec, utilizando colchetes.

### Ordem das instruções

O Dockerfile copia primeiro o requirements.txt, instala as dependências e depois copia o app.py.

Essa ordem aproveita o sistema de cache de camadas do Docker.

Durante os testes, ao executar novamente o build, várias etapas apareceram como CACHED, demonstrando o reaproveitamento das camadas anteriores.

A imagem construída apresentou 219 MB de uso em disco, conforme o comando docker image ls.

## 3. Docker Compose

O arquivo docker-compose.yml define toda a estrutura necessária para executar a aplicação.

### services

Define três serviços principais: db, web e nginx. Também existe o serviço opcional adminer, ativado somente com o profile debug.

### Serviço db

Utiliza a imagem oficial postgres:16-alpine.

As variáveis POSTGRES_DB, POSTGRES_USER e POSTGRES_PASSWORD configuram o banco e são obtidas do arquivo .env.

O banco utiliza dois tipos de montagem:

- Volume nomeado dados_banco, montado em /var/lib/postgresql/data.
- Bind mount do arquivo db/init.sql, montado como somente leitura em /docker-entrypoint-initdb.d/init.sql.

O init.sql cria a tabela itens e insere três registros na primeira inicialização de um banco com o diretório de dados vazio.

Foi configurado um healthcheck com pg_isready para verificar se o PostgreSQL está aceitando conexões.

O banco não possui porta publicada no host. A porta 5432 é utilizada somente para a comunicação interna.

### Serviço web

Utiliza o Dockerfile localizado na pasta app para construir a imagem da aplicação.

As variáveis DB_HOST, DB_PORT, DB_NAME, DB_USER e DB_PASSWORD permitem que o Flask se conecte ao PostgreSQL.

A variável UPLOAD_DIR define /app/uploads como diretório de armazenamento das fotos.

O serviço web utiliza a porta interna 5000, sem publicá-la diretamente no host. O Nginx publica a porta 8080 do computador, com mapeamento 8080:80, e encaminha as requisições para web:5000.

O volume nomeado fotos é montado em /app/uploads.

A opção depends_on com condition: service_healthy determina que o serviço web espere o banco ficar saudável antes de iniciar.

Os serviços principais utilizam a política de reinicialização restart: unless-stopped.

### Serviço nginx

Utiliza a imagem nginx:alpine e funciona como proxy reverso.

Publica a porta 8080 do computador, encaminhada para a porta 80 do contêiner.

O arquivo nginx/default.conf é montado como somente leitura e configura o encaminhamento das requisições HTTP para http://web:5000.

Dessa forma, o usuário acessa o Nginx, enquanto a aplicação Flask permanece acessível somente pela rede Docker.

### Serviço adminer

Utiliza a imagem adminer:latest e permite administrar o PostgreSQL por uma interface web.

O serviço está associado ao profile debug, portanto não é iniciado durante a execução normal do projeto.

Quando ativado, publica a porta 8081 somente no endereço 127.0.0.1 do computador.

O Adminer utiliza o endereço db para acessar o PostgreSQL pela rede Docker.

### Rede rede_interna

Foi criada uma rede Docker do tipo bridge para a comunicação entre os serviços.

O Docker possui resolução de nomes interna, permitindo que a aplicação encontre o PostgreSQL utilizando o nome db.

Na inspeção realizada, a rede utilizava a sub-rede 172.21.0.0/16.

O serviço web recebeu o IP 172.21.0.3 e o banco recebeu o IP 172.21.0.2.

Esses endereços podem mudar quando os contêineres são recriados. Por isso, utilizamos o nome do serviço em vez de um endereço IP fixo.

### Volumes

Foram definidos dois volumes nomeados:

- dados_banco: armazena os arquivos do PostgreSQL.
- fotos: armazena as imagens cadastradas.

Esses volumes são gerenciados pelo Docker e possuem ciclo de vida independente dos contêineres.

## 4. Caminho de uma requisição POST com foto

O cadastro de um objeto com foto acontece da seguinte maneira:

1. O usuário executa uma requisição HTTP POST utilizando o curl, informando descrição, local e uma imagem.

2. A requisição chega à porta 8080 do computador.

3. O Docker encaminha a requisição para a porta 80 do contêiner nginx. O Nginx atua como proxy reverso e repassa a solicitação para web:5000.

4. O Gunicorn recebe a conexão HTTP e encaminha a requisição para a aplicação Flask.

5. O Flask executa a função cadastrar(), que lê os campos descricao, local e foto.

6. A aplicação verifica se descrição e local foram informados.

7. Quando existe uma foto, o Python gera um nome de arquivo utilizando UUID e salva a imagem no diretório /app/uploads.

8. Como esse diretório está associado ao volume fotos, o arquivo é armazenado no volume Docker.

9. A aplicação utiliza o Psycopg para se conectar ao PostgreSQL pelo endereço db:5432, utilizando a rede interna.

10. O PostgreSQL recebe o comando INSERT e armazena a descrição, o local e o nome da foto na tabela itens.

11. O banco retorna o identificador do novo registro.

12. A aplicação devolve uma resposta HTTP 201 Created com o ID e o nome da foto.

Durante o teste realizado, o objeto Oculos de grau foi cadastrado com ID 4 e uma imagem JPG.

O PostgreSQL armazenou as informações do cadastro, enquanto a imagem foi armazenada separadamente no volume fotos.

## 5. Testes funcionais

Após executar docker compose up -d --build, foram realizados testes nas rotas da aplicação.

### GET /health

A aplicação retornou status ok e banco conectado, demonstrando que o Flask conseguiu estabelecer comunicação com o PostgreSQL.

### GET /itens

A aplicação listou os três registros criados inicialmente pelo init.sql.

### GET /itens?local=biblio

O filtro retornou o objeto Garrafa termica azul, cadastrado no local Biblioteca.

### POST /itens

Foi realizado o cadastro de objetos com fotos utilizando curl e envio de formulário multipart.

A aplicação retornou HTTP 201 Created.

### GET /estatisticas

Após os cadastros de teste, a aplicação registrava sete itens no banco e quatro fotos no volume.

### GET /fotos

As fotos dos óculos e da carteira foram acessadas após a recriação dos contêineres, ambas retornando HTTP 200.

## 6. Testes de persistência

Foram realizados testes para verificar o comportamento dos volumes Docker.

### Teste 1 — Remoção e recriação dos contêineres

Antes da remoção, a aplicação possuía:

- 7 itens cadastrados.
- 4 fotos armazenadas.

Foi executado o comando docker compose down, removendo os contêineres e a rede.

Em seguida, o ambiente foi iniciado novamente com docker compose up -d.

Depois da recriação, as estatísticas continuaram mostrando sete itens e quatro fotos.

Isso aconteceu porque o comando down não remove os volumes nomeados.

As fotos também continuaram acessíveis, retornando HTTP 200.

### Teste 2 — Inspeção dos volumes

O comando docker volume ls confirmou a existência dos volumes:

- achados-perdidos_dados_banco
- achados-perdidos_fotos

O comando docker volume inspect mostrou os caminhos de armazenamento no ambiente do Docker:

/var/lib/docker/volumes/achados-perdidos_dados_banco/_data

/var/lib/docker/volumes/achados-perdidos_fotos/_data

Também foi executado docker compose exec web ls -l /app/uploads.

O resultado mostrou as quatro imagens armazenadas com proprietário appuser.

### Teste 3 — Remoção dos volumes

Foi executado docker compose down -v.

Esse comando removeu os contêineres, a rede e os dois volumes nomeados do projeto.

Ao iniciar novamente o ambiente, o Docker criou novos volumes vazios.

O PostgreSQL executou novamente o init.sql, criando a tabela e inserindo os três registros iniciais.

As estatísticas passaram a mostrar:

- 3 itens no banco.
- 0 fotos no volume.

Isso demonstrou que os dados anteriores foram apagados junto com os volumes.

## 7. Bônus implementados

Além dos requisitos principais, foram implementados três recursos opcionais: Nginx como proxy reverso, backup e restauração do volume de fotos e Adminer com profile debug.

### 7.1. Bônus 1 — Nginx como proxy reverso

Foi criado o arquivo nginx/default.conf para configurar o encaminhamento das requisições HTTP.

O serviço nginx utiliza a imagem nginx:alpine e publica a porta 8080 no computador.

O Nginx recebe a requisição e a encaminha para o endereço web:5000, utilizando a resolução de nomes da rede Docker.

O serviço web deixou de publicar uma porta diretamente no host.

Após a configuração, foram realizados testes de inicialização dos contêineres e uma requisição à rota /health.

O retorno HTTP 200 confirmou que o Nginx conseguiu encaminhar a requisição para o Flask e que a aplicação conseguiu consultar o PostgreSQL.

As evidências foram registradas nos arquivos bonus_nginx_subida.txt, bonus_nginx_status.txt e bonus_nginx_health.txt.

### 7.2. Bônus 2 — Backup e restauração do volume

Foi realizado um backup do volume fotos utilizando um contêiner temporário baseado na imagem alpine:3.20.

O volume original foi montado como somente leitura, evitando alterações durante a cópia.

O comando tar foi utilizado para gerar o arquivo compactado fotos_backup.tar.gz, armazenado em docs/evidencias.

Para verificar o procedimento, foi criado um volume separado chamado achados-perdidos_fotos_restauradas.

O backup foi extraído nesse volume, sem modificar as fotos originais.

Em seguida, foi realizada uma comparação entre os arquivos originais e restaurados utilizando o comando cmp.

O teste retornou a mensagem OK: arquivos identicos, comprovando que a foto restaurada possuía o mesmo conteúdo da original.

As evidências foram registradas nos arquivos bonus_backup_conteudo.txt, bonus_restauracao.txt e bonus_comparacao_backup.txt.

Esse procedimento demonstrou como realizar uma cópia de segurança de um volume Docker e testar sua restauração.

### 7.3. Bônus 3 — Adminer com profile debug

Foi adicionado o serviço adminer ao docker-compose.yml, utilizando o profile debug.

O painel pode ser iniciado com o comando:

```bash
docker compose --profile debug up -d
```

O acesso é realizado pelo endereço http://localhost:8081.

A porta foi vinculada ao endereço 127.0.0.1, restringindo seu acesso ao computador local.

O Adminer foi utilizado para acessar o PostgreSQL e consultar a tabela itens.

Durante o teste, foram visualizados quatro registros: garrafa térmica azul, carregador de notebook USB-C, casaco preto com capuz e óculos de grau.

Depois da verificação, o contêiner adminer foi parado e removido sem interromper os outros serviços.

O comando docker compose ps confirmou que apenas db, web e nginx permaneceram em execução.

As evidências foram registradas nos arquivos bonus_adminer_subida.txt, bonus_adminer_status.txt e bonus_adminer_profile_desativado.txt.

### 7.4. Verificação final do ambiente

Após a implementação dos bônus, foi realizado um último teste da rota /health por meio do Nginx.

A requisição para http://localhost:8080/health retornou HTTP 200 OK, com status ok e banco conectado.

O PostgreSQL permaneceu saudável e os serviços web e nginx continuaram em execução.

O teste foi salvo em docs/evidencias/bonus_health_final.txt.

## 8. Conclusão

A atividade permitiu aplicar os conceitos de Docker, Dockerfile, Docker Compose, redes bridge, volumes e persistência de dados estudados na disciplina.

A aplicação foi organizada em três serviços principais: nginx, web e db. Também foi configurado o serviço opcional adminer para administração do banco de dados.

Os testes demonstraram que os volumes nomeados preservam os registros e as fotos quando os contêineres são removidos e recriados, desde que os volumes não sejam excluídos.

Também foi verificado que a remoção dos volumes provoca a perda dos dados armazenados, fazendo com que o PostgreSQL execute novamente o script de inicialização.

Como recursos adicionais, foram implementados um proxy reverso com Nginx, um procedimento de backup e restauração das fotos e um painel administrativo com ativação opcional.

Ao final, os serviços principais estavam funcionando, e a aplicação retornou HTTP 200 com a conexão ao banco estabelecida.

O projeto demonstra como o Docker Compose pode organizar uma aplicação em serviços separados e facilitar sua execução e reprodução em outros ambientes.
