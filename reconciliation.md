# Laravel Reconciliation

An overview of the architecture can be found [here](https://docs.google.com/a/flagshipcompany.com/drawings/d/1v5ejEbdLpRqEOIqFigHtpRH55Lko0r1H5e6LCQ-4NeI/edit?usp=sharing).

#### Tips for testing
The Laravel reconciliation, unlike the symfony one, doesn't use costs and billables; it uses "accountables".
There were some inconsistencies in the symfony system when it comes to these accountables because the implementation had not been fully completed; therefore, we will ignore data created before October 31, 2014 at 19:00.

1. Load a dabase backup; ideally, the one for the day after the reconciliation you want to work with was run.
2. Run the necessary migrations to avoid any errors.
3. Delete all shipments that were created on or after the date the reconciliation was supposed to be run (so that we always have the same quantity of 3rd-party shipments).
```sql
DELETE FROM entity_shipments
WHERE created_at > [SUPPOSED_RUN_DATE_HERE (Y-m-d H:i:s)]
;
```
4. Run the reconciliation or reconciliations in symfony.
5. After it's done, take a screenshot of the reconciliation numbers from the reconciliations page and **name it properly**.
6. Execute this query to create a table containing the costs and billables data from the reconciliation:
```sql
CREATE TABLE IF NOT EXISTS old_vals AS
SELECT bi.shipment_id, bi.invoice_id, 
bi.attribute_id, co.value as costValue, 
bi.value, CONCAT(a.module, a.sub_module, a.name) AS attName, 
s.tracking_number, s.is_adjusted, s.is_manual, s.courier_id
FROM entity_billables bi
LEFT JOIN entity_shipments sh ON sh.id=bi.shipment_id
LEFT JOIN attributes a ON  bi.attribute_id = a.id
LEFT JOIN entity_shipments s ON s.id = bi.shipment_id
LEFT JOIN entity_costs co ON co.shipment_id = bi.shipment_id AND co.attribute_id = bi.attribute_id
WHERE bi.invoice_id = [CYCLE_ID_HERE]
AND co.invoice_id = [CYCLE_ID_HERE]
AND bi.attribute_id = 56
#AND sh.courier_id = [COURIER_ID_HERE] # This line may be uncommented if checking for only one courier
ORDER BY bi.shipment_id
;
```
7. You may copy the reconlogs that belong to the reconciliation and paste them in a spreadsheet. They may be useful.

#### Running the test reconciliation
Since the interface to run a reconciliation didn't exist at the time this text was written, the reconciliator class test must be used to run a reconciliation.

1. Make sure your app/testing/database.php file is using your development database and not the test one. Comment out the migration, seeding, queue facade mocks and assertions. Make sure you specify the correct cycle_id and that you are using the corresponding file.
2. Test your reconciliator. It should load all the necessary data for a reconciliation and add the required jobs to the queue.
3. If your queue is not listening, execute ```php artisan queue:listen```
4. Wait for the reconciliation to finish.
5. Compare your numbers with the screenshots of the symfony reconciliation. Take a screenshot of the reconciliator numbers and name it properly.
6. Check the reconlogs for problems.
7. This query will show you the differences between the symfony reconciliation data and the one you just ran:
```sql
SELECT ov.*, if(abs(ov.costValue - nv.fcs_total) < 0.004, 0.0, (ov.costValue - nv.fcs_total)) as diffCost
, if(abs(ov.value - nv.total) < 0.004, 0.0, (ov.value - nv.total)) as diffBill
, nv.fcs_amount, nv.fcs_total, nv.total
FROM old_vals ov
RIGHT JOIN (
	SELECT ac.shipment_id, 
	sum(ac.fcs_amount) as fcs_amount, 
	sum(ac.fcs_total) as fcs_total, 
	sum(if(ac.amount > 0.0, ac.amount, 0.0)) as amount, #For UPS, remove the if
	sum(if(ac.total > 0.0, ac.total, 0.0)) as total #For UPS, remove the if
	FROM entity_accountables ac
	LEFT JOIN entity_shipments s ON s.id = ac.shipment_id
	WHERE ac.cycle_id = [CYCLE_ID_HERE]
	AND s.courier_id = [COURIER_ID_HERE]
	AND ac.type != 'debit'
	AND s.created_at > '2014-10-31 19:00:00' #We ignore shipments created before this time
	GROUP BY ac.shipment_id
	ORDER BY ac.shipment_id
) nv ON nv.shipment_id = ov.shipment_id
HAVING ABS(diffCost) > 0.001 OR ABS(diffBill) >= 0.01
ORDER BY ov.shipment_id
;
```

##### How to interpret this query's result
##### UPS
* Negative difference in costs with zero difference in what was billed. It means that the shipment was not adjusted because the cost was lower than what was originally quoted. These rows don't represent a problem.
* Positive, less than 1.0 difference in cost and a difference of one or two dollars in what was billed. It means a small charge was ignored. These rows don't represent a problem.
* Zero or small difference in cost and small difference in what was billed. It means that, most likely, the service was adjusted to match the costs and, this time, the markup was calculated properly. Check its accountables to see if that's the case. If so, these rows don't represent a problem.
* Negative, more than 5 dollars difference in costs and what was billed. Maybe an address correction was applied to the shipment automatically; check the recon logs. If no address correction was applied, then this row represents a problem.
* If in doubt, check that the accountables match the invoice charges, if they do, everything is fine. If the differences in costs and what was billed are negative and there are no **adjustment** accountables, everything is fine also.
* Fully-debited shipments *should* appear as zero difference in cost and a couple of dollars, positive difference in what was billed.
* Check the reconciliation's entity_reconlog rows. We need to target the items type **Mismatch**. To begin, discard "*Costs(0 $) do not match actual courier Costs([NEGATIVE_AMOUNT])*" because they are debits (if I'm not mistaken). Then analyze each remaining case individually, if any.
* It is very important not to lose your patience.

##### Purolator
* Negative difference in costs with zero difference in what was billed. It means that the shipment was not adjusted because the cost was lower than what was originally quoted. These rows don't represent a problem.
* Positive, less than 1.0 difference in cost and a difference of one or two dollars in what was billed. It means a small charge was ignored. These rows don't represent a problem.
* A marginal difference in costs (less than one cent) and a big negative difference (more than 5 dollars) could mean that we were charged for a third-party shipment because somebody used our account number to ship. This should be a temporary issue but, in the meantime, we have to inform about these cases so somebody charges it manually to the customer. See [Trello card #473](https://trello.com/c/E8zuvbmX/473-forbid-flagship-accounts-for-third-party-or-collect-shipment)


#### Other useful queries
* Total debits for a cycle and courier:
```sql
SELECT sum(ac.fcs_amount), sum(ac.fcs_total), sum(ac.amount), sum(ac.total) 
FROM entity_accountables ac
LEFT JOIN entity_shipments sh ON sh.id=ac.shipment_id
WHERE ac.cycle_id = [CYCLE_ID_HERE]
AND ac.type = 'debit'
AND sh.courier_id = [COURIER_ID_HERE]
GROUP BY ac.cycle_id
;
```
* Total insurance for a cycle and courier:
```sql
SELECT sum(ac.fcs_amount), sum(ac.fcs_total), sum(ac.amount), sum(ac.total) #ac.*
FROM entity_accountables ac
LEFT JOIN entity_shipments sh ON sh.id=ac.shipment_id
WHERE cycle_id = [CYCLE_ID_HERE]
AND attribute_id = 62
AND sh.courier_id = [COURIER_ID_HERE]
GROUP BY cycle_id
;
```

#### How to clear the queue
The queue can be cleared by usin the redis command line interface
```bash
$ redis-cli
```
Then execute the following command to delete everything from all databases:
```
> flushall
```
If you want to delete the contents of the **redis-queue** database only, then research how to do it and update this document.

Execute **exit** to exit the redis-cli
```
> exit
```
