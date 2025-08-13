## GEN 
Def: gorm.io/gen is a code generation tool for GORM.
- it generates type-safe, boilerplate-free code for interacting with your database via GORM. That means you get a clean, compile-time-checked API for queries, but you're still responsible for defining your database schema and migrations yourself.
- Rather than writing SQL or generic GORM queries, you generate code that makes writing queries safe and developer-friendly.

### What Does gorm.io/gen Actually Do?
#### Automates CRUD & DAO Layer
It scans your GORM models or database schema and generates code that lets you interact with the DB using native Go types and methods
- Basic DAO: Automatically supports CRUD operations from models.
- Interface-Based: Define dynamic SQL logic via interfaces and get generated implementations.
- Database-to-Struct: Reverse-engineer your DB schema into Go models following GORM conventions. 
GORM


#### Dynamic SQL
Gen allows ==generate fully-type-safe idiomatic Go code from Raw SQL==, it uses annotations on interfaces, those interfaces could be applied to multiple models during code generation.

```go
type Querier interface {
  // SELECT * FROM @@table WHERE id=@id
  GetByID(id int) (gen.T, error) // GetByID query data by id and return it as *struct*

  // GetByRoles query data by roles and return it as *slice of pointer*
  //   (The below blank line is required to comment for the generated method)
  //
  // SELECT * FROM @@table WHERE role IN @rolesName
  GetByRoles(rolesName ...string) ([]*gen.T, error)

  // InsertValue insert value
  //
  // INSERT INTO @@table (name, age) VALUES (@name, @age)
  InsertValue(name string, age int) error
}

g := gen.NewGenerator(gen.Config{
  // ... some config
})

// Apply the interface to existing `User` and generated `Employee`
g.ApplyInterface(func(Querier) {}, model.User{}, g.GenerateModel("employee"))

g.Execute()
```

