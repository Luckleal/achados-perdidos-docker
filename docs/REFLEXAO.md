
# Questões de Reflexão — Achados & Perdidos

## Q1 — Comunicação entre contêineres e DNS

O serviço web consegue acessar o banco pelo nome db porque os dois estão conectados à rede rede_interna, criada pelo Docker Compose.
O DNS interno do Docker resolve o nome db para o endereço IP do contêiner PostgreSQL.
Isso evita configurar endereços IP fixos, que podem mudar quando os contêineres são recriados.
Se os contêineres fossem criados com docker run na rede bridge padrão, não teríamos essa resolução automática por nomes.
Para obter o mesmo comportamento, seria necessário criar uma rede bridge definida pelo usuário e conectar os dois contêineres a ela.
Essa configuração está no docker-compose.yml e foi verificada em docs/evidencias/rede_inspect.txt.


## Q2 — Diferença entre volume nomeado e bind mount

O volume nomeado é gerenciado pelo Docker e utilizado para armazenar dados que precisam continuar existindo após a remoção dos contêineres.

Já o bind mount permite disponibilizar um arquivo ou diretório específico do computador dentro do contêiner.

No projeto, utilizamos o volume dados_banco para manter os registros do PostgreSQL.

O arquivo init.sql foi montado como bind mount porque já existe na pasta db do projeto e precisa ser lido pelo PostgreSQL durante a inicialização.

Também utilizamos a opção :ro para impedir que o contêiner modifique esse arquivo.

Essa diferença foi demonstrada nos testes de persistência e inspeção dos volumes.


## Q3 — Alteração do init.sql após a inicialização

Não. O PostgreSQL executa o init.sql somente quando o diretório de dados é inicializado pela primeira vez.

Como utilizamos o volume dados_banco, o banco já possui seus arquivos e não executa novamente o script ao utilizar docker compose up -d.

Para adicionar uma coluna sem perder os dados, podemos utilizar o comando ALTER TABLE diretamente no banco.

O ideal é realizar um backup antes e aplicar a alteração por meio de uma migração SQL.

Não seria correto utilizar docker compose down -v para isso, pois esse comando remove os volumes e apaga os dados armazenados.


## Q4 — depends_on e healthcheck

O depends_on sozinho controla a ordem de inicialização dos contêineres, mas não garante que o banco já esteja pronto para receber conexões.

O PostgreSQL pode estar iniciado e ainda estar realizando sua preparação interna.

Por isso, configuramos um healthcheck utilizando o comando pg_isready, que verifica se o banco está aceitando conexões.

No serviço web, utilizamos depends_on com condition: service_healthy para aguardar o banco ficar saudável antes de iniciar a aplicação.

Essa configuração foi comprovada pelo docker compose ps, que mostrou o PostgreSQL como healthy, e pelo teste GET /health, que retornou banco conectado.


## Q5 — Cache de camadas do Docker

A ordem das instruções do Dockerfile influencia o tempo de construção porque o Docker utiliza cache de camadas.

No nosso projeto, copiamos primeiro o requirements.txt e instalamos as dependências antes de copiar o app.py.

Dessa forma, se somente o código da aplicação for alterado, o Docker pode reutilizar as camadas anteriores, sem instalar as bibliotecas novamente.

A partir da instrução COPY app.py, as etapas afetadas pela mudança precisam ser reconstruídas.

Durante nossos testes, várias etapas apareceram como CACHED, mostrando que o Docker reutilizou camadas anteriores.

Essa organização torna o processo de build mais rápido e evita operações desnecessárias.


## Q6 — Segurança do banco de dados e LGPD

A porta 5432 do PostgreSQL não foi publicada no host para evitar o acesso direto ao banco e reduzir os riscos de ataques.

Na configuração inicial, o Flask publicava diretamente a porta 8080. Na versão final, somente o Nginx publica essa porta, encaminhando as requisições ao Flask pela rede Docker. O PostgreSQL continua sem porta publicada no host.

Como as fotos dos objetos podem conter documentos e informações pessoais, é necessário proteger esses dados conforme os princípios de segurança da LGPD.

Uma medida adicional seria implementar autenticação e controle de acesso, permitindo que apenas usuários autorizados visualizem as fotos.

Outra medida seria utilizar HTTPS para criptografar a comunicação entre o usuário e a aplicação.

Essas proteções seriam implementadas em uma futura versão, pois o sistema atual ainda não possui esses recursos.

**Atualização após a implementação do Nginx:** Antes, a aplicação Flask publicava diretamente a porta 8080 do computador, com o mapeamento `8080:5000`. Depois, o Nginx passou a publicar a porta 8080, encaminhando as requisições para o Flask pela rede Docker. Assim, a aplicação web e o PostgreSQL permaneceram sem portas publicadas diretamente no host.


## Q7 — Escalabilidade da aplicação

Ao executar docker compose up -d --scale web=3, o Docker tentou criar três réplicas da aplicação, mas ocorreu um erro porque a porta 8080 do host já estava sendo utilizada pelo primeiro contêiner.

Na configuração inicial, seria necessário retirar a publicação fixa da porta do serviço web para evitar o conflito entre as réplicas. Posteriormente, implementamos o Nginx como proxy reverso e removemos essa publicação. O balanceamento entre três réplicas não foi configurado nem testado.

As réplicas poderiam compartilhar o volume fotos no mesmo host Docker, mas seria necessário ter cuidado com os acessos simultâneos, permissões e possíveis alterações nos arquivos.

Em servidores diferentes, um volume local não seria compartilhado automaticamente. Nesse caso, seria mais adequado utilizar um armazenamento compartilhado, como um serviço de armazenamento de objetos na nuvem.

O erro encontrado foi registrado em docs/evidencias/teste_escalabilidade.txt.

**Atualização após a implementação do Nginx:** Antes, o teste com três réplicas do Flask apresentou conflito porque todas tentavam utilizar a mesma porta 8080 do computador. Depois, retiramos essa publicação do serviço web e deixamos somente o Nginx como ponto de entrada. Isso resolveu a configuração que causava o conflito de portas, mas ainda não testamos três réplicas funcionando nem o balanceamento de carga entre elas.


## 5.1 — Do ambiente local para a nuvem

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
|---|---|---|---|
| Contêiner web | Azure Container Apps | Amazon ECS com Fargate | Cloud Run |
| Contêiner db + volume dados_banco | Azure Database for PostgreSQL | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Volume fotos | Azure Blob Storage | Amazon S3 | Cloud Storage |
| rede_interna | Azure Virtual Network (VNet) | Amazon VPC | Google Cloud VPC |
| Arquivo .env (segredos) | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Imagem construída localmente | Azure Container Registry | Amazon ECR | Artifact Registry |

### Qual componente mudaria mais na migração?

Na minha avaliação, o armazenamento das fotos seria uma das partes que mais precisariam mudar.

Atualmente, a aplicação grava as imagens diretamente no diretório /app/uploads, que está associado a um volume Docker.

Na nuvem, seria mais adequado utilizar um serviço de armazenamento de objetos, como Amazon S3, Azure Blob Storage ou Google Cloud Storage.

Para isso, precisaríamos adaptar o código da aplicação para enviar e recuperar as imagens utilizando a API do serviço escolhido.

O PostgreSQL também poderia ser migrado para um serviço de banco gerenciado, mantendo o mesmo tipo de banco de dados.

Essa mudança ajudaria na escalabilidade e permitiria executar várias réplicas da aplicação sem depender de um volume local compartilhado.
