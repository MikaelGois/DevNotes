---
title: A general guide to object-oriented programming with Python
type: docs
weight: 1
editURL: "https://devnotes.msglabs.com.br/articles/python-oop/"
next: /articles/2026/07/1-ansible
---

Object-oriented programming (OOP) is the paradigm behind most modern languages, such as Java, C# and Python. Instead of organizing the code into loose variables and scattered functions, OOP groups data and behavior around real-world entities, represented by objects.

This guide was organized so that the concepts appear in the order that makes the most sense. The idea is to start with the problem that OOP solves, create simple objects and gradually advance to organizing code into packages and *decorators*.

A small school system will be used as the main example.

> [!NOTE]
> To follow this article, it is important to master the topics listed in section [0. Prerequisites](#0-prerequisites). If you already master them, you can go straight to section [1. What problem does OOP solve?](#1-what-problem-does-oop-solve).

## 0. Prerequisites

Before studying OOP, it is important to master:

- variables;
- basic types: `str`, `int`, `float`, `bool`;
- conditionals: `if`, `elif` and `else`;
- loops: `for` and `while`;
- functions;
- lists and dictionaries;
- basic imports.

Example of a regular function:

```python
def apresentar_nome(nome):
    print(f"Olá, {nome}!")

apresentar_nome("Ana")
```

In OOP, functions related to the same entity are organized inside classes.

## 1. What problem does OOP solve?

Consider a system that registers students.

Without OOP, we could create loose variables:

```python
nome_aluno_1 = "Ana"
idade_aluno_1 = 20
matricula_aluno_1 = "2025001"

nome_aluno_2 = "Carlos"
idade_aluno_2 = 22
matricula_aluno_2 = "2025002"
```

We would also need functions that receive many arguments:

```python
def apresentar_aluno(nome, idade, matricula):
    print(f"Aluno: {nome}")
    print(f"Idade: {idade}")
    print(f"Matrícula: {matricula}")
```

This model becomes confusing as the system grows, because the related data and functions end up scattered.

OOP solves this by representing real-world entities as objects. A student has:

- data: name, age and enrollment number;
- behavior: introducing themselves, studying, receiving a grade.

> [!IMPORTANT]
> The central idea is: **a class groups data and behavior that belong to the same entity.**

## 2. Classes and objects

### 2.1 Class: the template

A class is a definition, a model or a blueprint for creating objects.

```python
class Aluno:
    pass
```

The class above states that the concept of Aluno (Student) exists, but there is no specific student yet.

Think of it this way:

| Concept  | Comparison               |
| -------- | ------------------------ |
| Class    | Blueprint of a house     |
| Object   | Built house              |
| Class    | Recipe                   |
| Object   | Prepared dish            |
| Class    | Student record template  |
| Object   | Ana's record             |

### 2.2 Object: an instance of the class

An object is a concrete occurrence created from a class.

```python
class Aluno:
    pass

aluno1 = Aluno()
aluno2 = Aluno()
```

In this code:

- `Aluno` is the class;
- `aluno1` is an object;
- `aluno2` is another object.

Creating an object from a class is called **instantiation**.

Even though they are created from the same class, objects are independent:

```python
aluno1.nome = "Ana"
aluno2.nome = "Carlos"

print(aluno1.nome)
print(aluno2.nome)
```

Output:

```text
Ana
Carlos
```

## 3. Attributes: an object's data

Attributes are characteristics stored in objects.

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

Although this works, there is a problem: the attributes are created manually.

```python
aluno1 = Aluno()
aluno1.nome = "Ana"

aluno2 = Aluno()
aluno2.idade = 20
```

In this case, `aluno2` has no name and `aluno1` has no age. This can cause errors and incomplete objects. The solution is to use a constructor.

## 4. Constructor: `__init__`

The constructor is the special `__init__` method. It runs automatically when an object is created.

```python
class Aluno:
    def __init__(self, nome, idade, matricula):
        self.nome = nome
        self.idade = idade
        self.matricula = matricula
```

Now, to create a student, all required data must be provided:

```python
aluno = Aluno("Ana", 20, "2025001")

print(aluno.nome)
print(aluno.idade)
print(aluno.matricula)
```

**Why use constructors?**

The constructor ensures that objects are created with the required data. Without a constructor, an object can be left incomplete:

```python
aluno = Aluno()
```

With a constructor:

```python
aluno = Aluno("Ana", 20, "2025001")
```

The object is born with a valid structure.

### 4.1 Default values

Not every piece of data needs to be required.

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

Output:

```text
True
```

It is also possible to provide the value:

```python
aluno_inativo = Aluno("Carlos", 22, "2025002", False)
```

## 5. Understanding `self`

`self` represents the object itself at the moment it is being used.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome
```

When running:

```python
aluno1 = Aluno("Ana")
aluno2 = Aluno("Carlos")
```

Python works, in a simplified way, as if it were:

```python
Aluno.__init__(aluno1, "Ana")
Aluno.__init__(aluno2, "Carlos")
```

Therefore:

- when creating `aluno1`, `self` represents `aluno1`;
- when creating `aluno2`, `self` represents `aluno2`.

Notice:

```python
self.nome = nome
```

- `nome` is the received parameter;
- `self.nome` is the attribute stored in the object.

Without `self`, the value would exist only during the method's execution:

```python
class Aluno:
    def __init__(self, nome):
        nome = nome
```

This code does not store `nome` inside the object.

> [!NOTE]
> The convention is to always call the first parameter of instance methods `self`.

## 6. Methods: an object's behavior

Methods are functions created inside a class. They represent the actions an object can perform.

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

Output:

```text
Olá! Meu nome é Ana, tenho 20 anos e minha matrícula é 2025001.
```

The `apresentar()` method uses `self` to access the data of the object that called it.

### 6.1 Methods that take parameters

A method can also receive additional data.

> [!NOTE]
> Throughout this English translation, the code examples intentionally retain Portuguese identifiers and user-facing messages from the original article. The sample outputs reproduce those messages as printed.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome

    def estudar(self, assunto):
        print(f"{self.nome} está estudando {assunto}.")

aluno = Aluno("Ana")
aluno.estudar("Programação Orientada a Objetos")
```

Output:

```text
Ana está estudando Programação Orientada a Objetos.
```

### 6.2 Methods that return values

Not every method needs to use `print()`. Often, it is better to return a value so that another part of the program can use it.

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
> **Rule of thumb**
> - Use `print()` to display something directly to the user.
> - Use `return` when the result will be used by another part of the program.

## 7. Practicing modeling with classes

Modeling means deciding:

1. which entity will be represented;
2. what data it has;
3. what behavior it performs.

Example: a `Livro` (Book) class.

| Question                       | Answer                            |
| ------------------------------ | --------------------------------- |
| Which entity will be represented? | Book                            |
| What data does it have?        | Title, author, availability       |
| What behavior does it have?    | Lending and returning             |

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

This example shows that the class keeps the object's state:

- before lending: `disponivel = True`;
- after lending: `disponivel = False`;
- after returning: `disponivel = True`.

## 8. Encapsulation: protecting important data

Encapsulation is the practice of controlling how an object's attributes can be accessed or changed.

Imagine a bank account:

```python
class ContaBancaria:
    def __init__(self, titular, saldo):
        self.titular = titular
        self.saldo = saldo
```

With this structure, anyone can change the balance directly:

```python
conta = ContaBancaria("Ana", 100)
conta.saldo = -5000
```

> [!WARNING]
> Such a change can break the system's rules.

To prevent improper changes, we use internal attributes and methods that validate operations:

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

Usage:

```python
conta = ContaBancaria("Ana", 100)
conta.depositar(50)
conta.sacar(30)
print(conta.consultar_saldo())
```

**What does `__saldo` mean?**

The `__saldo` attribute starts with two underscores. This indicates that it is internal to the class. Python applies a technique called *name mangling*, which makes accidental access to the attribute from outside the class harder.

> [!NOTE]
> The goal is not to make the attribute impossible to access, but to make it clear that it should not be changed directly.

{{% details title="Access conventions in Python (Click to expand)" closed="true" %}}

| Format      | Meaning                              |
| ----------- | ------------------------------------ |
| `attribute` | Public: can be accessed freely       |
| `_attribute`  | Internal/protected by convention   |
| `__attribute` | Internal with name hiding          |

{{% /details %}}

## 9. `@property`: encapsulation with simple syntax

The `@property` decorator allows creating methods that can be accessed as attributes.

Example: a product cannot have a negative price.

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

Usage:

```python
produto = Produto("Livro Python", 80)
print(produto.preco)

produto.preco = 90
print(produto.preco)
```

Even when using `produto.preco = 90`, Python internally calls the *setter*, and the validation runs before the value is changed.

**Why not use a regular method?**

Without `@property`, you would have to write:

```python
produto.definir_preco(90)
```

With `@property`, the syntax is more natural:

```python
produto.preco = 90
```

But the protection and the rules still exist.

## 10. Class attributes

So far, attributes belonged to each object.

```python
class Aluno:
    def __init__(self, nome):
        self.nome = nome
```

Each student has their own name. However, some data is shared by all objects of the class:

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

Output:

```text
Grupo de Estudos Python
Grupo de Estudos Python
```

`instituicao` belongs to the `Aluno` class, while `nome` belongs to each object.

{{% details title="Instance attribute x class attribute (Click to expand)" closed="true" %}}

| Type                  | Example             | Belongs to             |
| --------------------- | ------------------- | ---------------------- |
| Instance attribute    | `self.nome`         | A specific object      |
| Class attribute       | `Aluno.instituicao` | All objects of the class |

{{% /details %}}

## 11. Class methods and static methods

Besides instance methods, there are two other important types.

### 11.1 Class methods: `@classmethod`

Class methods receive `cls`, which represents the class itself.

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

Output:

```text
2
```

Use `@classmethod` when the method works with information shared among all objects.

### 11.2 Static methods: `@staticmethod`

Static methods receive neither `self` nor `cls`.

```python
class Validador:
    @staticmethod
    def email_valido(email):
        return "@" in email and "." in email

print(Validador.email_valido("ana@email.com"))
print(Validador.email_valido("email_invalido"))
```

A static method is used when the function has a logical relationship with the class, but does not need to access data from an object or from the class itself.

## 12. Inheritance: reusing characteristics

Inheritance allows creating a more specific class from a more general one.

In a school system, students and teachers have things in common:

- name;
- age;
- introduction.

Without inheritance, there would be repetition:

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

We can create a general class called `Pessoa` (Person):

```python
class Pessoa:
    def __init__(self, nome, idade):
        self.nome = nome
        self.idade = idade

    def apresentar(self):
        return f"Olá, meu nome é {self.nome}."
```

Now, `Aluno` inherits from `Pessoa`:

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
> **What does `super()` do?**
> `super().__init__(nome, idade)` calls methods of the parent class. In this case, it calls `Pessoa`'s constructor, which sets `self.nome = nome` and `self.idade = idade`. This avoids code repetition.

**When to use inheritance?**

Use inheritance when a class has a clear "is a" relationship:

- a student is a person;
- a teacher is a person;
- a dog is an animal.

> [!WARNING]
> Avoid using inheritance just because two classes have something similar. Sometimes composition is more suitable. For example: a class group *has* students, but a class group *is not* a student.

## 13. Method overriding

A child class can change a method it inherited from the parent class.

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

Output:

```text
Sou o aluno Ana.
Sou o professor Marcos.
```

Overriding allows reusing a common structure while adapting specific behaviors.

## 14. Polymorphism

Polymorphism means that objects from different classes can respond to the same method in different ways.

Using the previous classes:

```python
pessoas = [
    Aluno("Ana"),
    Professor("Marcos"),
    Pessoa("Carla")
]

for pessoa in pessoas:
    print(pessoa.apresentar())
```

Output:

```text
Sou o aluno Ana.
Sou o professor Marcos.
Sou Carla.
```

The loop does not need to know whether each object is a student, a teacher or a person. It only knows that each element has a method called `apresentar()`.

> [!IMPORTANT]
> This is the main advantage of polymorphism: the code can work with common behavior without depending on the exact class of each object.

## 15. Packages and modules

When a project has many classes, keeping everything in a single file is not appropriate.

### 15.1 Modules

A module is a Python file.

```python {filename="aluno.py"}
class Aluno:
    pass
```

We can import the class in another file:

```python
from aluno import Aluno
```

### 15.2 Packages

A package is a folder that groups related modules.

Suggested structure:

```text
sistema_escolar/
├── main.py
└── escola/
    ├── __init__.py
    ├── pessoa.py
    ├── aluno.py
    └── professor.py
```

- `escola/` is the package;
- `pessoa.py`, `aluno.py` and `professor.py` are modules;
- `main.py` runs the program.

### 15.3 Why use `__init__.py`?

Traditionally, the `__init__.py` file tells Python that the folder is a package. In modern versions of Python, a folder can work as a package even without this file. Still, it is recommended in common projects because it:

- makes it explicit that the folder is a package;
- allows configuring imports;
- can store package information;
- improves compatibility with existing tools and projects.

Initially, it can be empty:

```python {filename="escola/__init__.py"}
```

### 15.4 What to put in `__init__.py`?

One possibility is to make imports easier:

```python {filename="escola/__init__.py"}
from .aluno import Aluno
from .professor import Professor
```

The dot in `.aluno` means the module is inside the same package. Without configuring `__init__.py`:

```python
from escola.aluno import Aluno
```

With the configuration:

```python
from escola import Aluno
```

It is also possible to store a version:

```python {filename="escola/__init__.py"}
__version__ = "1.0.0"
```

> [!WARNING]
> Avoid putting heavy logic in `__init__.py`, such as database connections, extensive file reading or running main code. It should be simple, as it may run when the package is imported.

### 15.5 Example of an organized project

File `escola/pessoa.py`:

```python {filename="escola/pessoa.py"}
class Pessoa:
    def __init__(self, nome, idade):
        self.nome = nome
        self.idade = idade

    def apresentar(self):
        return f"Meu nome é {self.nome}."
```

File `escola/aluno.py`:

```python {filename="escola/aluno.py"}
from .pessoa import Pessoa

class Aluno(Pessoa):
    def __init__(self, nome, idade, matricula):
        super().__init__(nome, idade)
        self.matricula = matricula

    def estudar(self, assunto):
        return f"{self.nome} está estudando {assunto}."
```

File `main.py`:

```python {filename="main.py"}
from escola.aluno import Aluno

aluno = Aluno("Ana", 20, "2025001")
print(aluno.apresentar())
print(aluno.estudar("POO com Python"))
```

## 16. Decorators

*Decorators* are functions that change or add behavior to other functions or methods without directly modifying their code. They are useful for behavior that repeats, such as:

- recording *logs*;
- checking permissions;
- measuring execution time;
- validating access;
- caching results;
- recording audits.

### 16.1 The idea before the syntax

Consider a regular function:

```python
def gerar_relatorio():
    print("Relatório gerado.")
```

If we want to log whenever a function starts and ends, we could repeat the code:

```python
def gerar_relatorio():
    print("Iniciando execução...")
    print("Relatório gerado.")
    print("Execução finalizada.")
```

But repeating this behavior in every function causes duplication. A *decorator* allows centralizing this rule.

### 16.2 First decorator

```python
def registrar_execucao(funcao):
    def interna():
        print("Iniciando execução...")
        funcao()
        print("Execução finalizada.")
    return interna
```

Application:

```python
@registrar_execucao
def gerar_relatorio():
    print("Relatório gerado.")

gerar_relatorio()
```

Output:

```text
Iniciando execução...
Relatório gerado.
Execução finalizada.
```

The syntax `@registrar_execucao` is equivalent to:

```python
gerar_relatorio = registrar_execucao(gerar_relatorio)
```

That is, the *decorator* receives the original function and returns a new function with additional behavior.

### 16.3 Decorators with arguments

Functions and methods usually take arguments. To create a reusable *decorator*, we use:

- `*args`: positional arguments;
- `**kwargs`: named arguments.

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

Output:

```text
Executando: somar
Execução finalizada.
15
```

### 16.4 Why use `@wraps`?

`@wraps(funcao)` preserves information about the original function, such as:

- name;
- documentation;
- metadata.

Without `@wraps`, Python may identify the decorated function by the name of the inner function, `interna`, instead of `somar`.

### 16.5 Decorators in methods

*Decorators* can also be applied to class methods.

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

Output:

```text
Executando o método: estudar
Ana está estudando Decorators.
```

## 17. Complete example: student system

This example brings together constructor, encapsulation, inheritance, polymorphism, class attributes, class methods and *decorators*.

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

Usage:

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

Output:

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

## 18. Common mistakes

### Confusing class with object

```python
class Aluno:
    pass
```

`Aluno` is a class.

```python
aluno = Aluno()
```

`aluno` is an object.

### Forgetting `self`

Wrong:

```python
class Aluno:
    def apresentar():
        print("Olá")
```

Correct:

```python
class Aluno:
    def apresentar(self):
        print("Olá")
```

Instance methods must take `self` as their first parameter.

### Using class attributes when they should be instance attributes

Wrong:

```python
class Aluno:
    notas = []
```

In this case, the same list may be shared by every student.

Correct:

```python
class Aluno:
    def __init__(self):
        self.notas = []
```

Now, each student has their own list.

### Changing internal attributes directly

Avoid:

```python
conta.__saldo = -100
```

Prefer calling methods that respect the rules:

```python
conta.sacar(100)
```

### Using inheritance without an "is a" relationship

Avoid modeling a class group as a subclass of student:

```python
class Turma(Aluno):
    pass
```

A class group has students, but a class group is not a student. A more appropriate model would be:

```python
class Turma:
    def __init__(self, nome):
        self.nome = nome
        self.alunos = []

    def adicionar_aluno(self, aluno):
        self.alunos.append(aluno)
```

## 19. Suggested exercises

### Exercise 1 — `Produto` class

Create a `Produto` (Product) class with:

- name;
- price;
- stock.

Create the methods:

- `exibir_dados()`;
- `aplicar_desconto(percentual)`;
- `repor_estoque(quantidade)`.

### Exercise 2 — `ContaBancaria` class

Create a class with:

- holder;
- encapsulated balance.

Create the methods:

- `depositar(valor)`;
- `sacar(valor)`;
- `consultar_saldo()`.

Rules:

- deposits must be greater than zero;
- withdrawals must be greater than zero;
- the balance cannot become negative.

### Exercise 3 — Inheritance

Create a `Veiculo` (Vehicle) class with:

- brand;
- model;
- `ligar()` method.

Create the subclasses:

- `Carro` (Car);
- `Moto` (Motorcycle).

Each subclass must override the `ligar()` method with its own message.

### Exercise 4 — Polymorphism

Create a list with `Carro` and `Moto` objects. Iterate over the list and call `ligar()` on each object, without using `if` to check the type of the vehicle.

### Exercise 5 — Package

Organize the `Pessoa`, `Aluno` and `Professor` classes into a package named `escola`.

Expected structure:

```text
projeto/
├── main.py
└── escola/
    ├── __init__.py
    ├── pessoa.py
    ├── aluno.py
    └── professor.py
```

## 20. Final summary

| Concept      | What it is                                | Why use it                                    |
| ------------ | ----------------------------------------- | --------------------------------------------- |
| Class        | Template for creating objects             | Represent the system's entities               |
| Object       | Instance of a class                       | Represent concrete elements                   |
| Attribute    | Data of an object or class                | Store characteristics                         |
| Constructor  | The `__init__` method                     | Create complete, valid objects                |
| `self`       | Reference to the current object           | Access the object's own data and methods      |
| Method       | Function inside a class                   | Define behavior                               |
| Encapsulation | Control over access to data              | Protect rules and prevent invalid states      |
| `@property`  | Attribute control through methods         | Validate reading and changing data            |
| Inheritance  | Creation of specialized classes           | Reuse common characteristics                  |
| Polymorphism | Same method with different behaviors      | Work with objects flexibly                    |
| Package      | Folder that groups modules                | Organize larger projects                      |
| `__init__.py`  | Package configuration file              | Make the package explicit and simplify imports |
| Decorator    | Function that modifies another function or method | Reuse cross-cutting behavior          |
