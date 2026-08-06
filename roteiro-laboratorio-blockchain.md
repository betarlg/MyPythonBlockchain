# Laboratório Prático: Construindo sua Própria Blockchain em Python
## Guia de Laboratório e Roteiro de Avaliação Individualizada

Este laboratório é um guia prático para construir uma blockchain funcional do zero usando Python. O roteiro é baseado no tutorial clássico *"Build your own blockchain in Python: a practical guide"*, mas com uma modificação crucial: **todas as etapas contêm mecanismos de comprovação criptográfica individualizada**. 

Como cada bloco é encadeado criptograficamente ao anterior, se você utilizar o seu **Nome Completo** e **Número de Matrícula** nas transações, todos os hashes e Proof of Work subsequentes serão exclusivos do seu computador. Não é possível copiar os prints ou resultados de colegas, pois qualquer alteração de dados gerará hashes completamente diferentes.

---

## Objetivos de Aprendizado
1. Compreender a estrutura de dados de uma blockchain (blocos, transações, hashes e encadeamento).
2. Implementar e entender um algoritmo de consenso por Proof of Work (Prova de Trabalho).
3. Expor as funcionalidades da blockchain através de uma API HTTP (Flask).
4. Simular uma rede descentralizada ponto a ponto (P2P) com resolução de conflitos (consenso pela cadeia mais longa).

---

## Pré-requisitos
Antes de começar, certifique-se de ter o Python 3.x instalado em sua máquina. Instale também as bibliotecas necessárias para a API web e requisições HTTP:

```bash
pip install Flask requests
```

---

## ETAPA 1: Estrutura de Dados e Criação do Bloco Gênese

Uma blockchain é um livro-razão público, distribuído e descentralizado que armazena transações em blocos. Cada bloco possui um índice, um registro de data/hora (timestamp), uma lista de transações, um `proof` (prova) e o hash do bloco anterior (`previous_hash`), garantindo a imutabilidade da cadeia.

### 1.1 Código Base: `blockchain.py`
Crie um arquivo chamado `blockchain.py` e adicione a estrutura inicial da classe `Blockchain`:

```python
import hashlib
import json
from time import time
from urllib.parse import urlparse

class Blockchain(object):
    def __init__(self):
        self.chain = []
        self.current_transactions = []
        
        # Criação do bloco gênese (o primeiro bloco da cadeia)
        # O bloco gênese tem um proof padrão de 100 e previous_hash de "1"
        self.new_block(proof=100, previous_hash='1')
        
    def new_block(self, proof, previous_hash=None):
        """
        Cria um novo bloco na Blockchain
        :param proof: O proof dado pelo algoritmo de Proof of Work
        :param previous_hash: Hash do bloco anterior (opcional)
        :return: Novo Bloco
        """
        block = {
            'index': len(self.chain) + 1,
            'timestamp': time(),
            'transactions': self.current_transactions,
            'proof': proof,
            'previous_hash': previous_hash or self.hash(self.chain[-1]),
        }
        
        # Limpa a lista de transações correntes para o próximo bloco
        self.current_transactions = []
        self.chain.append(block)
        return block

    def new_transaction(self, sender, recipient, amount):
        """
        Cria uma nova transação para ir para o próximo bloco minerado
        :param sender: Endereço do remetente
        :param recipient: Endereço do destinatário
        :param amount: Valor da transação
        :return: O índice do bloco que guardará esta transação
        """
        self.current_transactions.append({
            'sender': sender,
            'recipient': recipient,
            'amount': amount,
        })
        return self.last_block['index'] + 1

    @staticmethod
    def hash(block):
        """
        Cria um hash SHA-256 de um Bloco
        :param block: Bloco
        :return: string hexadecimal de 64 caracteres
        """
        # Garante que o dicionário esteja ordenado para evitar hashes inconsistentes
        block_string = json.dumps(block, sort_keys=True).encode()
        return hashlib.sha256(block_string).hexdigest()

    @property
    def last_block(self):
        # Retorna o último bloco da cadeia
        return self.chain[-1]
```

### 1.2 Atividade de Validação Individualizada
Para provar que você configurou corretamente seu ambiente e iniciou a blockchain em seu computador, você criará um script de teste temporário chamado `teste_etapa1.py` na mesma pasta:

```python
# teste_etapa1.py
from blockchain import Blockchain
import json

# 1. Inicializa a blockchain
bc = Blockchain()

# 2. ADICIONE SEUS DADOS INDIVIDUAIS AQUI:
SEU_NOME = "Insira_Seu_Nome_Completo"
SUA_MATRICULA = "Insira_Sua_Matricula"

# Registra uma transação que insere você na blockchain
bc.new_transaction(
    sender="SISTEMA_UNIVERSITARIO",
    recipient=f"{SEU_NOME} ({SUA_MATRICULA})",
    amount=50
)

# 3. Fecha o bloco criando o Bloco #2
bloco2 = bc.new_block(proof=42)

# Imprime o bloco gerado e o hash correspondente
print("--- DADOS DO SEU BLOCO #2 (INDIVIDUALIZADO) ---")
print(json.dumps(bloco2, indent=4, sort_keys=True))
print("\nHASH DO SEU BLOCO #2:")
print(bc.hash(bloco2))
```

### 📝 O que entregar na Etapa 1?
Execute `python teste_etapa1.py` no terminal.
1. **Print do terminal** mostrando a execução do script com o JSON do seu bloco #2 e o hash gerado.
2. **O Hash SHA-256 resultante em formato texto**.
*Nota para avaliação: O professor verificará criptograficamente se o hash fornecido corresponde exatamente ao seu Nome e Matrícula informados.*

---

## ETAPA 2: Algoritmo de Mineração (Proof of Work)

O Proof of Work (PoW) é o mecanismo que impede fraudes e gasto duplo na blockchain. O objetivo é encontrar um número (`proof` ou `nonce`) que, quando combinado com o hash do bloco anterior e processado por SHA-256, resulte em um hash que atenda a uma regra de dificuldade (por exemplo, iniciar com um certo número de zeros).

Nesta etapa, implementaremos o sistema de Proof of Work seguro, que associa criptograficamente o bloco atual ao hash do bloco anterior (`last_hash`), garantindo a segurança de toda a cadeia.

### 2.1 Atualização de `blockchain.py`
Adicione os seguintes métodos na classe `Blockchain` dentro de `blockchain.py`:

```python
    def proof_of_work(self, last_block):
        """
        Algoritmo simples de Proof of Work:
         - Encontra um número 'p' tal que hash(last_proof * p * last_hash) contenha 4 zeros à esquerda
         - last_proof é o proof do bloco anterior, p é o novo proof, last_hash é o hash do bloco anterior
        """
        last_proof = last_block['proof']
        last_hash = self.hash(last_block)

        proof = 0
        while self.valid_proof(last_proof, proof, last_hash) is False:
            proof += 1

        return proof

    @staticmethod
    def valid_proof(last_proof, proof, last_hash):
        """
        Valida a prova: o hash(last_proof, proof, last_hash) contém 4 zeros à esquerda?
        """
        guess = f'{last_proof}{proof}{last_hash}'.encode()
        guess_hash = hashlib.sha256(guess).hexdigest()
        return guess_hash[:4] == "0000"
```

### 2.2 Atividade de Validação Individualizada
Crie o arquivo `teste_etapa2.py` para executar a mineração do seu bloco personalizado:

```python
# teste_etapa2.py
from blockchain import Blockchain
import json
import time

bc = Blockchain()

SEU_NOME = "Insira_Seu_Nome_Completo"
SUA_MATRICULA = "Insira_Sua_Matricula"

# Transação inicializada no Bloco Gênese
bc.new_transaction(
    sender="SISTEMA_UNIVERSITARIO",
    recipient=f"{SEU_NOME} ({SUA_MATRICULA})",
    amount=100
)

# Minerando o bloco seguinte
print("Iniciando mineração baseada nos dados do aluno...")
inicio = time.time()
novo_proof = bc.proof_of_work(bc.last_block)
fim = time.time()

# Fecha o bloco com a prova encontrada
bloco_minerado = bc.new_block(proof=novo_proof)

print("\n--- BLOCO MINERADO COM SUCESSO ---")
print(f"Tempo gasto: {fim - inicio:.4f} segundos")
print(f"Proof encontrado (Nonce): {novo_proof}")
print("\nJSON do Bloco:")
print(json.dumps(bloco_minerado, indent=4, sort_keys=True))
print(f"\nHash do Bloco Anterior usado no cálculo: {bloco_minerado['previous_hash']}")
```

### 📝 O que entregar na Etapa 2?
Execute `python teste_etapa2.py` no terminal.
1. **Print do terminal** mostrando o tempo de execução e o `Proof encontrado (Nonce)`.
2. **O JSON impresso do Bloco**.
*Nota para avaliação: Como o Bloco 1 contém suas credenciais, o previous_hash será exclusivo seu. Consequentemente, o número do Proof (Nonce) que resolve o problem matemático será único para cada aluno.*

---

## ETAPA 3: Exposição da API HTTP (Flask)

Nesta etapa, você transformará sua blockchain em um servidor de rede local usando a biblioteca Flask. Isso permitirá criar transações e minerar blocos por meio de chamadas HTTP.

### 3.1 Escrevendo o Servidor Flask: `app.py`
Crie um arquivo chamado `app.py` na mesma pasta do seu arquivo `blockchain.py`:

```python
# app.py
from flask import Flask, jsonify, request
from uuid import uuid4
from blockchain import Blockchain

# Inicializa o Flask
app = Flask(__name__)

# Gera um endereço único e aleatório para a carteira deste servidor (nó)
# IMPORTANTE: Altere esta string para conter seu Nome e Matrícula (exemplo abaixo)
ALUNO_SENDER = "aluno_nome_completo_matricula"
node_identifier = f"miner_{ALUNO_SENDER}"

# Inicializa a Blockchain
blockchain = Blockchain()

@app.route('/mine', methods=['GET'])
def mine():
    # Executa o algoritmo de Proof of Work para achar a prova do novo bloco
    last_block = blockchain.last_block
    proof = blockchain.proof_of_work(last_block)

    # O minerador recebe uma recompensa pela mineração.
    # O sender é "0" para sinalizar que este bloco gerou uma nova moeda.
    blockchain.new_transaction(
        sender="0",
        recipient=node_identifier,
        amount=1,
    )

    # Adiciona o novo bloco à cadeia
    previous_hash = blockchain.hash(last_block)
    block = blockchain.new_block(proof, previous_hash)

    response = {
        'message': "Novo Bloco Minerado com Sucesso!",
        'index': block['index'],
        'transactions': block['transactions'],
        'proof': block['proof'],
        'previous_hash': block['previous_hash'],
    }
    return jsonify(response), 200

@app.route('/transactions/new', methods=['POST'])
def new_transaction():
    values = request.get_json()

    # Verifica se os campos obrigatórios estão presentes
    required = ['sender', 'recipient', 'amount']
    if not all(k in values for k in required):
        return 'Campos obrigatórios ausentes', 400

    # Cria uma nova transação
    index = blockchain.new_transaction(values['sender'], values['recipient'], values['amount'])

    response = {'message': f'A transação será adicionada ao Bloco {index}'}
    return jsonify(response), 201

@app.route('/chain', methods=['GET'])
def full_chain():
    response = {
        'chain': blockchain.chain,
        'length': len(blockchain.chain),
    }
    return jsonify(response), 200

if __name__ == '__main__':
    # Rodando o servidor local na porta 5000
    app.run(host='0.0.0.0', port=5000)
```

### 3.2 Atividade de Validação Individualizada
1. Abra um terminal e inicie o servidor Flask:
   ```bash
   python app.py
   ```
2. Abra **outro terminal** para interagir com o seu servidor através do comando `curl` (ou use um programa como o Postman):

   * **Passo A:** Adicione uma nova transação com o seu nome. Substitua os campos abaixo pelos seus dados:
     ```bash
     curl -X POST -H "Content-Type: application/json" -d "{\"sender\": \"PROVEDOR_ETAPA3\", \"recipient\": \"SEU_NOME_MATRICULA\", \"amount\": 10}" http://localhost:5000/transactions/new
     ```
   * **Passo B:** Realize a mineração do bloco que consolida essa transação:
     ```bash
     curl -X GET http://localhost:5000/mine
     ```
   * **Passo C:** Visualize o estado atualizado da sua blockchain:
     ```bash
     curl -X GET http://localhost:5000/chain
     ```

### 📝 O que entregar na Etapa 3?
1. **Print do terminal que está executando o Flask** (`python app.py`) mostrando os logs de requisições recebidas (POST `/transactions/new`, GET `/mine`, GET `/chain`).
2. **O JSON completo retornado pela chamada `/chain`**. O JSON deve conter o Bloco 2 com a transação contendo seu nome/matrícula e o Bloco 3 contendo a recompensa de mineração direcionada à sua carteira individualizada (`miner_seu_nome_matricula`).

---

## ETAPA 4: Rede P2P, Registro de Nós e Consenso

Uma blockchain só faz sentido se distribuída em uma rede de computadores. Se diferentes nós tiverem cadeias diferentes, aplicamos a **Regra da Cadeia mais Longa**: o nó adota a maior cadeia válida disponível na rede.

### 4.1 Atualização de `blockchain.py` com Consenso
Para finalizar a sua classe `Blockchain`, importe as bibliotecas adicionais no topo do arquivo `blockchain.py`:
```python
import requests
```

Agora adicione os seguintes métodos necessários para a rede ponto a ponto no corpo da classe `Blockchain`:

```python
    def __init__(self):
        self.chain = []
        self.current_transactions = []
        # Armazena o conjunto de nós registrados na rede
        self.nodes = set()
        
        # Criação do bloco gênese
        self.new_block(proof=100, previous_hash='1')

    def register_node(self, address):
        """
        Adiciona um novo nó à lista de nós da rede
        :param address: Endereço do nó (ex: 'http://192.168.0.5:5000')
        """
        parsed_url = urlparse(address)
        if parsed_url.netloc:
            self.nodes.add(parsed_url.netloc)
        elif parsed_url.path:
            # Permite formatos sem esquema como '127.0.0.1:5001'
            self.nodes.add(parsed_url.path)
        else:
            raise ValueError('Endereço inválido')

    def valid_chain(self, chain):
        """
        Verifica se a blockchain fornecida é válida
        :param chain: Uma blockchain
        :return: True se for válida, False caso contrário
        """
        last_block = chain[0]
        current_index = 1

        while current_index < len(chain):
            block = chain[current_index]

            # 1. Verifica se o hash do bloco anterior está correto
            if block['previous_hash'] != self.hash(last_block):
                return False

            # 2. Verifica se o Proof of Work correspondente é válido
            # Note que passamos o hash do bloco anterior como parâmetro adicional
            if not self.valid_proof(last_block['proof'], block['proof'], self.hash(last_block)):
                return False

            last_block = block
            current_index += 1

        return True

    def resolve_conflicts(self):
        """
        Algoritmo de Consenso: Resolve conflitos substituindo
        nossa cadeia pela mais longa válida disponível na rede.
        :return: True se a cadeia foi substituída, False caso contrário
        """
        neighbours = self.nodes
        new_chain = None

        # Buscaremos apenas por cadeias maiores que a nossa
        max_length = len(self.chain)

        for node in neighbours:
            try:
                response = requests.get(f'http://{node}/chain', timeout=3)
                if response.status_code == 200:
                    length = response.json()['length']
                    chain = response.json()['chain']

                    # Verifica se o comprimento é maior e se a cadeia é estruturalmente válida
                    if length > max_length and self.valid_chain(chain):
                        max_length = length
                        new_chain = chain
            except requests.exceptions.RequestException:
                # Se o nó estiver offline, ignora
                continue

        # Substitui nossa cadeia se encontrarmos uma maior válida
        if new_chain:
            self.chain = new_chain
            return True

        return False
```

### 4.2 Atualização do Servidor `app.py`
Atualize seu arquivo `app.py` adicionando as duas rotas HTTP que faltam para registrar nós e resolver conflitos:

```python
# Adicione no final do arquivo app.py (antes do bloco __main__)

@app.route('/nodes/register', methods=['POST'])
def register_nodes():
    values = request.get_json()

    nodes = values.get('nodes')
    if nodes is None:
        return "Erro: forneça uma lista válida de nós", 400

    for node in nodes:
        blockchain.register_node(node)

    response = {
        'message': 'Novos nós foram integrados à rede com sucesso',
        'total_nodes': list(blockchain.nodes),
    }
    return jsonify(response), 201

@app.route('/nodes/resolve', methods=['GET'])
def consensus():
    replaced = blockchain.resolve_conflicts()

    if replaced:
        response = {
            'message': 'Sua cadeia local de blocos estava desatualizada e foi substituída',
            'new_chain': blockchain.chain
        }
    else:
        response = {
            'message': 'Sua cadeia local é a versão autoritativa e válida',
            'chain': blockchain.chain
        }
    return jsonify(response), 200
```

### 4.3 Atividade de Validação Individualizada
Você simulará uma rede distribuída com **dois nós concorrentes na sua máquina**.

1. **Inicie o Nó A** na porta 5000:
   * Abra um terminal e execute:
     ```bash
     python app.py
     ```
2. **Inicie o Nó B** na porta 5001:
   * Abra um novo terminal. Para rodar o mesmo código em outra porta sem alterar o script, você pode usar uma variável de ambiente ou rodar diretamente no Python definindo a porta. 
   * Modifique temporariamente a última linha do seu `app.py` para permitir escolher a porta via linha de comando ou simplesmente execute o comando alterando a porta em um script rápido ou rodando uma cópia `app_no_b.py` contendo `app.run(host='0.0.0.0', port=5001)`.
   * *Dica didática:* Faça uma cópia do arquivo chamada `app_no_b.py`, mude a porta final para `5001`, e mude a carteira para `miner_no_b_seu_nome_matricula`. Execute:
     ```bash
     python app_no_b.py
     ```

3. **Interaja com os Nós e Force a Resolução do Consenso:**
   * **Passo A (Registrar o Nó B no Nó A):**
     Envie uma requisição para o Nó A informando que o Nó B (porta 5001) faz parte da rede:
     ```bash
     curl -X POST -H "Content-Type: application/json" -d "{\"nodes\": [\"http://localhost:5001\"]}" http://localhost:5000/nodes/register
     ```
   * **Passo B (Registrar o Nó A no Nó B):**
     Informe ao Nó B que o Nó A (porta 5000) faz parte da rede:
     ```bash
     curl -X POST -H "Content-Type: application/json" -d "{\"nodes\": [\"http://localhost:5000\"]}" http://localhost:5001/nodes/register
     ```
   * **Passo C (Criar uma cadeia mais longa no Nó A):**
     Gere uma transação com seus dados e mine **dois blocos seguidos** no Nó A:
     ```bash
     # Cria transação no Nó A
     curl -X POST -H "Content-Type: application/json" -d "{\"sender\": \"PROVEDOR_ETAPA4\", \"recipient\": \"SEU_NOME_MATRICULA\", \"amount\": 15}" http://localhost:5000/transactions/new
     # Mina primeiro bloco no Nó A
     curl -X GET http://localhost:5000/mine
     # Mina segundo bloco no Nó A (para garantir que a cadeia do Nó A seja maior)
     curl -X GET http://localhost:5000/mine
     ```
   * **Passo D (Executar o algoritmo de Consenso no Nó B):**
     Neste momento, o Nó B possui apenas o bloco gênese (comprimento 1), enquanto o Nó A possui uma cadeia de blocos maior (comprimento 3). Chame o consenso no Nó B para forçá-lo a sincronizar:
     ```bash
     curl -X GET http://localhost:5001/nodes/resolve
     ```

### 📝 O que entregar na Etapa 4?
1. **Print do terminal do Nó B** mostrando a resposta da rota `/nodes/resolve`. A mensagem deve dizer *"Sua cadeia local de blocos estava desatualizada e foi substituída"* e listar a cadeia atualizada.
2. **O JSON final retornado pelo Nó B**. Note que o bloco gênese do Nó B será integrado com a cadeia vinda do Nó A, comprovando criptograficamente a adoção da cadeia que contém seu nome e matrícula.

---

## Critérios de Avaliação e Correção (Para o Professor)
Cada aluno submeterá um relatório contendo os prints e os códigos JSON de cada etapa. A validação é simples e direta:
1. **Dificuldade de Fraude:** O hash de cada bloco contém os dados pessoais do aluno. Se o aluno tentar clonar o relatório de um colega, o professor poderá re-haxear os blocos de teste com os dados do aluno em um validador rápido. Se os hashes não coincidirem, a fraude é identificada.
2. **Exclusividade do Proof of Work:** O proof (nonce) gerado em cada etapa é exclusivo. Como os dados e os timestamps são gerados em tempo de execução de forma individual, é estatisticamente impossível que dois alunos gerem exatamente o mesmo `proof` e os mesmos `hashes`.
