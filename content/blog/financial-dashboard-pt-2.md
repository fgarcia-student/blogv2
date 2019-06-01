---
title: "Financial Dashboard Pt 2"
date: 2019-05-27T21:11:19-04:00
draft: true
---

![C Sharp (C#)](/img/04_hero.jpg)

Alright! We are starting! Let's get the excitment out now.

> ...

### Technologies

We will be using a SQLite database, with EntityFramework as our ORM. This lives in a .NETCore application.

### Roadmapping

Cool! So lets begin by redefining what our objective really is. We described in an overview what the need for the application was, but now we can translate that into more workable requirements.

* We want to be able to track income and expenses
* We want to be able to set savings goals that we can reach based on our spending habits.
* We want to be able to see general analysis of our spending to better track our progress.

Now, there are many ways we can go about meeting these requirements, so if you think of a better way, or just think my plan is garbage, let me know in the comments what I can do to improve!

Ok, so what are some takeaways:

* Each entry into the application will either be:
  * Income
  * Expense
  * Saving

We also want a way to setup frequent payments like rent, or electricity, or a paycheck amount. We can call these `templates`.

When we are actually marking the day that the expense occurs, we can instantiate the template and append date information and additional tracking features that can be used for our analysis later on.

So to recap:

> We have our types, which classify specific templates for expenses like rent and income like a paycheck. When the expense is incurred, we instantiate the proper template and mark as completed on a certain day.

We can visualize this in the following way:

We are out shopping and spend $65.99 on a hoodie and some jeans. We have a template titled "clothing" in our application, where we are logging all clothing purchases. We create a new clothing entry with a price of 65.99, which affects our overall available budget and saving goals that were previously planned out.

Pretty cool huh?

> ...

I've found it very good to plan out what you will be doing before writing any code, as this allows the application to "pre-build" itself in your mind. You can code much faster without thinking about much of the structure as it has been laid out already. 

> Dude, let's see some code

Well, hold on. We still don't know where we will be storing all of this data.

### Persistent Storage

We will be using SQLite for our example, with our ORM of choice being EntityFramework. The benefits of using an ORM outweigh the drawbacks in this specific scenario, if we prioritize portability. With EntityFramework, we can simply install the EntityFramework package that targets our underlying database, abstracting our code away from the SQL that is lives on. This is especially beneficial if your development environment database is different than the database you will be deploying your application against.

So you will need the following packages from NuGet:

* Microsoft.EntityFrameworkCore Version="2.2.4"
* Microsoft.EntityFrameworkCore.Sqlite Version="2.2.4"
* Autofac Version="4.9.2"
* Autofac.Extensions.DependencyInjection Version="4.4.0"
* AutoMapper Version="8.1.0"
* AutoMapper.Extensions.Microsoft.DependencyInjection Version="6.1.0"

The first two tell dotnet that we will be using EntityFramework and we are targeting a SQLite database. If you are targeting a different database, check [here](https://www.nuget.org/packages) for the appropriate package to install.

The Autofac packages are for implementing an Inversion of Control design pattern. For more information on IoC and design patterns, buy [this book](https://books.google.com/books?id=6oHuKQe3TjQC&printsec=frontcover&dq=gang+of+four&hl=en&sa=X&ved=0ahUKEwiOy_O6j73iAhVGI6wKHbnwBN4Q6AEIKDAA#v=onepage&q&f=false) or just [google it dude.](http://lmgtfy.com/?q=Inversion+of+Control+Design+Pattern)

AutoMapper allows us to save time on changing between different models in our code. This is especially helpful when changing from a database model to a client facing model.

Ok, so with that defined, we can finally begin.

> WOW, took long enough.

Alright, alright. So let's start by creating our dotnet application!

( Make sure to have the dotnet cli installed!! )

Run the following command from your desktop:

`dotnet new webapi -n FinanceApp`

Feel free to swap out FinanceApp with whatever name you choose, just know I will be using FinanceApp.

now change directories into FinanceApp and open the directory with your text editor of choice. I choose you, VSCode!

`cd FinanceApp && code .` (or via the GUI, your call)

Once the directory is open we are greeted with the following folder structure:

![Folder Structure of our current application](/img/04_folder_structure.png)

## Opinion Alert

I won't lie to you guys, folder structure won't improve code performance. It won't make or break an application. But, it will help you structure your application in a systematic way that allows you to perform much faster. If you are looking for a specific file, you already know where it is generally. Or if you need to add a feature, you can first think of where in your application it makes sense to add that feature first so you can build in a single direction. This took me a long time to understand and I hope whoever is reading this also learns to understand the importance of proper file management.

So I am going to structure my files in the following way:

![Desired Folder Structure of our application](/img/04_folder_structure_2.png)

*ignore the Migrations folder, we will discuss that below*

You can do whatever you choose, this just helps me sleep better at night.

## End Opinion

So we want to make the above folder strucure. We are moving the Controllers folder inside of the Api folder.

So this breakdown keeps our controller code, our business logic or application specific code, and our database accessing code in all separate areas. And using IoC we can import from the bottom-up, from our database module into our business logic module, and into our api.

So in our Models folder we will create another folder called Entities. This will hold the database model that will represent our table.

> Wait wait, hold on, we didn't even write any SQL!

Exactly! We are leveraging the power of Code-First development, using EntityFramework as our support, to create models that represent tables in our database. And after registering them with EntityFramework we are able to create sql based on our code that targets the specified platform chosen above. This also means that if we want to change a column name or add a new column, we simply add the field to our model, and generate the sql to make the necessary changes. To learn more about Code-First development check out [this](https://docs.microsoft.com/en-us/ef/ef6/modeling/code-first/migrations/) link.

