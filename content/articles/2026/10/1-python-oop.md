---
title: Guia geral de programação orientada a objetos com Python
type: docs
weight: 1
editURL: "https://devnotes.msglabs.com.br/articles/python-oop/"
next: /articles/2026/07/1-ansible
---

A programação orientada a objetos (POO) é o paradigma por trás da maioria das linguagens modernas, como Java, C# e Python. Em vez de organizar o código em variáveis soltas e funções avulsas, a POO agrupa dados e comportamentos em torno de entidades do mundo real, representadas por objetos.

Este guia foi organizado para que os conceitos apareçam na ordem em que fazem mais sentido. A ideia é começar com o problema que a POO resolve, criar objetos simples e avançar gradualmente até a organização em pacotes e *decorators*.

Como exemplo principal, será utilizado um pequeno sistema escolar.

> [!NOTE]
> Para acompanhar este artigo, é importante dominar os tópicos listados na seção [0. Pré-requisitos](#0-pré-requisitos). Caso você já os domine, pode seguir direto para a seção [1. Qual problema a POO resolve?](#1-qual-problema-a-poo-resolve).

## 0. Pré-requisitos

Antes de estudar POO, é importante dominar:

- variáveis;
- tipos básicos: `str`, `int`, `float`, `bool`;
- condicionais: `if`, `elif` e `else`;
- laços: `for` e `while`;
- funções;
- listas e dicionários;
- importações básicas.

Exemplo de função comum:

```python
def apresentar_nome(nome):
    print(f"Olá, {nome}!")

apresentar_nome("Ana")
```

Na POO, funções relacionadas a uma mesma entidade passam a ser organizadas dentro de classes.

## 1. Qual problema a POO resolve?

Considere um sistema que cadastra alunos.

Sem POO, poderíamos criar variáveis soltas:

```python
nome_aluno_1 = "Ana"
idade_aluno_1 = 20
matricula_aluno_1 = "2025001"

nome_aluno_2 = "Carlos"
idade_aluno_2 = 22
matricula_aluno_2 = "2025002"
```

Também precisaríamos de funções que recebem vários dados:

```python
def apresentar_aluno(nome, idade, matricula):
    print(f"Aluno: {nome}")
    print(f"Idade: {idade}")
    print(f"Matrícula: {matricula}")
```

Esse modelo se torna confuso conforme o sistema cresce, pois os dados e as funções relacionadas ficam espalhados.

A POO resolve isso ao representar entidades do mundo real como objetos. Um aluno possui:

- dados: nome, idade e matrícula;
- comportamentos: apresentar-se, estudar, receber nota.

> [!IMPORTANT]
> A ideia central é: **uma classe agrupa dados e comportamentos que pertencem à mesma entidade.**

## 2. Classes e objetos

### 2.1 Classe: o molde

Uma classe é uma definição, um modelo ou uma planta para criar objetos.

```python
class Aluno:
    pass
```

A classe acima informa que existe o conceito de Aluno, mas ainda não existe um aluno específico.

Pense assim:

| Conceito | Comparação           |
| -------- | -------------------- |
| Classe   | Planta de uma casa   |
| Objeto   | Casa construída      |
| Classe   | Receita              |
| Objeto   | Prato preparado      |
| Classe   | Modelo de ficha de aluno |
| Objeto   | Ficha da Ana         |

### 2.2 Objeto: uma instância da classe

Um objeto é uma ocorrência concreta criada a partir de uma classe.

```python
class Aluno:
    pass

aluno1 = Aluno()
aluno2 = Aluno()
```

Neste código:

- `Aluno` é a classe;
- `aluno1` é um objeto;
- `aluno2` é outro objeto.

Criar um objeto a partir de uma classe é chamado de **instanciação**.

Mesmo sendo criados a partir da mesma classe, os objetos são independentes:

```python
aluno1.nome = "Ana"
aluno2.nome = "Carlos"

print(aluno1.nome)
print(aluno2.nome)
```

Saída:

```text
Ana
Carlos
```

## 3. Atributos: dados de um objeto

Atributos são características armazenadas em objetos.

```python
class Aluno:
    pass

aluno = Aluno()
aluno.nome = "Ana"
aluno.idade = 20
aluno.matricula = "2025001"

print(aluno.nome)
print(aluno.idade)
print(aluno.matricula)
```

Embora isso funcione, há um problema: os atributos são criados manualmente.

```python
aluno1 = Aluno()
aluno1.nome = "Ana"

aluno2 = Aluno()
aluno2.idade = 20
```

Nesse caso, `aluno2` não tem nome e `aluno1` não tem idade. Isso pode causar erros e objetos incompletos. A solução é usar um construtor.

## 4. Construtor: `__init__`

O construtor é o método especial `__init__`. Ele é executado automaticamente quando um objeto é criado.

```python
class Aluno:
    def __init__(self, nome, idade, matricula):
        self.nome = nome
        self.idade = idade
        self.matricula = matricula
```

Agora, para criar um aluno, todos os dados obrigatórios precisam ser informados:

```python
aluno = Aluno("Ana", 20, "2025001")

print(aluno.nome)
print(aluno.idade)
print(aluno.matricula)
```

**Por que usar construtores?**

O construtor garante que os objetos sejam criados com os dados necessários. Sem construtor, um objeto pode ficar incompleto:

```python
aluno = Aluno()
```

Com construtor:

```python
aluno = Aluno("Ana", 20, "2025001")
```

O objeto já nasce com uma estrutura válida.

### 4.1 Valores padrão

Nem todo dado precisa ser obrigatório.

```python
class Aluno:
    def __init__(self, nome, idade, matricula, ativo=True):
        self.nome = nome
        self.idade = idade
        self.matricula = matricula
        self.ativo = ativo

aluno = Aluno("Ana", 20, "2025001")
print(aluno.ativo)
```

Saída:

```text
True
```

Também é possível fornecer o valor:

```python
aluno_inativo = Aluno("Carlos", 22, "2025002", False)
```

## 5. Entendendo o `self`

O `self` representa o próprio objeto que está sendo usado naquele momento.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome
```

Ao executar:

```python
aluno1 = Aluno("Ana")
aluno2 = Aluno("Carlos")
```

O Python trabalha, de forma simplificada, como se fosse:

```python
Aluno.__init__(aluno1, "Ana")
Aluno.__init__(aluno2, "Carlos")
```

Portanto:

- na criação de `aluno1`, `self` representa `aluno1`;
- na criação de `aluno2`, `self` representa `aluno2`.

Observe:

```python
self.nome = nome
```

- `nome` é o parâmetro recebido;
- `self.nome` é o atributo armazenado no objeto.

Sem `self`, o valor existiria apenas durante a execução do método:

```python
class Aluno:
    def __init__(self, nome):
        nome = nome
```

Esse código não armazena `nome` dentro do objeto.

> [!NOTE]
> A convenção é sempre chamar o primeiro parâmetro dos métodos de instância de `self`.

## 6. Métodos: comportamentos dos objetos

Métodos são funções criadas dentro de uma classe. Eles representam as ações que um objeto pode executar.

```python
class Aluno:
    def __init__(self, nome, idade, matricula):
        self.nome = nome
        self.idade = idade
        self.matricula = matricula

    def apresentar(self):
        print(
            f"Olá! Meu nome é {self.nome}, "
            f"tenho {self.idade} anos "
            f"e minha matrícula é {self.matricula}."
        )

aluno = Aluno("Ana", 20, "2025001")
aluno.apresentar()
```

Saída:

```text
Olá! Meu nome é Ana, tenho 20 anos e minha matrícula é 2025001.
```

O método `apresentar()` usa `self` para acessar os dados do objeto que o chamou.

### 6.1 Métodos que recebem parâmetros

Um método também pode receber dados adicionais.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome

    def estudar(self, assunto):
        print(f"{self.nome} está estudando {assunto}.")

aluno = Aluno("Ana")
aluno.estudar("Programação Orientada a Objetos")
```

Saída:

```text
Ana está estudando Programação Orientada a Objetos.
```

### 6.2 Métodos que retornam valores

Nem todo método precisa usar `print()`. Muitas vezes, é melhor retornar um valor para que outro trecho do programa possa usá-lo.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome

    def obter_mensagem(self):
        return f"Olá, eu sou {self.nome}."

aluno = Aluno("Ana")
mensagem = aluno.obter_mensagem()
print(mensagem)
```

> [!TIP]
> **Regra prática**
> - Use `print()` para exibir algo diretamente ao usuário.
> - Use `return` quando o resultado será usado por outro trecho do programa.

## 7. Praticando modelagem com classes

Modelar é decidir:

1. qual entidade será representada;
2. quais dados ela possui;
3. quais comportamentos ela executa.

Exemplo: uma classe `Livro`.

| Pergunta                       | Resposta                          |
| ------------------------------ | --------------------------------- |
| Qual entidade será representada? | Livro                           |
| Quais dados possui?            | Título, autor, disponibilidade    |
| Quais comportamentos possui?   | Emprestar e devolver              |

```python
class Livro:
    def __init__(self, titulo, autor):
        self.titulo = titulo
        self.autor = autor
        self.disponivel = True

    def emprestar(self):
        if self.disponivel:
            self.disponivel = False
            return "Livro emprestado com sucesso."
        return "O livro já está emprestado."

    def devolver(self):
        self.disponivel = True
        return "Livro devolvido com sucesso."

livro = Livro("Python para Iniciantes", "Ana Silva")
print(livro.emprestar())
print(livro.emprestar())
print(livro.devolver())
```

Esse exemplo mostra que a classe mantém o estado do objeto:

- antes do empréstimo: `disponivel = True`;
- após o empréstimo: `disponivel = False`;
- após a devolução: `disponivel = True`.

## 8. Encapsulamento: protegendo dados importantes

Encapsulamento é a prática de controlar como os atributos de um objeto podem ser acessados ou alterados.

Imagine uma conta bancária:

```python
class ContaBancaria:
    def __init__(self, titular, saldo):
        self.titular = titular
        self.saldo = saldo
```

Com essa estrutura, alguém pode alterar o saldo diretamente:

```python
conta = ContaBancaria("Ana", 100)
conta.saldo = -5000
```

> [!WARNING]
> Tal alteração pode quebrar regras do sistema.

Para evitar mudanças indevidas, usamos atributos internos e métodos que validam as operações:

```python
class ContaBancaria:
    def __init__(self, titular, saldo_inicial=0):
        self.titular = titular
        self.__saldo = saldo_inicial

    def consultar_saldo(self):
        return self.__saldo

    def depositar(self, valor):
        if valor <= 0:
            raise ValueError("O depósito deve ser maior que zero.")
        self.__saldo += valor

    def sacar(self, valor):
        if valor <= 0:
            raise ValueError("O saque deve ser maior que zero.")
        if valor > self.__saldo:
            raise ValueError("Saldo insuficiente.")
        self.__saldo -= valor
```

Uso:

```python
conta = ContaBancaria("Ana", 100)
conta.depositar(50)
conta.sacar(30)
print(conta.consultar_saldo())
```

**O que significa `__saldo`?**

O atributo `__saldo` começa com dois sublinhados. Isso indica que ele é interno à classe. O Python aplica uma técnica chamada *name mangling*, que dificulta o acesso acidental ao atributo fora da classe.

> [!NOTE]
> O objetivo não é tornar o atributo impossível de acessar, mas deixar claro que ele não deve ser alterado diretamente.

{{% details title="Convenções de acesso em Python (Clique para expandir)" closed="true" %}}

| Formato      | Significado                          |
| ------------ | ------------------------------------ |
| `atributo`   | Público: pode ser acessado livremente |
| `_atributo`  | Interno/protegido por convenção      |
| `__atributo` | Interno com ocultação de nome        |

{{% /details %}}

## 9. `@property`: encapsulamento com sintaxe simples

O *decorator* `@property` permite criar métodos que podem ser acessados como atributos.

Exemplo: um produto não pode ter preço negativo.

```python
class Produto:
    def __init__(self, nome, preco):
        self.nome = nome
        self.preco = preco

    @property
    def preco(self):
        return self.__preco

    @preco.setter
    def preco(self, novo_preco):
        if novo_preco < 0:
            raise ValueError("O preço não pode ser negativo.")
        self.__preco = novo_preco
```

Uso:

```python
produto = Produto("Livro Python", 80)
print(produto.preco)

produto.preco = 90
print(produto.preco)
```

Mesmo usando `produto.preco = 90`, o Python chama internamente o *setter*, e a validação é executada antes de alterar o valor.

**Por que não usar um método comum?**

Sem `@property`, seria necessário escrever:

```python
produto.definir_preco(90)
```

Com `@property`, a sintaxe fica mais natural:

```python
produto.preco = 90
```

Mas a proteção e as regras continuam existindo.

## 10. Atributos de classe

Até agora, os atributos pertenciam a cada objeto.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome
```

Cada aluno tem um nome próprio. Porém, alguns dados são compartilhados por todos os objetos da classe:

```python
class Aluno:
    instituicao = "Grupo de Estudos Python"

    def __init__(self, nome):
        self.nome = nome

aluno1 = Aluno("Ana")
aluno2 = Aluno("Carlos")

print(aluno1.instituicao)
print(aluno2.instituicao)
```

Saída:

```text
Grupo de Estudos Python
Grupo de Estudos Python
```

`instituicao` pertence à classe `Aluno`, enquanto `nome` pertence a cada objeto.

{{% details title="Atributo de instância x atributo de classe (Clique para expandir)" closed="true" %}}

| Tipo                  | Exemplo             | Pertence a               |
| --------------------- | ------------------- | ------------------------ |
| Atributo de instância | `self.nome`         | Um objeto específico     |
| Atributo de classe    | `Aluno.instituicao` | Todos os objetos da classe |

{{% /details %}}

## 11. Métodos de classe e métodos estáticos

Além dos métodos de instância, existem outros dois tipos importantes.

### 11.1 Métodos de classe: `@classmethod`

Métodos de classe recebem `cls`, que representa a própria classe.

```python
class Aluno:
    total_alunos = 0

    def __init__(self, nome):
        self.nome = nome
        Aluno.total_alunos += 1

    @classmethod
    def quantidade_cadastrada(cls):
        return cls.total_alunos

Aluno("Ana")
Aluno("Carlos")
print(Aluno.quantidade_cadastrada())
```

Saída:

```text
2
```

Use `@classmethod` quando o método trabalha com informações compartilhadas entre todos os objetos.

### 11.2 Métodos estáticos: `@staticmethod`

Métodos estáticos não recebem `self` nem `cls`.

```python
class Validador:
    @staticmethod
    def email_valido(email):
        return "@" in email and "." in email

print(Validador.email_valido("ana@email.com"))
print(Validador.email_valido("email_invalido"))
```

Um método estático é usado quando a função possui relação lógica com a classe, mas não precisa acessar dados de um objeto ou da própria classe.

## 12. Herança: reaproveitando características

Herança permite criar uma classe mais específica a partir de outra mais geral.

Em um sistema escolar, alunos e professores possuem dados em comum:

- nome;
- idade;
- apresentação.

Sem herança, haveria repetição:

```python
class Aluno:
    def __init__(self, nome, idade, matricula):
        self.nome = nome
        self.idade = idade
        self.matricula = matricula

class Professor:
    def __init__(self, nome, idade, disciplina):
        self.nome = nome
        self.idade = idade
        self.disciplina = disciplina
```

Podemos criar uma classe geral chamada `Pessoa`:

```python
class Pessoa:
    def __init__(self, nome, idade):
        self.nome = nome
        self.idade = idade

    def apresentar(self):
        return f"Olá, meu nome é {self.nome}."
```

Agora, `Aluno` herda de `Pessoa`:

```python
class Aluno(Pessoa):
    def __init__(self, nome, idade, matricula):
        super().__init__(nome, idade)
        self.matricula = matricula

    def estudar(self, assunto):
        print(f"{self.nome} está estudando {assunto}.")

aluno = Aluno("Ana", 20, "2025001")
print(aluno.apresentar())
aluno.estudar("Herança")
```

> [!NOTE]
> **O que faz `super()`?**
> `super().__init__(nome, idade)` chama métodos da classe pai. Nesse caso, chama o construtor de `Pessoa`, que define `self.nome = nome` e `self.idade = idade`. Isso evita repetição de código.

**Quando usar herança?**

Use herança quando uma classe possui uma relação clara de "é um":

- um aluno é uma pessoa;
- um professor é uma pessoa;
- um cachorro é um animal.

> [!WARNING]
> Evite usar herança apenas porque duas classes possuem algo parecido. Às vezes, a composição é mais adequada. Por exemplo: uma turma **possui** alunos, mas uma turma **não é** um aluno.

## 13. Sobrescrita de métodos

Uma classe filha pode alterar um método que herdou da classe pai.

```python
class Pessoa:
    def __init__(self, nome):
        self.nome = nome

    def apresentar(self):
        return f"Sou {self.nome}."

class Aluno(Pessoa):
    def apresentar(self):
        return f"Sou o aluno {self.nome}."

class Professor(Pessoa):
    def apresentar(self):
        return f"Sou o professor {self.nome}."

aluno = Aluno("Ana")
professor = Professor("Marcos")

print(aluno.apresentar())
print(professor.apresentar())
```

Saída:

```text
Sou o aluno Ana.
Sou o professor Marcos.
```

A sobrescrita permite aproveitar uma estrutura comum, mas adaptar comportamentos específicos.

## 14. Polimorfismo

Polimorfismo significa que objetos de classes diferentes podem responder ao mesmo método de maneiras diferentes.

Usando as classes anteriores:

```python
pessoas = [
    Aluno("Ana"),
    Professor("Marcos"),
    Pessoa("Carla")
]

for pessoa in pessoas:
    print(pessoa.apresentar())
```

Saída:

```text
Sou o aluno Ana.
Sou o professor Marcos.
Sou Carla.
```

O laço não precisa saber se cada objeto é aluno, professor ou pessoa. Ele apenas sabe que cada elemento possui um método chamado `apresentar()`.

> [!IMPORTANT]
> Essa é a principal vantagem do polimorfismo: o código pode trabalhar com comportamentos comuns, sem depender da classe exata de cada objeto.

## 15. Pacotes e módulos

Quando um projeto possui muitas classes, não é adequado manter tudo em um único arquivo.

### 15.1 Módulos

Um módulo é um arquivo Python.

```python {filename="aluno.py"}
class Aluno:
    pass
```

Podemos importar a classe em outro arquivo:

```python
from aluno import Aluno
```

### 15.2 Pacotes

Um pacote é uma pasta que agrupa módulos relacionados.

Estrutura sugerida:

```text
sistema_escolar/
├── main.py
└── escola/
    ├── __init__.py
    ├── pessoa.py
    ├── aluno.py
    └── professor.py
```

- `escola/` é o pacote;
- `pessoa.py`, `aluno.py` e `professor.py` são módulos;
- `main.py` executa o programa.

### 15.3 Por que usar `__init__.py`?

Tradicionalmente, o arquivo `__init__.py` informa ao Python que a pasta é um pacote. Em versões modernas do Python, uma pasta pode funcionar como pacote mesmo sem esse arquivo. Ainda assim, é recomendado utilizá-lo em projetos comuns porque:

- deixa explícito que a pasta é um pacote;
- permite configurar importações;
- pode armazenar informações do pacote;
- melhora a compatibilidade com ferramentas e projetos existentes.

Inicialmente, ele pode estar vazio:

```python {filename="escola/__init__.py"}
```

### 15.4 O que colocar no `__init__.py`?

Uma possibilidade é facilitar importações:

```python {filename="escola/__init__.py"}
from .aluno import Aluno
from .professor import Professor
```

O ponto em `.aluno` significa que o módulo está dentro do mesmo pacote. Sem configurar o `__init__.py`:

```python
from escola.aluno import Aluno
```

Com a configuração:

```python
from escola import Aluno
```

Também é possível guardar uma versão:

```python {filename="escola/__init__.py"}
__version__ = "1.0.0"
```

> [!WARNING]
> Evite colocar lógicas pesadas no `__init__.py`, como conexões com banco de dados, leitura extensa de arquivos ou execução de código principal. Ele deve ser simples, pois pode ser executado ao importar o pacote.

### 15.5 Exemplo de projeto organizado

Arquivo `escola/pessoa.py`:

```python {filename="escola/pessoa.py"}
class Pessoa:
    def __init__(self, nome, idade):
        self.nome = nome
        self.idade = idade

    def apresentar(self):
        return f"Meu nome é {self.nome}."
```

Arquivo `escola/aluno.py`:

```python {filename="escola/aluno.py"}
from .pessoa import Pessoa

class Aluno(Pessoa):
    def __init__(self, nome, idade, matricula):
        super().__init__(nome, idade)
        self.matricula = matricula

    def estudar(self, assunto):
        return f"{self.nome} está estudando {assunto}."
```

Arquivo `main.py`:

```python {filename="main.py"}
from escola.aluno import Aluno

aluno = Aluno("Ana", 20, "2025001")
print(aluno.apresentar())
print(aluno.estudar("POO com Python"))
```

## 16. Decorators

*Decorators* são funções que alteram ou adicionam comportamentos a outras funções ou métodos sem modificar diretamente seu código. Eles são úteis para comportamentos que se repetem, como:

- registrar *logs*;
- verificar permissões;
- medir tempo de execução;
- validar acessos;
- salvar resultados em *cache*;
- registrar auditorias.

### 16.1 A ideia antes da sintaxe

Considere uma função comum:

```python
def gerar_relatorio():
    print("Relatório gerado.")
```

Se quisermos registrar sempre que uma função começa e termina, poderíamos repetir o código:

```python
def gerar_relatorio():
    print("Iniciando execução...")
    print("Relatório gerado.")
    print("Execução finalizada.")
```

Mas repetir esse comportamento em todas as funções gera duplicação. Um *decorator* permite centralizar essa regra.

### 16.2 Primeiro decorator

```python
def registrar_execucao(funcao):
    def interna():
        print("Iniciando execução...")
        funcao()
        print("Execução finalizada.")
    return interna
```

Aplicação:

```python
@registrar_execucao
def gerar_relatorio():
    print("Relatório gerado.")

gerar_relatorio()
```

Saída:

```text
Iniciando execução...
Relatório gerado.
Execução finalizada.
```

A sintaxe `@registrar_execucao` é equivalente a:

```python
gerar_relatorio = registrar_execucao(gerar_relatorio)
```

Ou seja, o *decorator* recebe a função original e devolve uma nova função com comportamentos adicionais.

### 16.3 Decorators com argumentos

Funções e métodos normalmente recebem argumentos. Para criar um *decorator* reutilizável, usamos:

- `*args`: argumentos posicionais;
- `**kwargs`: argumentos nomeados.

```python
from functools import wraps

def registrar_execucao(funcao):
    @wraps(funcao)
    def interna(*args, **kwargs):
        print(f"Executando: {funcao.__name__}")
        resultado = funcao(*args, **kwargs)
        print("Execução finalizada.")
        return resultado
    return interna

@registrar_execucao
def somar(numero1, numero2):
    return numero1 + numero2

resultado = somar(10, 5)
print(resultado)
```

Saída:

```text
Executando: somar
Execução finalizada.
15
```

### 16.4 Por que usar `@wraps`?

`@wraps(funcao)` preserva informações da função original, como:

- nome;
- documentação;
- metadados.

Sem `@wraps`, o Python pode identificar a função decorada pelo nome da função interna, `interna`, em vez de `somar`.

### 16.5 Decorators em métodos

*Decorators* também podem ser aplicados a métodos de classes.

```python
from functools import wraps

def registrar_execucao(funcao):
    @wraps(funcao)
    def interna(*args, **kwargs):
        print(f"Executando o método: {funcao.__name__}")
        return funcao(*args, **kwargs)
    return interna

class Aluno:
    def __init__(self, nome):
        self.nome = nome

    @registrar_execucao
    def estudar(self, assunto):
        print(f"{self.nome} está estudando {assunto}.")

aluno = Aluno("Ana")
aluno.estudar("Decorators")
```

Saída:

```text
Executando o método: estudar
Ana está estudando Decorators.
```

## 17. Exemplo completo: sistema de alunos

Este exemplo integra construtor, encapsulamento, herança, polimorfismo, atributos de classe, métodos de classe e *decorators*.

```python
from functools import wraps

def registrar_execucao(funcao):
    @wraps(funcao)
    def interna(*args, **kwargs):
        print(f"Executando: {funcao.__name__}")
        return funcao(*args, **kwargs)
    return interna

class Pessoa:
    def __init__(self, nome, idade):
        self.nome = nome
        self.idade = idade

    def apresentar(self):
        return f"Meu nome é {self.nome} e tenho {self.idade} anos."

class Aluno(Pessoa):
    total_alunos = 0

    def __init__(self, nome, idade, matricula):
        super().__init__(nome, idade)
        self.matricula = matricula
        self.__notas = []
        Aluno.total_alunos += 1

    @property
    def notas(self):
        return self.__notas.copy()

    @registrar_execucao
    def adicionar_nota(self, nota):
        if not 0 <= nota <= 10:
            raise ValueError("A nota deve estar entre 0 e 10.")
        self.__notas.append(nota)

    def calcular_media(self):
        if not self.__notas:
            return 0
        return sum(self.__notas) / len(self.__notas)

    def apresentar(self):
        return (
            f"Sou o aluno {self.nome}, tenho {self.idade} anos "
            f"e minha matrícula é {self.matricula}."
        )

    @classmethod
    def quantidade_cadastrada(cls):
        return cls.total_alunos

class Professor(Pessoa):
    def __init__(self, nome, idade, disciplina):
        super().__init__(nome, idade)
        self.disciplina = disciplina

    def apresentar(self):
        return (
            f"Sou o professor {self.nome}, tenho {self.idade} anos "
            f"e ministro a disciplina {self.disciplina}."
        )
```

Uso:

```python
aluno = Aluno("Ana", 20, "2025001")
professor = Professor("Marcos", 35, "Python")

aluno.adicionar_nota(8.5)
aluno.adicionar_nota(9.0)

print(aluno.apresentar())
print(f"Notas: {aluno.notas}")
print(f"Média: {aluno.calcular_media():.2f}")
print(professor.apresentar())

pessoas = [aluno, professor]
for pessoa in pessoas:
    print(pessoa.apresentar())

print(f"Total de alunos: {Aluno.quantidade_cadastrada()}")
```

Saída:

```text
Executando: adicionar_nota
Executando: adicionar_nota
Sou o aluno Ana, tenho 20 anos e minha matrícula é 2025001.
Notas: [8.5, 9.0]
Média: 8.75
Sou o professor Marcos, tenho 35 anos e ministro a disciplina Python.
Sou o aluno Ana, tenho 20 anos e minha matrícula é 2025001.
Sou o professor Marcos, tenho 35 anos e ministro a disciplina Python.
Total de alunos: 1
```

## 18. Erros comuns

### Confundir classe com objeto

```python
class Aluno:
    pass
```

`Aluno` é uma classe.

```python
aluno = Aluno()
```

`aluno` é um objeto.

### Esquecer o `self`

Errado:

```python
class Aluno:
    def apresentar():
        print("Olá")
```

Correto:

```python
class Aluno:
    def apresentar(self):
        print("Olá")
```

Métodos de instância precisam receber `self` como primeiro parâmetro.

### Usar atributos de classe quando deveriam ser de instância

Errado:

```python
class Aluno:
    notas = []
```

Nesse caso, a mesma lista poderá ser compartilhada por todos os alunos.

Correto:

```python
class Aluno:
    def __init__(self):
        self.notas = []
```

Agora, cada aluno possui sua própria lista.

### Alterar atributos internos diretamente

Evite:

```python
conta.__saldo = -100
```

Prefira chamar métodos que respeitam as regras:

```python
conta.sacar(100)
```

### Usar herança sem uma relação "é um"

Evite modelar uma turma como herdeira de aluno:

```python
class Turma(Aluno):
    pass
```

Uma turma possui alunos, mas não é um aluno. Uma modelagem mais adequada seria:

```python
class Turma:
    def __init__(self, nome):
        self.nome = nome
        self.alunos = []

    def adicionar_aluno(self, aluno):
        self.alunos.append(aluno)
```

## 19. Exercícios sugeridos

### Exercício 1 — Classe `Produto`

Crie uma classe `Produto` com:

- nome;
- preço;
- estoque.

Crie os métodos:

- `exibir_dados()`;
- `aplicar_desconto(percentual)`;
- `repor_estoque(quantidade)`.

### Exercício 2 — Classe `ContaBancaria`

Crie uma classe com:

- titular;
- saldo encapsulado.

Crie os métodos:

- `depositar(valor)`;
- `sacar(valor)`;
- `consultar_saldo()`.

Regras:

- depósitos devem ser maiores que zero;
- saques devem ser maiores que zero;
- o saldo não pode ficar negativo.

### Exercício 3 — Herança

Crie uma classe `Veiculo` com:

- marca;
- modelo;
- método `ligar()`.

Crie as subclasses:

- `Carro`;
- `Moto`.

Cada subclasse deve sobrescrever o método `ligar()` com uma mensagem própria.

### Exercício 4 — Polimorfismo

Crie uma lista com objetos `Carro` e `Moto`. Percorra a lista e chame `ligar()` para cada objeto, sem usar `if` para verificar o tipo do veículo.

### Exercício 5 — Pacote

Organize as classes `Pessoa`, `Aluno` e `Professor` em um pacote chamado `escola`.

Estrutura esperada:

```text
projeto/
├── main.py
└── escola/
    ├── __init__.py
    ├── pessoa.py
    ├── aluno.py
    └── professor.py
```

## 20. Resumo final

| Conceito       | O que é                                  | Por que usar                                        |
| -------------- | ---------------------------------------- | --------------------------------------------------- |
| Classe         | Molde para criar objetos                 | Representar entidades do sistema                    |
| Objeto         | Instância de uma classe                  | Representar elementos concretos                     |
| Atributo       | Dado de um objeto ou classe              | Armazenar características                           |
| Construtor     | Método `__init__`                        | Criar objetos completos e válidos                   |
| `self`         | Referência ao objeto atual               | Acessar dados e métodos do próprio objeto           |
| Método         | Função dentro de uma classe              | Definir comportamentos                              |
| Encapsulamento | Controle de acesso aos dados             | Proteger regras e evitar estados inválidos          |
| `@property`    | Controle de atributos por métodos        | Validar leitura e alteração de dados                |
| Herança        | Criação de classes especializadas        | Reutilizar características comuns                   |
| Polimorfismo   | Mesmo método com comportamentos diferentes | Trabalhar com objetos de forma flexível           |
| Pacote         | Pasta que agrupa módulos                 | Organizar projetos maiores                          |
| `__init__.py`  | Arquivo de configuração do pacote        | Tornar o pacote explícito e simplificar importações |
| Decorator      | Função que modifica outra função ou método | Reutilizar comportamentos transversais            |
