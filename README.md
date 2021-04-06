# Gumroad Customer Payouts

I decided to write this part of the challenge up as a markdown styled blog post, since it would be quite hard to actually implement real code with a working database and everything. 😄

Solution assumptions: 

- All purchases and refunds are in the same table, where a column like something like 'amount' would just change it's arithmetic sign based on the type of transaction.

- The payout logic itself _must_ be run on a per-seller basis, and a complete timestamp of the calculation start time must be stored per-seller. Otherwise, purchases which are made or refunded while this potentially long running process. This is the most important part of the process and is discussed in more detail further down.

I first discuss what database level queries would be needed, then I show some pseudo JavaScript code as to what an actual code automation process could look like.

# Queries Needed

We will need three database queries to accomplish this task.

### Query 1: All Sellers Query

We will need a query where we can get all seller IDs and when there last payment calculation was made:

```sql
SELECT seller_id, last_payment_calculation_executed FROM sellers;
```

This is a simple SELECT query and no variables need to be passed into this query.

### Query 2: All Transactions for All Products Per Seller Query

We will also need a query where we can get all the purchases for all products for a given seller since the last payment calculation was run. Note that this can be based on a system wide date, but ultimately has to be per customer set and retrieved (more on that further down):

```sql
SELECT * FROM transactions t INNER JOIN products p ON p.product_id = t.product_id WHERE p.seller_id = seller_id AND t.transaction_date > last_payment_calculation_executed
```

Here, `seller_id` and `last_payment_calculation_executed` are variables that would be passed into this query.

### Query 3: Update Calculation Executed Query

We'll also need an UPDATE query to be able to update when the last payment calculation was executed.

```sql
UPDATE SET last_payment_calculation_executed=calculation_executed sellers WHERE seller_id = seller_id
```

This query would require a `calculation_executed` datetime, and the seller's ID `seller_id`.

# Main Program Logic (JavaScript Pseudo Code)

With these queries defined, we can run our payout logic for all sellers. In JavaScript pseudo-code, this would look something like:

```javascript
// get all sellers (Query 1 from above)
const sellers = runSellerQuery();

// for each seller, run the payout logic
sellers.forEach(seller => {
    // current time - critical to save this before querying a seller's transactions!
    const now = new Date();

    // get all transactions for this seller based on ID and last payment calculation executed (Query 2 from above)
    const transactions = runTransactionJoinQuery(seller.seller_id, seller.last_payment_calculation_executed);

    // calculate the sum - in javascript, a nice reduce function should do
    const sum = transactions.reduce((a, b) => a.transactionAmount + b.transactionAmount, 0);
    try {
        // pseudo code functions of all the stuff to do - actually sending payment, notification emails, etc.
        postPaymentToSeller(sum)
        emailSeller()
        moreCoolStuff()

        // nothing was thrown in our resource-intense functions, safe to update last calculation for this seller (Query 3 from above):
        runUpdateCalculationDateQuery(seller.seller_id, now);
    } catch {
        // logs emails to admins that there was an error with this seller
        logAndAlertGumroadStaff(seller)
        // alternatively, could fire off a subprocess to attempt to retry automatically for this customer at intervals
    }
})
```

_Most critically_, we update the seller table with the _exact_ datetime `now` _if and only if_ the entire payout logic for that seller was successful. 

This ensures that the seller is getting the payout in accordance to the state of the `transactions` table exactly at the moment we started the program logic. This is set to their respect row in the `sellers` table, to be accessed the next time the next payout should be calculated. You could add an additional column like `retries_needed` to try and identify later sellers with the most errors - and with some advanced SQL, the `last_payment_calculation_executed` column could become an array of datetimes as apposed to a single one, to view the history of past retries or fails of payout calculation processes.

# Comments and Notes

### Indexes

To make the `JOIN` query (Query 3) as fast as possible, the columns `product_id` on the `transactions` table and `seller_id` on the `products` table should be indexes - which I assume already are in the existing system. Then also the `transaction_date` column on the `transactions` table could be indexed.

Ideally, these tables would be siloed to no more than a single customer, or groups of customers as these intersect calculations can become slow for massive tables.

### System Timer

Ideally, I would set up this process in something like a UNIX system timer, running for example every friday at 5PM:

```
[Unit]
Description=Calculate payout for all Gumroad users
Requires=CalculatePayment.service

[Timer]
Unit=CalculatePayment.service
OnCalendar=Fri *-*-* 17:00:00
AccuracySec=1s

[Install]
WantedBy=timers.target
```

And the service would include the call to a script, which could look like the JavaScript pseudo code I offered above.

### Error Handling and Retries

Doing automatic retries as suggested in the `catch` in the pseudo code block above can get messy and lead to a bunch of garbage logging and alerts. Best would be to build out some sort of separate functionality that would be an attempt to parse exactly what went wrong in payout logic calculation (for example, what part when wrong - did the email not send, was there an NaN in the transaction calculation?)

## Cheers!

That's it for my application, I did the best I could!

Thank you for the opportunity,

-Chris
