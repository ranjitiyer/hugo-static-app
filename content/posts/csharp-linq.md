+++
title = 'Csharp Linq'
date = 2024-03-28T15:51:33-07:00
+++

Enumerables in C# support a functional style fluid API for manipulating collections and running aggregations over them. This is definitely one of the coolest features natively supported in C#. I've had opportunities to use Linq at work and here are some example drills. Someone day, I'd like to implement all the SQL challenges on DataLemur using C#.

> GroupBy and Max

Here we solve a very common SQL exercise to return the maximum salary earned in each department. In it's most basic form `GroupBy` lets us specify a grouping key and gives us a tuple of with department record as the key and an enumerable of employee objects that map to it. We can then return an anonymous object with just the relevant pieces of information required for reporting the result. 

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

> Left Join and Aggregation

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

> Inner joins, Grouping, Left join

We first set up the records / 'tables' relevant for this exercise. There are bills to be paid, communications that were sent and payments received and we'll do some slicing over data in this domain.

```csharp
 private record Bill(string patientId, int billAmount, long sentTimestamp);
 private record Communication(string patientId, string medium, long billAmount, long sentTimestamp,
     long score = default);
 private record Payment(string patientId, int paymentAmount, long paidTimestamp);

List<Bill> Bills =
[
 new("p1", 100, 10000),
 new("p2", 200, 10010),
 new("p2", 400, 10020),
 new("p3", 500, 10020),
];

List<Payment> Payments =
[
 new("p1", 100, 100010),
 new("p2", 400, 100030),
];

List<Communication> Communications =
[
 new("p1", "sms", 100, 10001),
 new("p1", "text", 100, 10002),
 new("p2", "text", 100, 10004),
 new("p2", "email", 400, 10003),
 new("p2", "email", 400, 10005),
 new("p3", "email", 500, 10020),
];
```

First we're asking for all the paid bills which requires joining bills and payments with a simple inner join but with a compound primary key using the patient id and bill amount.

```csharp
var paidBills = Bills
 .Join(Payments, bill => bill.patientId + "-" + bill.billAmount,
     payment => payment.patientId + "-" + payment.paymentAmount
     , (bill, payment) => new { bill.patientId, bill.billAmount }).ToList();
paidBills.ForEach(b => Console.WriteLine($"{b.patientId} : {b.billAmount}"));
```	

A typical use of `GroupBy` would ask an aggregate question about an entity. In this case we're counting the number of bills sent out per patient. Since the `GroupBy` gives us an Enumerable collection of items belonging to a group, we can simply `Count` over it and get the answer.

```csharp
Bills.GroupBy(
     b => b.patientId, 
     bill => bill, 
     (patientId, bills) => new { patientId, TotalBills = bills.Count() })
 .ToList()
 .ForEach(bill => Console.WriteLine($"Patient {bill.patientId} : {bill.TotalBills} bills"));
```
We might now want to know how many bills were paid without any communication sent to the patient, which is a little bit more involved. We setup a result set for bills paid and communication count

```csharp
var billsPaidAndCommuications = Bills
 .Join(Payments, bill => bill.patientId + "-" + bill.billAmount,
     payment => payment.patientId + "-" + payment.paymentAmount
     , (bill, payment) => new
     {
         bill.patientId,
         billTimestamp = bill.sentTimestamp,
         bill.billAmount,
         payment.paidTimestamp
     })
 .GroupJoin(Communications,
     arg => arg.patientId + "-" + arg.billAmount,
     communication => communication.patientId + "-" + communication.billAmount,
     (billAndPayment, communications) =>
         new { BillAndPayment = billAndPayment, Communications = communications }).ToList();

billsPaidAndCommuications.ForEach(obj =>
{
 var billAndPayments = obj.BillAndPayment;
 var comms = obj.Communications;

 Console.WriteLine($"Payment for {billAndPayments.patientId} for amount {billAndPayments.billAmount} " +
                   $" billed on {billAndPayments.billTimestamp} " +
                   $"was paid on {billAndPayments.paidTimestamp} after " +
                   $"{comms.Count()} messages were sent");
});
```

```csharp
public Dictionary<string, double> CalculateEffectiveness2()
{
 // all the Paid bills
 var paidBills = Bills
     .Join(Payments, bill => bill.patientId + "-" + bill.billAmount,
         payment => payment.patientId + "-" + payment.paymentAmount
         , (bill, payment) => new { bill.patientId, bill.billAmount }).ToList();
 Console.WriteLine("All the paid bills");
 paidBills.ForEach(b => Console.WriteLine($"{b.patientId} : {b.billAmount}"));
 

 
 // Bills payed without communication
 var billsPaidAndCommuications = Bills
     .Join(Payments, bill => bill.patientId + "-" + bill.billAmount,
         payment => payment.patientId + "-" + payment.paymentAmount
         , (bill, payment) => new
         {
             bill.patientId,
             billTimestamp = bill.sentTimestamp,
             bill.billAmount,
             payment.paidTimestamp
         })
     .GroupJoin(Communications,
         arg => arg.patientId + "-" + arg.billAmount,
         communication => communication.patientId + "-" + communication.billAmount,
         (billAndPayment, communications) =>
             new { BillAndPayment = billAndPayment, Communications = communications }).ToList();

 billsPaidAndCommuications.ForEach(obj =>
 {
     var billAndPayments = obj.BillAndPayment;
     var comms = obj.Communications;

     Console.WriteLine($"Payment for {billAndPayments.patientId} for amount {billAndPayments.billAmount} " +
                       $" billed on {billAndPayments.billTimestamp} " +
                       $"was paid on {billAndPayments.paidTimestamp} after " +
                       $"{comms.Count()} messages were sent");
 });
 
 return null;
}
```

How effective were these communications in getting a patient to pay their outstanding bill? The's different ways of score an outreach method (text/email/sms), but we can start with a simple scheme below where each outreach for a paid bill receives an equally weighted fraction of the total comms sent resulting in a payment. We then penalize communications by a fixed (and totally random amount) that didn't result in any payment and then report `commScore` as the result (a dictionary to communication means and it's score). THis example also demonstrates the use of `Distinct` and `ToDictionary` to convert a list to a dictionary.


```csharp
// Intialize a score for all distinct communication mediums
var commScore = Communications.Select(c => c.medium).Distinct().ToDictionary(c => c, c=> 0.0);

// Find communications that resulted in a payment
var paymentToComms = Payments.Select(payment => 
    (payment, Communications.Where(c =>
        c.patientId == payment.patientId
        && c.billAmount == payment.paymentAmount
        && c.sentTimestamp < payment.paidTimestamp)))
    .ToDictionary(tuple => tuple.payment, tuple=> tuple.Item2);

// score based on number of attempts it took to realize a payment
foreach (var key in paymentToComms.Keys)
{
    var allComms = paymentToComms[key].ToList();
    allComms.ForEach(comm =>
    {
        var score = (double) 1 / allComms.Count();
        commScore[comm.medium] += score;
    });
}

// Penalize communications that didn't result in a payment
var communicationsNoPayment = Communications.Where(c => !Payments.Any(payment => payment.patientId == c.patientId &&
            payment.paymentAmount == c.billAmount));  // exists
communicationsNoPayment.Select(c => commScore[c.medium] -= 0.2);
```

Linq also support a query syntax, a custom SQL-like DSL in addition to the method syntax explored in this post. To learn more about Linq please the official [MSDN](https://learn.microsoft.com/en-us/dotnet/standard/linq/#language-level-query-syntax) website.