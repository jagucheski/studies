# SOLID em Java — exemplos práticos

---

## S — Single Responsibility Principle
> Uma classe deve ter apenas um motivo para mudar

```java
// ❌ Errado — classe faz tudo
public class Funcionario {
    public void calcularSalario() { ... }
    public void salvarNoBanco() { ... }           // responsabilidade de persistência
    public void enviarEmailContracheque() { ... } // responsabilidade de email
}

// ✅ Correto — cada classe tem uma responsabilidade
public class Funcionario {
    private String nome;
    private double salario;

    public double calcularSalario() { return salario; }
}

public class FuncionarioRepository {
    public void salvar(Funcionario f) { /* persiste no banco */ }
}

public class ContrachequeEmailService {
    public void enviar(Funcionario f) { /* envia email */ }
}
```

---

## O — Open/Closed Principle
> Aberto para extensão, fechado para modificação

```java
// ❌ Errado — precisa modificar a classe para cada novo desconto
public class CalculadoraDesconto {
    public double calcular(Funcionario f) {
        if (f.getCargo().equals("JUNIOR")) return f.getSalario() * 0.05;
        if (f.getCargo().equals("SENIOR")) return f.getSalario() * 0.10;
        // se vier um novo cargo, preciso modificar aqui ❌
        return 0;
    }
}

// ✅ Correto — extende sem modificar
public interface RegraDesconto {
    double calcular(Funcionario f);
}

public class DescontoJunior implements RegraDesconto {
    public double calcular(Funcionario f) {
        return f.getSalario() * 0.05;
    }
}

public class DescontoSenior implements RegraDesconto {
    public double calcular(Funcionario f) {
        return f.getSalario() * 0.10;
    }
}

// novo cargo = nova classe, sem tocar no código existente ✅
public class DescontoEstagiario implements RegraDesconto {
    public double calcular(Funcionario f) {
        return f.getSalario() * 0.02;
    }
}

public class CalculadoraDesconto {
    public double calcular(Funcionario f, RegraDesconto regra) {
        return regra.calcular(f);
    }
}
```

---

## L — Liskov Substitution Principle
> Subclasses devem poder substituir a classe pai sem quebrar o sistema

```java
// ❌ Errado — Estagiario quebra o contrato de Funcionario
public class Funcionario {
    public double calcularHoraExtra() {
        return getSalario() * 0.5;
    }
}

public class Estagiario extends Funcionario {
    @Override
    public double calcularHoraExtra() {
        throw new UnsupportedOperationException("Estagiário não tem hora extra!");
        // quebra o contrato — quem usa Funcionario não espera uma exceção ❌
    }
}

// ✅ Correto — separa o contrato corretamente
public abstract class Funcionario {
    protected String nome;
    protected double salario;

    public abstract double calcularRemuneracao();
}

public class FuncionarioCLT extends Funcionario {
    public double calcularRemuneracao() {
        return salario + calcularHoraExtra();
    }

    private double calcularHoraExtra() {
        return salario * 0.5;
    }
}

public class Estagiario extends Funcionario {
    public double calcularRemuneracao() {
        return salario; // sem hora extra — mas sem quebrar contrato ✅
    }
}

// qualquer Funcionario pode ser usado aqui sem surpresas
public void processarFolha(List<Funcionario> funcionarios) {
    funcionarios.forEach(f -> System.out.println(f.calcularRemuneracao()));
}
```

---

## I — Interface Segregation Principle
> Nenhuma classe deve ser forçada a implementar métodos que não usa

```java
// ❌ Errado — interface gorda força implementação desnecessária
public interface Trabalhador {
    void trabalhar();
    void receberSalario();
    void tirarFerias();
    void receberBeneficios();
}

public class Estagiario implements Trabalhador {
    public void trabalhar() { /* ok */ }
    public void receberSalario() { /* ok */ }
    public void tirarFerias() { throw new UnsupportedOperationException(); }     // ❌
    public void receberBeneficios() { throw new UnsupportedOperationException(); } // ❌
}

// ✅ Correto — interfaces pequenas e específicas
public interface Trabalhavel {
    void trabalhar();
}

public interface Remuneravel {
    void receberSalario();
}

public interface Beneficiavel {
    void tirarFerias();
    void receberBeneficios();
}

// Estagiário só implementa o que faz sentido
public class Estagiario implements Trabalhavel, Remuneravel {
    public void trabalhar() { System.out.println("Trabalhando..."); }
    public void receberSalario() { System.out.println("Recebendo bolsa..."); }
}

// CLT implementa tudo
public class FuncionarioCLT implements Trabalhavel, Remuneravel, Beneficiavel {
    public void trabalhar() { System.out.println("Trabalhando..."); }
    public void receberSalario() { System.out.println("Recebendo salário..."); }
    public void tirarFerias() { System.out.println("De férias!"); }
    public void receberBeneficios() { System.out.println("VT + VR..."); }
}
```

---

## D — Dependency Inversion Principle
> Dependa de abstrações, não de implementações

```java
// ❌ Errado — service depende diretamente da implementação concreta
public class FuncionarioService {
    // acoplado ao PostgreSQL — trocar o banco exige mudar o service ❌
    private FuncionarioRepositoryPostgres repository = new FuncionarioRepositoryPostgres();

    public void salvar(Funcionario f) {
        repository.salvar(f);
    }
}

// ✅ Correto — depende da abstração
public interface FuncionarioRepository {
    void salvar(Funcionario f);
    Optional<Funcionario> buscarPorId(Long id);
}

public class FuncionarioRepositoryPostgres implements FuncionarioRepository {
    public void salvar(Funcionario f) { /* salva no Postgres */ }
    public Optional<Funcionario> buscarPorId(Long id) { /* busca no Postgres */ }
}

public class FuncionarioRepositoryMock implements FuncionarioRepository {
    public void salvar(Funcionario f) { /* salva em memória — para testes */ }
    public Optional<Funcionario> buscarPorId(Long id) { /* busca em memória */ }
}

// service recebe a abstração via construtor — não sabe qual implementação é usada ✅
public class FuncionarioService {
    private final FuncionarioRepository repository;

    public FuncionarioService(FuncionarioRepository repository) {
        this.repository = repository;
    }

    public void salvar(Funcionario f) {
        repository.salvar(f);
    }
}

// na aplicação — injeta o Postgres
FuncionarioService service = new FuncionarioService(new FuncionarioRepositoryPostgres());

// nos testes — injeta o mock sem mudar nada no service ✅
FuncionarioService serviceTest = new FuncionarioService(new FuncionarioRepositoryMock());
```

---

## 📌 Resumo

| Princípio | Definição |
|---|---|
| **S** — Single Responsibility | Uma classe = uma responsabilidade |
| **O** — Open/Closed | Extenda com novas classes, não modifique as existentes |
| **L** — Liskov Substitution | Subclasse sempre pode substituir a pai sem quebrar |
| **I** — Interface Segregation | Interfaces pequenas e específicas, não genéricas |
| **D** — Dependency Inversion | Dependa de interfaces, não de classes concretas |