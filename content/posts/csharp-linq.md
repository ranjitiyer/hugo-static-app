+++
title = 'Csharp Linq'
date = 2024-03-28T15:51:33-07:00
+++

Collections in C# support a functional style fluid API for manipulating collections and running aggregations over them. This is definitely one of the coolest features natively supported in C#. I've had opportunities to use Linq at work and here are some example drills. Someone day, I'd like to implement all the SQL challenges on DataLemur using C#.

> GroupBy and Max

Here we solve a very common SQL exercise to return the maximum salary earned in each department. In it's most basic for `GroupBy` lets us specify a grouping key and gives us a tuple of with department record as the key and an enumerable of employee objects that map to it. We can then return an anonymous object with just the relevant pieces of information required for reporting the result. 

```csharp
record Employee (int empId, string department, int salary);
public void MaxSalaryByDepartment()
{
    Employee[] employees =
    [
        new(1, "HR", 100),
        new(2, "HR", 200),
        new(3, "IT", 400),
        new(4, "IT", 9900),
    ];
    
    foreach (var keyValuePair in employees.GroupBy(e => e.department, (department, employees) =>
                 {
                     return new { Department = department, MaxSalary = employees.Max(e => e.salary) };
                 }).ToDictionary(o => o.Department, o => o.MaxSalary))
    {
        Console.WriteLine($"Max salary in department {keyValuePair.Key} " +
                          $"is {keyValuePair.Value}");
    }
}
```

> Left Join

This examples demonstrates the need for a left-join achieved using the `GroupJoin` method on a collection. We're asked to report the percentage of shipments sent to a user's home address as opposed to elsewhere. Recognizing that a user may not have a shipment, we place `Users` on the left side of the join. Then we specify the outer and inner key to match rows on either side and expect to get an `null` list of transactions for users that haven't bought anything. This we handle with the null check and return 0. The result is a just a count of shipments sent to a home address for which we use the `Aggregate` function that takes a `seed` to set the  accumulator value of the lambda which is rolled up to a single number. This we divide by the total shipments to get our answer 

```csharp
record Transaction(int txId, int userId, string shippedAddress);
record User(int userId, string address);

public double PercentShippedToHomeAddress()
{
    Transaction[] txs =
    [
        new Transaction(1, 1, "Laguna Niguel"),
        new Transaction(2, 1, "Irvine"), // home address
        new Transaction(3, 2, "Mumbai"), // home address
        new Transaction(4, 3, "LA"),
        new Transaction(5, 3, "MN"),
    ];

    User[] users =
    [
        new User(1, "Irvine"),
        new User(2, "Mumbai"),
        new User(3, "LA"),
        new User(4, "SFO"),  // user may not have any transactions
        new User(5, "MN")  // user may not have any transactions
    ];

    var ordersShippedToHome = users.GroupJoin(txs,
        user => user.userId, // outer key
        tx => tx.userId,
        (user, listOfTransactions) =>
        {
            if (listOfTransactions != null)
                return listOfTransactions.Count(tx => tx.shippedAddress.Equals(user.address));
            return 0;
        }).Aggregate(0, (acc, sum) => acc + sum);

    
    Console.WriteLine($"Shipped to home {ordersShippedToHome}");
    Console.WriteLine(ordersShippedToHome/(double) txs.Length);

    return ordersShippedToHome / txs.Length;
}
```