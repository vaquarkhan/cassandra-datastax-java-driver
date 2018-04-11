I'm back with more Cassandra and Java integration today this time focusing on using the Datastax Java driver rather than Spring Data Cassandra which I have already written about quite a lot. The Datastax driver is actually used by Spring Data to interact with Cassandra but comes with some extra goodies built on top of it. But we don't want any of these today! We are going to use the Datastax driver directly and compare it to Spring Data when appropriate.

First things first, dependencies.
```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>

  <dependency>
    <groupId>com.datastax.cassandra</groupId>
    <artifactId>cassandra-driver-core</artifactId>
    <version>3.4.0</version>
  </dependency>

  <dependency>
    <groupId>com.datastax.cassandra</groupId>
    <artifactId>cassandra-driver-mapping</artifactId>
    <version>3.4.0</version>
  </dependency>

  <!-- Not required, just made my life easier -->
  <dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.4</version>
  </dependency>

  <dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.7</version>
  </dependency>

</dependencies>
```
As always I am using Spring Boot, just because we are depriving ourselves of Spring Data doesn't mean we need to go completely cold turkey from all Spring libraries. The Datastax related dependencies here are `cassandra-driver-core` and `cassandra-driver-mapping`. `cassandra-driver-core`, as the name suggests provides the core functionality to interact with Cassandra such as setting up a session and writing queries. `cassandra-driver-mapping` is not required to query Cassandra but does provide some object mapping, inconjunction with the core driver it will now serve as an ORM rather than a way to execute CQL statements.

We now have our dependencies sorted, the next step is to get connected to Cassandra so that we can actually start quering it.
```java
@Configuration
public class CassandraConfig {

  @Bean
  public Cluster cluster(
    @Value("${cassandra.host:127.0.0.1}") String host, @Value("${cassandra.cluster.name:cluster}") String clusterName,
    @Value("${cassandra.port:9042}") int port) {
    return Cluster.builder()
             .addContactPoint(host)
             .withPort(port)
             .withClusterName(clusterName)
             .build();
  }

  // @Bean(destroyMethod = "close")
  @Bean
  public Session session(Cluster cluster, @Value("${cassandra.keyspace}") String keyspace) throws IOException {
    final Session session = cluster.connect(keyspace);
    setupKeyspace(session);
    return session;
  }

  private void setupKeyspace(Session session) throws IOException {
    String[] statements = split(IOUtils.toString(getClass().getResourceAsStream("/cql/setup.cql")), ";");
    Arrays.stream(statements).map(statement -> normalizeSpace(statement) + ";").forEach(session::execute);
  }

  @Bean
  public MappingManager mappingManager(Session session) {
    final PropertyMapper propertyMapper = new DefaultPropertyMapper().setNamingStrategy(new DefaultNamingStrategy(LOWER_CAMEL_CASE, LOWER_SNAKE_CASE));
    final MappingConfiguration configuration = MappingConfiguration.builder().withPropertyMapper(propertyMapper).build();
    return new MappingManager(session, configuration);
  }
}
```
There is a bit more core here when compared to a similar setup using Spring Data (this class isn't even needed when combined with Spring Boot's auto-configuration) but class itself is pretty simple. The basic setup of the `Cluster` and `Session` beans shown here is the bare minimum required for the driver to work and will likely remain the same for any application you write, with more methods provided to add any additional configuration to make it suitable for your usecase.

Using values from `application.properties` we set the host address, cluster name and port of the `Cluster`. The `Cluster` is then used to create a `Session` for a particular keyspace on the Cassandra instance, again taken from the properties file.

The `MappingManager` bean comes from the `cassandra-driver-mapping` dependency and will mapping from a `ResultSet` to an entity (which we will look at later). For now we just need to create the bean and if we aren't happy with the default naming strategy of converting Java camel case to all lowercase with no separators in Cassandra we will need to set our own. To do this we can pass in a `DefaultNamingStrategy` to define the case that we are using within our Java classes and what we are using in Cassandra. Since in Java it is typical to use camel case we pass in `LOWER_CAMEL_CASE` and since I like to use snake case in Cassandra we can use `LOWER_SNAKE_CASE` (these are found in the `NamingConventions` class). The reference to lower specifies the case of the first character in a string, so `LOWER_CAMEL_CASE` represents `firstName` and `UPPER_CAMEL_CASE` represents `FirstName`. `DefaultPropertyMapper` comes with extra methods for more specific configuration but `MappingConfiguration` only has one job of taking in a `PropertyMapper` to be passed to a `MappingManager`.

---------------- SORT OUT THE CREATION OF THE KEYSPACE AND TABLES ---------------------------------

The next thing we should look at is the entity that will be persisted to and retrieved from Cassandra saving us the effort of manually setting values for inserts and converting results from reads. The Datastax driver provides us with a relatively simple way to do just that by using annotations to mark properties like the name of the table it is mapping to, which field matches to what Cassandra columns and which fields does the primary key comprise of.
```java
@Table(name = "people_by_country")
public class Person {

  @PartitionKey
  private String country;

  @ClusteringColumn
  private String firstName;

  @ClusteringColumn(1)
  private String lastName;

  @ClusteringColumn(2)
  private UUID id;

  private int age;
  private String profession;
  private int salary;

  private Person() {

  }

  public Person(String country, String firstName, String lastName, UUID id, int age, String profession, int salary) {
    this.country = country;
    this.firstName = firstName;
    this.lastName = lastName;
    this.id = id;
    this.age = age;
    this.profession = profession;
    this.salary = salary;
  }

  // getters and setters for each property

  // equals, hashCode, toString
}
```
This entity represents the `people_by_country` table as denoted by the `@Table`. Below is the CQL that you would execute to create the matching Cassandra table.
```sql
CREATE TABLE IF NOT EXISTS people_by_country(
  country TEXT,
  first_name TEXT,
  last_name TEXT,
  id UUID,
  age INT,
  profession TEXT,
  salary INT,
  PRIMARY KEY((country), first_name, last_name, id)
);
```
The primary key consists of the `country`, `first_name`, `last_name` and `id` field. The partition key consists of just the `country` field and the clustering columns are the remaining keys in the key, `id` is only included for uniqueness as you can obviously have people with the same names. I go into the topic of primary keys in much more depth in my earlier post, [Getting started with Spring Data Cassandra](URL).

Back to the Java code. I already mentioned the `@Table` annotation where we must specify the name of the table the entity represents, which also comes with various other options depending on your requirements, such as `keyspace` if you don't want to use the default keyspace the `Session` bean is configured to use and `caseSensitiveTable` which is self explanatory.

What about he primary key? As touched on above, a primary key consists of a partition key that itself contains one or more columns and/or clustering columns(s). To match up to the Cassandra table defined above we added the `@PartitionKey` and `@ClusteringColumn` annotations to the required fields. Both of the annotations have one property, `value` which specifies the order which the column appears in the primary key. The default value is `0` which is why some fields do not include a value.

The last requirements to get this entity to work are getters and setters as well as a default constructor so that the mapper can do it's thing. The default constructor can be private if you don't want anyone accessing it as the mapper uses reflection to retrieve it. You might not want to have setters on your entity since you would like the object to be immutable, unfortunately, there isn't anything you can really do about htis and will just have to concede the fight. Although I personally think this is fine as you could (and maybe should) convert the entity into another object that can be passed around your application without any of the entity annotations and thus no knowledge of the database itself. The entity can then be left as mutable and the other object that you are passing around can work exactly as you wish.

One last thing I want to mention before we move on. Remember the `DefaultNamingConvention` we defined earlier? This means that our fields are being matched to the correct columns without any extra work in the entity. If you didn't do this or wanted to provide a different field name to your column name then you could use the `@Column` annotation and specify it there.

We nearly have all the components we need to build our example application. The penultamate component is creating a repository that will contain all the logic for persisting and reading data to and from Cassandra. We will make use of the `MappingManager` bean that we created earlier and the annotations that we put onto the entity to convert a `ResultSet` into an entity without needing to do anything else ourselves.
```java
@Repository
public class PersonRepository {

  private Mapper<Person> mapper;
  private Session session;

  private static final String TABLE = "people_by_country";

  public PersonRepository(MappingManager mappingManager) {
    this.mapper = mappingManager.mapper(Person.class);
    this.session = mappingManager.getSession();
  }

  public Person find(String country, String firstName, String secondName, UUID id) {
    return mapper.get(country, firstName, secondName, id);
  }

  public List<Person> findAll() {
    final ResultSet result = session.execute(select().all().from(TABLE));
    return mapper.map(result).all();
  }

  public List<Person> findAllByCountry(String country) {
    final ResultSet result = session.execute(select().all().from(TABLE).where(eq("country", country)));
    return mapper.map(result).all();
  }

  public void delete(String country, String firstName, String secondName, UUID id) {
    mapper.delete(country, firstName, secondName, id);
  }

  public Person save(Person person) {
    mapper.save(person);
    return person;
  }
}
```
By injecting the `MappingManager` in via the constructor and calling the `mapper` method for the `Person` class, we are returned with a `Mapper<Person>` that will personally handle all of our mapping needs. We also need to retrieve the `Session` to be able to execute queries which is nicely contained within the `MappingManager` we injected.

For three of the queries we are directly relying on the mapper to retrieve data from Cassandra, but this only works for a single record, we call `get`, `save` or `delete`. Each of these methods work by accepting in the values that make up the `Person` entity's primary key and they must be entered in the correct order or you will experience unexpected results or exceptions will be thrown.

The other situations require a query to be executed before the mapper can be called to convert the returned `ResultSet` into an entity or collection of entities. I have made use of `QueryBuilder` to write queries and I have also chosen for this post to not write prepared statements. Although in most cases you should be using prepared statements I thought I would cover these in a separate post in the future, although they are similar enough and `QueryBuilder` can still be used so I am confident you could figure it out on your own if needed.

`QueryBuilder` provides static methods to create `select`, `insert`, `update` and `delete` statements which can then be chained together to (I know this sounds obvious) build the query. The `QueryBuilder` used here is also the same one that you can use in Spring Data Cassandra, in my earlier post, [Getting started with Spring Data Cassandra](URL), I also went into more depth on ways to write and execute queries.

The final step to creating this little application, is actually running it. Since we are using Spring Boot we just add the standard `@SpringBootApplication` and `run` the class. I have done just that below, as well as using `CommandLineRunner` to execute the methods within the repository so we can check that they are doing what we expect.
```java
@SpringBootApplication
public class Application implements CommandLineRunner {

  @Autowired
  private PersonRepository personRepository;

  public static void main(String args[]) {
    SpringApplication.run(Application.class);
  }

  @Override
  public void run(String... args) {

    final Person bob = new Person("UK", "Bob", "Bobbington", UUID.randomUUID(), 50, "Software Developer", 50000);

    final Person john = new Person("UK", "John", "Doe", UUID.randomUUID(), 30, "Doctor", 100000);

    personRepository.save(bob);
    personRepository.save(john);

    System.out.println("Find all");
    personRepository.findAll().forEach(System.out::println);

    System.out.println("Find one record");
    System.out.println(personRepository.find(john.getCountry(), john.getFirstName(), john.getLastName(), john.getId()));

    System.out.println("Find all by country");
    personRepository.findAllByCountry("UK").forEach(System.out::println);

    john.setProfession("Unemployed");
    john.setSalary(0);
    personRepository.save(john);
    System.out.println("Demonstrating updating a record");
    System.out.println(personRepository.find(john.getCountry(), john.getFirstName(), john.getLastName(), john.getId()));

    personRepository.delete(john.getCountry(), john.getFirstName(), john.getLastName(), john.getId());
    System.out.println("Demonstrating deleting a record");
    System.out.println(personRepository.find(john.getCountry(), john.getFirstName(), john.getLastName(), john.getId()));
  }
}
```
The `run` method contains some print lines so we can see whats happening, below is what they output.
```
Find all
Person{country='US', firstName='Alice', lastName='Cooper', id=e113b6c2-5041-4575-9b0b-a0726710e82d, age=45, profession='Engineer', salary=1000000}
Person{country='UK', firstName='Bob', lastName='Bobbington', id=d6af6b9a-341c-4023-acb5-8c22e0174da7, age=50, profession='Software Developer', salary=50000}
Person{country='UK', firstName='John', lastName='Doe', id=f7015e45-34d7-4f25-ab25-ca3727df7759, age=30, profession='Doctor', salary=100000}

Find one record
Person{country='UK', firstName='John', lastName='Doe', id=f7015e45-34d7-4f25-ab25-ca3727df7759, age=30, profession='Doctor', salary=100000}

Find all by country
Person{country='UK', firstName='Bob', lastName='Bobbington', id=d6af6b9a-341c-4023-acb5-8c22e0174da7, age=50, profession='Software Developer', salary=50000}
Person{country='UK', firstName='John', lastName='Doe', id=f7015e45-34d7-4f25-ab25-ca3727df7759, age=30, profession='Doctor', salary=100000}

Demonstrating updating a record
Person{country='UK', firstName='John', lastName='Doe', id=f7015e45-34d7-4f25-ab25-ca3727df7759, age=30, profession='Unemployed', salary=0}

Demonstrating deleting a record
null
```
We can see that `findAll`has returned all records and `find` has only retrieved the record that matches the input primary key values. `findAllByCountry` has excluded the Alice and only found the records from the UK. Calling `save` again on an existing record will update the record rather than inserting. Finally `delete` will delete the person's data from the database (like deleting facebook?!?!).

And thats a wrap.

I will try to write some follow up posts to this in the future as there are a few more interesting things that we can do with the Datastax driver that we haven't gone through in this post. What we have covered here should be enough to make your first steps in using the driver and start querying Cassandra from your application.

Before we go I would like to make a few comparisons between the Datastax driver and Spring Data Cassandra.