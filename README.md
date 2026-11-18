# Cidade Tech — Cloud

Infraestrutura em nuvem da solução Cidade Tech para a Global Solution 2026/2.

O Cidade Tech organiza consultas comunitárias por tema e bairro. Gestores publicam
propostas, moradores autenticados registram um voto e a apuração fica disponível
após o encerramento. As consultas são consultivas e não substituem votações oficiais.

## Arquitetura

A implantação acadêmica usa uma VM Linux no Azure. Docker Compose executa a API e
o Oracle Database Free em containers separados. A API será acessível pela porta
8080; o banco aceita conexões somente pela rede interna do Compose. Um volume
nomeado preserva os arquivos do Oracle entre reinícios dos containers.

O diagrama com os recursos Azure e as decisões de rede está em
[docs/arquitetura.md](docs/arquitetura.md). Os comandos para provisionar a
infraestrutura estão em [docs/azure-cli.md](docs/azure-cli.md); o roteiro
copiável em texto simples está em [azure-cli-comandos.txt](azure-cli-comandos.txt).

## Estado do trabalho

- [x] Proposta e modelo Oracle/PLSQL da disciplina Database.
- [x] Desenho da arquitetura de infraestrutura Azure.
- [x] Compose com Oracle e perfil para a futura API Java.
- [ ] Receber a API Java/Spring Boot e confirmar seu contrato de integração.
- [ ] Executar e validar a integração API–Oracle.
- [ ] Provisionar a VM Azure, implantar containers e coletar evidências.

O Compose mantém a API no perfil `application` até que o código e o Dockerfile
estejam disponíveis em `api/`. A API ainda não foi entregue pela equipe Java.

## Pré-requisitos

- Docker Engine e Docker Compose v2.
- Azure CLI, assinatura Azure com permissão para criar recursos e um IP público
  estável para restringir o acesso de demonstração.
- Código da API Java em `api/` para executar o ambiente completo.

## Executar o Oracle localmente

1. Copie `.env.example` para `.env` e preencha as senhas fortes. O arquivo `.env`
   é ignorado pelo Git; nunca versione credenciais.
2. Inicie o banco:

   ```powershell
   docker compose up -d oracle
   docker compose logs -f oracle
   ```

3. Aguarde o health check ficar `healthy`. O serviço Oracle fica disponível na
   rede Docker como `oracle:1521/FREEPDB1`. A porta 1521 não é publicada no host.
4. Execute os scripts SQL/PLSQL do projeto Database conectando-se ao usuário
   definido em `APP_DB_USER`.

## Executar com a API

Quando o código Java e seu Dockerfile estiverem em `api/`, configure a API para
usar o datasource Oracle fornecido pelo Compose e inicie os dois serviços:

```powershell
docker compose --profile application up -d --build
docker compose ps
docker compose logs -f api
```

Contrato inicial esperado pela API:

- JDBC URL: `jdbc:oracle:thin:@oracle:1521/FREEPDB1`
- Usuário e senha: `APP_DB_USER` e `APP_DB_PASSWORD` do `.env`
- Porta HTTP da aplicação: `8080`

O contrato deverá ser conferido com o responsável pela API antes da integração.

## Segurança e limites da demonstração

- Não há porta publicada para o Oracle; somente a API e o banco compartilham a
  rede Docker.
- A NSG do Azure deve permitir TCP/8080 somente do IP autorizado do avaliador.
- A VM não precisa de SSH público; use Azure Run Command para administração.
- Use somente dados sintéticos. Este MVP não verifica identidade civil nem
  residência e não deve ser apresentado como votação oficial.
- A implantação em uma VM é para demonstração acadêmica, não oferece alta
  disponibilidade. O volume Docker reside no disco da VM; apagar a VM também
  pode apagar os dados.

## Entrega

O provisionamento, o desenho da infraestrutura e o roteiro de validação estão
documentados em `docs/` e em `azure-cli-comandos.txt`. No vídeo final, mostre os
recursos Azure e a solução funcionando; use o arquivo TXT como referência para
os comandos executados. O vídeo deverá mostrar
containers, logs, CRUD e consulta direta ao Oracle, conforme o roteiro da
disciplina.
