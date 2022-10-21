---
title: Extracting Values From Types
tags: [C#]
author: "Yunier"
date: "2022-04-10"
description: "Deconstructing values out of types."
---

Learned a cool [little trick](https://twitter.com/buhakmeh/status/1308089098306039814/photo/1) a while back from [Khalid](https://twitter.com/buhakmeh). As a developer, you will often run into scenarios that require you to get a subset of all fields from a model. There are many ways to achieve this task, returning the type and then grabbing each property, for example, take the following User type.

```C#
public class User
{
    public User(string name, DateTime dob)
    {
        var random = new Random();

        Id = random.Next();
        Name = name;
        DateOfBirth = dob;
    }

    public int Id { get; set; }
    public string Name {get; set; }
    public DateTime DateOfBirth { get; set; }
}
```

If you want to obtain the name and id property you can take the following approach.

```C#
var user = new User("James", DateTime.Now);
var userName = user.Name;
var userDateOfBirth = user.DateOfBirth;
```

Simple enough, I'm sure every developer at some point in their career has written code similar to the example above. The trick I learned makes this even simpler. It involves declaring deconstructing methods to extract information from types. The same example as above can now be written as.

```C#
public class User
{
    public User(string name, DateTime dob)
    {
        var random = new Random();

        Id = random.Next();
        Name = name;
        DateOfBirth = dob;
    }

    public int Id { get; set; }
    public string Name {get; set; }
    public DateTime DateOfBirth { get; set; }

    public void Deconstruct(out string name, out DateTime dob)
    {
        name = Name;
        dob = DateOfBirth;
    }
}

var user = new User("James", DateTime.Now);
var (userName, userDateOfBirth) = user;
```

You can declare multiple deconstructors per type, simply change the signature, so in the example above if I wanted to get the Id and date of birth instead of the name and the date of birth, I would need to add the following code to the User type.

```C#
public void Deconstruct(out int id, out DateTime dob)
{
    id = Id;
    dob = DateOfBirth;
}
```

If you prefer not to pollute your types with many deconstruct methods, you have the option to declare them in an extension class. For example, in the case of the User type, I can declare a new class UserExtensions.

```C#
public static class UserExtension
{
    public static void Deconstruct(this User user, out string name , out DateTime dob) => (name, dob) = (user.Name, user.DateOfBirth);
}
```

and you use the same syntax to invoke it.

```C#
var user = new User("James", DateTime.Now);
var (userName, userDateOfBirth) = user;
```

This is super useful when you need to extract various values and don't want to use a Tuple or declare another class.