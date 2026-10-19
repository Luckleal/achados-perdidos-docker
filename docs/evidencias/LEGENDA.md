# Legendas das Evidências — Achados & Perdidos

As evidências foram obtidas durante a execução real da aplicação com Docker Compose.

## 1. Inicialização do ambiente

- **subida_ambiente.txt:** Demonstra a construção da imagem e a inicialização dos serviços com Docker Compose.
- **status_containers.txt:** Mostra os dois serviços da configuração inicial em execução e o PostgreSQL com status healthy.
- **imagem_web.txt:** Apresenta as imagens construídas e o tamanho de 219 MB informado pelo Docker.
- **usuario_web.txt:** Comprova que a aplicação utiliza o usuário appuser, sem privilégios de root.

## 2. Testes funcionais

- **teste_health.txt:** Comprova que a aplicação está funcionando e consegue acessar o PostgreSQL.
- **teste_itens.txt:** Demonstra a listagem dos três objetos inseridos pelo init.sql.
- **teste_filtro.txt:** Comprova o funcionamento do filtro por local, retornando o objeto da Biblioteca.
- **teste_estatisticas.txt:** Mostra os três registros iniciais e nenhuma foto após a reinicialização do banco.
- **teste_post_com_foto.txt:** Demonstra o cadastro dos óculos com foto, retornando HTTP 201 Created.
- **teste_estatisticas_apos_post.txt:** Confirma quatro itens no banco e uma foto no volume após o cadastro.
- **teste_foto_apos_post.txt:** Comprova que a imagem cadastrada pode ser recuperada pela aplicação, retornando HTTP 200.
- **teste_foto_volume_atual.txt:** Mostra a imagem cadastrada no diretório /app/uploads, pertencente ao usuário appuser.
- **teste_logs_web.txt:** Apresenta os logs de inicialização do Gunicorn na porta 5000.

## 3. Persistência dos dados

- **estatisticas_antes.txt:** Registra sete itens e quatro fotos antes da remoção dos contêineres.
- **estatisticas_depois.txt:** Demonstra que os sete itens e as quatro fotos permaneceram após a recriação dos contêineres.
- **fotos_apos_recriacao.txt:** Comprova que as fotos dos óculos e da carteira continuaram acessíveis após a recriação.
- **antes_remocao_volumes.txt:** Registra os dados e o acesso às fotos antes da remoção definitiva dos volumes.
- **estatisticas_apos_remocao_volumes.txt:** Mostra três itens e zero fotos após remover os volumes e reinicializar o ambiente.
- **itens_apos_remocao_volumes.txt:** Demonstra que somente os três objetos iniciais do init.sql foram recriados.

## 4. Inspeção dos volumes e da rede

- **volumes_lista.txt:** Apresenta os volumes Docker existentes, incluindo os dois volumes do projeto.
- **volume_fotos_inspect.txt:** Mostra as informações e o caminho de armazenamento do volume fotos.
- **volume_banco_inspect.txt:** Mostra as informações e o caminho de armazenamento do volume dados_banco.
- **conteudo_fotos.txt:** Confirma os quatro arquivos armazenados no volume antes de sua remoção.
- **rede_inspect.txt:** Demonstra a rede bridge e os endereços dos contêineres web e db durante a configuração inicial.

## 5. Escalabilidade

- **teste_escalabilidade.txt:** Registra a tentativa de executar três réplicas web e o conflito causado pela publicação fixa da porta 8080.

## 6. Bônus 1 — Nginx

- **bonus_nginx_subida.txt:** Registra a inicialização do ambiente após a inclusão do Nginx.
- **bonus_nginx_status.txt:** Mostra o Nginx publicando a porta 8080 e o Flask utilizando somente a porta interna.
- **bonus_nginx_health.txt:** Comprova que a requisição passou pelo Nginx e retornou HTTP 200, com o banco conectado.

## 7. Bônus 2 — Backup e restauração

- **fotos_backup.tar.gz:** Arquivo compactado gerado a partir do volume fotos.
- **bonus_backup_conteudo.txt:** Registra o conteúdo utilizado no backup.
- **bonus_restauracao.txt:** Demonstra a restauração dos arquivos em um volume separado.
- **bonus_comparacao_backup.txt:** Comprova que o arquivo restaurado é idêntico ao original.

## 8. Bônus 3 — Adminer

- **bonus_adminer_subida.txt:** Registra a inicialização do Adminer com o profile debug.
- **bonus_adminer_status.txt:** Mostra os quatro serviços em execução com o profile ativado.
- **bonus_adminer_profile_desativado.txt:** Comprova que o Adminer foi removido e os serviços db, web e nginx permaneceram funcionando.

O painel também foi utilizado para consultar a tabela itens e visualizar os quatro registros existentes no banco.

- **bonus_adminer_tabela.png:** Captura de tela do Adminer mostrando os quatro registros da tabela itens no PostgreSQL.

## 9. Verificação final

- **bonus_health_final.txt:** Registra a resposta HTTP 200 do Nginx, confirmando o funcionamento do Flask e a conexão com o PostgreSQL.

## Conclusão

Os resultados demonstram o funcionamento da aplicação com Docker Compose, rede Docker e volumes nomeados. Também registram a implementação do Nginx como proxy reverso, o backup e a restauração das fotos e o Adminer com profile debug.

Também comprovam que os dados sobrevivem à recriação dos contêineres, mas são eliminados quando os volumes são removidos com docker compose down -v.
