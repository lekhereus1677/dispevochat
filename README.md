## 📘 Tutorial de Automação de Disparo de Campanhas

Vamos começar o tutorial para fazer a automação do sistema de disparo de campanhas usando o n8n e a Evolution API junto ao ChatWoot. 

Antes de iniciar, certifique-se de que você já tem instalado:

- ChatWoot
- n8n
- Evolution API
- pgAdmin ou outro de sua preferência para acessar o banco de dados do Postgres

### Passo 1: Criar uma Caixa de Entrada de Canal SMS do Tipo Bandwidth

1. **Acesse o ChatWoot**: Faça login na sua conta do ChatWoot.
2. **Configurações**: Vá para a seção de configurações.
3. **Caixas de Entrada**: Selecione "Caixas de Entrada" no menu.
4. **Adicionar Nova Caixa de Entrada**: Clique no botão "Adicionar Nova Caixa de Entrada".
5. **Escolher Tipo de Canal**: Selecione "SMS" e escolha "Bandwidth" como o tipo de canal.
6. **Configurar Detalhes do Canal**:
   - Nome da Caixa de Entrada: Disparador (ou o nome que preferir).
   - Número de telefone: +741963
   - ID da Conta: 741963
   - ID da aplicação: 741963
   - Chave API: 741963
   - Chave secreta API: 741963
7. **Salvar Configurações**: Clique em "Criar canal Bandwidth" para criar a nova caixa de entrada.

### Passo 2: Adicionar Colunas no Banco de Dados do ChatWoot

1. **Acesse o Banco de Dados**: Use o pgAdmin ou outro software de sua preferência para acessar o banco de dados do ChatWoot.
2. **Adicionar Coluna na Tabela Accounts(Botão direito em Tables e Selecionar Querry Tools)**:
   - Execute o seguinte comando SQL para adicionar a coluna `limite_disparo`:
     ```sql
     ALTER TABLE accounts
     ADD COLUMN limite_disparo INTEGER NOT NULL DEFAULT 100;
     ```
3. **Adicionar Colunas na Tabela Campaigns**:
   - Execute os seguintes comandos SQL para adicionar as colunas `status_envia`, `enviou` e `falhou`:
     ```sql
     ALTER TABLE campaigns
     ADD COLUMN status_envia INTEGER NOT NULL DEFAULT 0;
     
     ALTER TABLE campaigns
     ADD COLUMN enviou INTEGER NOT NULL DEFAULT 0;
     
     ALTER TABLE campaigns
     ADD COLUMN falhou INTEGER NOT NULL DEFAULT 0;
     ```

4. **Adicionar nova Tabela para guardar os envios que falharem**:
   - Execute o seguinte comando SQL para adicionar a tabela campaigns_failled:
     ```sql
      -- Cria a sequência
      CREATE SEQUENCE campaigns_failled_id_seq;
      
      -- Cria a tabela com a coluna `id` usando a sequência criada
      CREATE TABLE campaigns_failled (
        id BIGINT PRIMARY KEY NOT NULL DEFAULT nextval('campaigns_failled_id_seq'::regclass),
        nomecontato TEXT NOT NULL,
        telefone CHARACTER VARYING NOT NULL,
        id_campanha INTEGER NOT NULL
      );
     ```

## 🛠️ OBRIGATORIO ❗ - 🚨 CORREÇÃO NO BANCO DE DADOS DO CHATWOOT ⚠️
### Após aplicar esta correção é recomendavel recriar as etiquetas (marcadores).

- Foi notado que os ID da tabela "labels" não condizia com os id ta tabela "tags" sendo assim criei algumas funções e triggers que corrigem esse problema.

5. **Criação das Funções de Replicação, Exclusão e Atualização (Botão direito em Trigger Functions e selecionar QUERRY TOOLS)**

   ***Cria na raiz do banco de dados***
   
      **Função para replicar inserções:**
      
      ```sql
      CREATE OR REPLACE FUNCTION replicate_labels_to_tags()
      RETURNS TRIGGER AS $$
      BEGIN
          INSERT INTO tags (id, name)
          VALUES (NEW.id, NEW.title);
          RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
      ```
      
      **Função para replicar exclusões:**
      
      ```sql
      CREATE OR REPLACE FUNCTION delete_labels_from_tags_and_taggings()
      RETURNS TRIGGER AS $$
      BEGIN
          -- Exclui da tabela tags
          DELETE FROM tags WHERE id = OLD.id;
          -- Exclui da tabela taggings
          DELETE FROM taggings WHERE tag_id = OLD.id;
          RETURN OLD;
      END;
      $$ LANGUAGE plpgsql;
      ```
      
      **Função para replicar atualizações:**
      
      ```sql
      CREATE OR REPLACE FUNCTION update_labels_to_tags()
      RETURNS TRIGGER AS $$
      BEGIN
          UPDATE tags
          SET name = NEW.title
          WHERE id = NEW.id;
          RETURN NEW;
      END;
      $$ LANGUAGE plpgsql;
      ```

6. **Criação dos Triggers**

   ***Criar na tabela labels***
   
      **Trigger para inserções:**
      
      ```sql
      CREATE TRIGGER after_insert_labels
      AFTER INSERT ON labels
      FOR EACH ROW
      EXECUTE FUNCTION replicate_labels_to_tags();
      ```
      
      **Trigger para exclusões:**
      
      ```sql
      CREATE TRIGGER after_delete_labels
      AFTER DELETE ON labels
      FOR EACH ROW
      EXECUTE FUNCTION delete_labels_from_tags_and_taggings();
      ```
      
      **Trigger para atualizações:**
      
      ```sql
      CREATE TRIGGER after_update_labels
      AFTER UPDATE ON labels
      FOR EACH ROW
      EXECUTE FUNCTION update_labels_to_tags();
      ```

---

### Passo 3: Importar Workflows no n8n

1. **Acesse o n8n**: Faça login na sua instância do n8n.
2. **Adicionar Novo Workflow**:
   - Clique em "Add Workflow".
3. **Importar Workflow**:
   - Clique nos três pontinhos no canto superior direito.
   - Selecione "Import from File".
4. **Importar o Fluxo Disparador**:
   - Importe o arquivo de workflow disparador.json.
5. **Importar o Fluxo Reset-Limite-Campanhas**:
   - Repita os passos acima e importe o reset-limite-campanha.json.

### Passo 4: Editar o Workflow Disparador no n8n

1. **Acesse o Workflow Disparador**: No n8n, abra o workflow Disparador que você importou.
2. **Editar Nó Info_Base**:
   - Preencha os seguintes campos com suas informações:
     - **URL do ChatWoot**
     - **URL da Evolution API**
     - **Token de acesso da conta do ChatWoot**
     - **Global API KEY da Evolution API**
     - **Nome da Caixa de Entrada cadastrada na Evolution API que vai disparar as mensagens**
     - **ID da conta do ChatWoot**
     - **Email que vai receber o relatório**
     - **Número do WhatsApp que vai receber o relatório**
3. **Editar Nó Buscar campanhas**:
   - Edite "account_id" pelo id da instancia do ChatWoot.
   - Edite "inbox_id" pelo id da caixa de entrada do disparador que voce crio no **Passo 1**.
4. **Conectar Nós do Postgres ao Banco de Dados do ChatWoot**:
   - Conecte todos os nós do Postgres ao banco de dados do ChatWoot, garantindo que as informações fluam corretamente entre os sistemas.

### Passo 5: Editar o Workflow reset-limite-campanha no n8n

1. **Acesse o Workflow reset-limite-campanha**: No n8n, abra o workflow reset-limite-campanha que você importou.
2. **Conectar Nós do Postgres ao Banco de Dados do ChatWoot**:
   - Conecte todos os nós do Postgres ao banco de dados do ChatWoot, garantindo que as informações sejam atualizadas corretamente para resetar o limite de disparo diário.

---
