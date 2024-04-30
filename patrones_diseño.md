# ¿Qué es un patrón de diseño?

Un patrón de diseño acaba siendo como un framework, ambos son "planos prefabricados" por los cuales podemos empezar a trabajar a partir de ellos y usar como una base.

Pero ambos tienen una diferencia, el framework se puede usar tal cual como la base y empezar ya a trabajar directamente.

El patrón de diseño, en cambio, es un concepto general de lo que es algo, no es código tal cual.

# ¿En qué consisten?

Explican el problema y la solución de forma breve, muestran cada una de las partes del patrón y cómo se relacionan, facilita la asimilación de la idea que se esconde tras este.

# ¿Por qué aprender sobre patrones?

Porque son una gran ayuda para cualquier programador, sobre todo para los principiantes, ya que nos dan una explicación bastante elaborada sobre un problema concreto, o varíos, ayudándonos a comprenderlos y a solucionarlos.

# Builder

En este caso, cogeré el patrón de diseño concocido como "builder", el cual tiene como objetivo solucionar el problema de que un objeto, que sea complejo ya que requiere de muchas funciones, campos y objetos anidados, esté dentro de un constructor el cual tenga una gran cantidad de parámtetros, o que esté disperso por todo el código, totalmente desordenado e inentendible.

La solución que propone "Builder", es que se saque el código del constructor de su propia clase, y se coloque dentro de objetos independientes. He aquí un ejemplo en PHP de un "Builder":

```php
<?php

namespace RefactoringGuru\Builder\RealWorld;

/**
 * The Builder interface declares a set of methods to assemble an SQL query.
 *
 * All of the construction steps are returning the current builder object to
 * allow chaining: $builder->select(...)->where(...)
 */
interface SQLQueryBuilder
{
    public function select(string $table, array $fields): SQLQueryBuilder;

    public function where(string $field, string $value, string $operator = '='): SQLQueryBuilder;

    public function limit(int $start, int $offset): SQLQueryBuilder;

    // +100 other SQL syntax methods...

    public function getSQL(): string;
}

/**
 * Each Concrete Builder corresponds to a specific SQL dialect and may implement
 * the builder steps a little bit differently from the others.
 *
 * This Concrete Builder can build SQL queries compatible with MySQL.
 */
class MysqlQueryBuilder implements SQLQueryBuilder
{
    protected $query;

    protected function reset(): void
    {
        $this->query = new \stdClass();
    }

    /**
     * Build a base SELECT query.
     */
    public function select(string $table, array $fields): SQLQueryBuilder
    {
        $this->reset();
        $this->query->base = "SELECT " . implode(", ", $fields) . " FROM " . $table;
        $this->query->type = 'select';

        return $this;
    }

    /**
     * Add a WHERE condition.
     */
    public function where(string $field, string $value, string $operator = '='): SQLQueryBuilder
    {
        if (!in_array($this->query->type, ['select', 'update', 'delete'])) {
            throw new \Exception("WHERE can only be added to SELECT, UPDATE OR DELETE");
        }
        $this->query->where[] = "$field $operator '$value'";

        return $this;
    }

    /**
     * Add a LIMIT constraint.
     */
    public function limit(int $start, int $offset): SQLQueryBuilder
    {
        if (!in_array($this->query->type, ['select'])) {
            throw new \Exception("LIMIT can only be added to SELECT");
        }
        $this->query->limit = " LIMIT " . $start . ", " . $offset;

        return $this;
    }

    /**
     * Get the final query string.
     */
    public function getSQL(): string
    {
        $query = $this->query;
        $sql = $query->base;
        if (!empty($query->where)) {
            $sql .= " WHERE " . implode(' AND ', $query->where);
        }
        if (isset($query->limit)) {
            $sql .= $query->limit;
        }
        $sql .= ";";
        return $sql;
    }
}

/**
 * This Concrete Builder is compatible with PostgreSQL. While Postgres is very
 * similar to Mysql, it still has several differences. To reuse the common code,
 * we extend it from the MySQL builder, while overriding some of the building
 * steps.
 */
class PostgresQueryBuilder extends MysqlQueryBuilder
{
    /**
     * Among other things, PostgreSQL has slightly different LIMIT syntax.
     */
    public function limit(int $start, int $offset): SQLQueryBuilder
    {
        parent::limit($start, $offset);

        $this->query->limit = " LIMIT " . $start . " OFFSET " . $offset;

        return $this;
    }

    // + tons of other overrides...
}


/**
 * Note that the client code uses the builder object directly. A designated
 * Director class is not necessary in this case, because the client code needs
 * different queries almost every time, so the sequence of the construction
 * steps cannot be easily reused.
 *
 * Since all our query builders create products of the same type (which is a
 * string), we can interact with all builders using their common interface.
 * Later, if we implement a new Builder class, we will be able to pass its
 * instance to the existing client code without breaking it thanks to the
 * SQLQueryBuilder interface.
 */
function clientCode(SQLQueryBuilder $queryBuilder)
{
    // ...

    $query = $queryBuilder
        ->select("users", ["name", "email", "password"])
        ->where("age", 18, ">")
        ->where("age", 30, "<")
        ->limit(10, 20)
        ->getSQL();

    echo $query;

    // ...
}


/**
 * The application selects the proper query builder type depending on a current
 * configuration or the environment settings.
 */
// if ($_ENV['database_type'] == 'postgres') {
//     $builder = new PostgresQueryBuilder(); } else {
//     $builder = new MysqlQueryBuilder(); }
//
// clientCode($builder);


echo "Testing MySQL query builder:\n";
clientCode(new MysqlQueryBuilder());

echo "\n\n";

echo "Testing PostgresSQL query builder:\n";
clientCode(new PostgresQueryBuilder());
```

Para poder implementar este código, debemos seguir 6 pasos:

    1. Asegurarnos de poder definir de forma clara los pasos comunes del constructor, de lo cotnrario, no se podrá proceder a implementar este patrón.
    2. Declarar los pasos en la interfaz del constructor base.
    3. Crear un constructor por representación del producto e implementar en cada uno sus pasos de construcción.
    4. Crear una clase directora puede encapsular varias formas de construir un producto utilizando el mismo objeto.
    5. Se debe pasar un objeto por lo menos a la clase directora mediante los parámetros del constructor del director. Entonces este usa el objeto constructor para el resto de la construcción.
    6. El resultado tan solo se puede obtener directamente del director si todos los productos siguen la misma interfaz.