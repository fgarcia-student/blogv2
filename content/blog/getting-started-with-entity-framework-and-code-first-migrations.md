---
title: "Getting Started with Entity Framework and Code-First Migrations"
date: 2019-05-31T20:14:32-04:00
draft: false
---

Hey all! I'm taking a break from writing my Finance App blog posts to write some fundamental blog posts. These will serve as references that I can point to from my Finance App posts, and any other future tutorials. Thanks to [my good friend, Devonta](http://www.codingscale.com/) for giving me a much better strategy when writing blog posts.

In trying to learn as much about the C# environment as I can, I decided to write about something almost everyone in the community is aware of. 

Entity Framework is an ORM or Object Relational Mapping tool that comes with many features to help developers write less domain specific data access code. This framework also has the ability to generate entire SQL schemas based on traditional Object Oriented classes. This helps developers in many ways, such as:

- Abstracting SQL code away from the developer
- Conforming the codebase to be entirely C# instead of C# + SQL / NoSQL
- Simplifying maintainability

Lets start with a simple example. We will be building a simple .NET Core application that communicates with a SQLite database using Entity Framework.

Start by opening up a terminal and changing directory to where you want to create your project.

Once you are in the desired location, lets create a new webapi using the .NET Core CLI:

`dotnet new webapi -n EntityFrameworkExample`

This will create a new project that has an api exposed for us. This will be helpful later on as we continue to develop our application and want to test it.

Now that we have our base application, we will want to install a few packages to help us out, most notably:

- Entity Framework 
- SQLite provider for Entity Framework

Run the following commands to install the above packages:

`dotnet add package Microsoft.EntityFrameworkCore --version 2.2.4`
`dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 2.2.4`

Let's start by defining the application. We are going to create a school database system where we can assign teachers and students to classrooms. We will focus on retrieving the information in this article, and [creating entries in the following article.](#)

Let's make 3 files:

- ClassRoom.cs
- Student.cs
- Teacher.cs

These three files will be our Entities. They will represent tables in our database. 

Our classes looks like the following:

```c#
public class Student
{
  public int Id { get; set; }

  public string Name { get; set; }

  public char Grade { get; set; }

  public int ClassRoomId { get; set; }

  [JsonIgnore]
  public ClassRoom ClassRoom { get; set; }
}
public class Teacher
{
  public int Id { get; set; }

  public string Name { get; set; }

  public int ClassRoomId { get; set; }

  [JsonIgnore]
  public ClassRoom ClassRoom { get; set; }
}
public class ClassRoom
{
  public int Id { get; set; }

  public string RoomName { get; set; }

  public ICollection<Teacher> Teachers { get; set; }

  public ICollection<Student> Students { get; set; }
}
```

There are a few things going on here. For one, we need the Id property for our class to tell Entity Framework that this is our Primary Key. This will automatically auto increment on new entity creation. Another thing you will notice is that we have references to a ClassRoom in each of our Teacher and Student classes. We also have a collection of Teachers and Students in our ClassRoom class. The relationship we want is a 1-N between a classroom and both a teacher and a student. So, a classroom can have many teachers and many students, but a teacher can only be in 1 classroom at a time, and same for a student. We represent this relationship to Entity Framework by saying the ClassRoom contains many Teachers and Students. And each Student and Teacher references the ClassRoom they belong to. This allows Entity Framework to map the relationship properly.

We follow certain naming conventions so that we do less work. This includes adding the entity name + id as a foreign key property, along with a reference to the Entity Type that it references as entity name. This looks like:

```c#
class SomeClass {
  public int Id { get; set; }

  public int SomeOtherClassId { get; set; }

  public SomeOtherClass SomeOtherClass { get; set; }
}
```

This naming convention allows EntityFramework to automatically detect these relationships without further annotations.

One last thing. We add the tag `[JsonIgnore]` on the ClassRoom references in both the Student and Teacher classes to prevent a circular reference when retrieving an entity.

Phew! Ok! Final thing we will do to help Entity Framework setup is define our Database Context, which we will call DbContext.

Let's make another file called `ApplicationContext.cs`

This class will extend the `DbContext` class from the EntityFramework package. Here we will register our classes with Entity Framework, and also define our Database type and connection details. 

Here is our class:

```c#
public class ApplicationContext : DbContext
{
  public DbSet<ClassRoom> ClassRoom { get; set; }

  public DbSet<Teacher> Teacher { get; set; }

  public DbSet<Student> Student { get; set; }

  protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
  {
    optionsBuilder.UseSqlite("Data Source=school.db");
  }
}
```

We Define our tables as DbSets of our Entity types so that Entity Framework knows how to create our tables properly. We also override a method on the DbContext class to define our database type. Here we register our SQLite database with the application as shown.

Now that we have finally finished setting up entity framework with our classes, and properly mapped the relationships, we can tell entity framework to make our tables and apply our models to the database. We do this in a 2 step process:

1) Generate our migrations

2) Apply our migrations to the database

To generate our first migration run the following command:

`dotnet ef migrations add InitialMigration`

This creates a Migrations folder in our project that holds code to create our tables in our database. These changes have not been applied to our database yet, however. This is done by the following command:

`dotnet ef database update`

This will apply our migrations, in order, onto the database. Once you run this command, you will see a `school.db` file appear in your project. This now contains our tables that we can perform queries against!

Let's try it out with some data. Here is some data I created for this example. 
Create a method in your Program.cs file called `SeedDatabase`. Call it before the `CreateWebHostBuilder` method. Write the following into the body of the SeedDatabase method:

```c#
using (var dbContext = new ApplicationContext())
{
  if (dbContext.ClassRoom.Count(x => x.RoomName == "Sunshine Room") == 1)
  {
    return;
  }
  dbContext.ClassRoom.Add(new ClassRoom()
  {
    RoomName = "Sunshine Room"
  });
  dbContext.ClassRoom.Add(new ClassRoom()
  {
    RoomName = "Moonlight Room"
  });
  dbContext.SaveChanges();
  var classRoom = dbContext.ClassRoom.Where(x => x.RoomName == "Sunshine Room").FirstOrDefault();
  dbContext.Teacher.Add(new Teacher()
  {
    Name = "Mrs. Tonkin",
    ClassRoomId = classRoom.Id,
    ClassRoom = classRoom
  });
  dbContext.Student.Add(new Student()
  {
    Name = "Tom",
    Grade = 'A',
    ClassRoomId = classRoom.Id,
    ClassRoom = classRoom
  });
  dbContext.Student.Add(new Student()
  {
    Name = "Mary",
    Grade = 'A',
    ClassRoomId = classRoom.Id,
    ClassRoom = classRoom
  });
  dbContext.Student.Add(new Student()
  {
    Name = "Richard",
    Grade = 'D',
    ClassRoomId = classRoom.Id,
    ClassRoom = classRoom
  });
  dbContext.SaveChanges();
}
```

This is creating 2 classrooms, a teacher and a few students, and assigning them to one of the classrooms.

Now let's go to our Controllers folder and rename the file and class from ValuesController to `SchoolsController`

Let's write the following code in the SchoolsController class:

```c#
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using EntityFrameworkExample.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace EntityFrameworkExample.Controllers
{
  [Route("api/[controller]")]
  [ApiController]
  public class SchoolsController : ControllerBase
  {

    private readonly ApplicationContext _dbContext;

    public SchoolsController()
    {
      _dbContext = new ApplicationContext();
    }
    // GET api/values
    [HttpGet("classrooms")]
    public ActionResult<IList<ClassRoom>> GetAllClassRooms()
    {
      return _dbContext.ClassRoom.ToList();
    }

    [HttpGet("teachers")]
    public ActionResult<IList<Teacher>> GetTeachersInSpecificClassRoom(
      [FromQuery] string classRoomName
    )
    {
      var classRoom = _dbContext.ClassRoom
        .Where(x => x.RoomName == classRoomName)
        .Include(x => x.Teachers)
        .FirstOrDefault();

      if (classRoom == null)
      {
        return new List<Teacher>();
      }

      return classRoom.Teachers.ToList();

    }

    [HttpGet("students")]
    public ActionResult<IList<Student>> GetStudentsInSpecificClassRoom(
      [FromQuery] string classRoomName
    )
    {
      var classRoom = _dbContext.ClassRoom
        .Where(x => x.RoomName == classRoomName)
        .Include(x => x.Students)
        .FirstOrDefault();

      if (classRoom == null)
      {
        return new List<Student>();
      }

      return classRoom.Students.ToList();

    }
  }
}

```

Here we are defining endpoints to retrieve our entries. We have an endpoint to retrieve all classrooms, and two endpoints to retrieve all teachers in a classroom as well as all students in a classroom.

With all of that setup, we can run the following command to start our application: `dotnet run`

If we navigate to <a href="https://localhost:5001/api/schools/classrooms">https://localhost:5001/api/schools/classrooms</a> we see a list of all the classrooms we have added to our database.

Awesome!

We can also <a href="https://localhost:5001/api/schools/students?classRoomName=Sunshine%20Room">retrieve a list of students in the Sunshine Room</a>. And also <a href="https://localhost:5001/api/schools/teachers?classRoomName=Sunshine%20Room">retrieve a list of teachers in the Sunshine Room</a>.

This is great!

We have learned what Entity Framework is and how to have Entity Framework manage our persistance layer of our application. We have alo finished setting up Entity Framework for our application, with some starter data.

<a href="#">In my follow up article</a>, we will be revising this application by making improvments to our setup, and adding the ability to create new entries via Postman. We will also be going more in depth into Entity Framework and the inner workings of an ORM Framework of this complexity.

I have posted the code to my [github repo](https://github.com/fgarcia-student/EntityFrameworkTutorialPt1/tree/topic/part-1). Feel free to check it out!

Thanks for reading, and stay tuned for future articles!
