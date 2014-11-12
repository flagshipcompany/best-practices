# Laravel Reconciliation

An overview of the architecture can be found [here](https://docs.google.com/a/flagshipcompany.com/drawings/d/1v5ejEbdLpRqEOIqFigHtpRH55Lko0r1H5e6LCQ-4NeI/edit?usp=sharing).

#### Tips for development
The Laravel reconciliation, unlike the symfony one, doesn't use costs and billables; it uses "accountables".
There are some inconsistencies in the symfony system when it comes to these accountables because the implementation has not been fully completed. So, when you load a database backup, there are some queries you need to execute to make sure the database is in the ideal condition to work with it.

1. Load a dabase backup; ideally, the one for the day after the reconciliation you want to work with was run.
2. Run the necessary migrations to avoid any errors.
3. Note the run date of the reconciliation you want to work with.
4. Crash and restart the reconciliation you're going to work with.
5. Delete all shipments that were created on or after the date the reconciliation was run (so that we always have the same quantity of 3rd-party shipments).
```sql
DELETE FROM entity_shipments
WHERE created_at > [RUN_DATE_HERE (Y-m-d H:i:s)]
;
```
6. We need to make sure that the accountables corresponding to non-reconciled shipments.
```sql
# Removes cycle_id from non-reconciled shipments' accountables
UPDATE entity_accountables ac
LEFT JOIN entity_shipments s ON s.id = ac.shipment_id
SET ac.cycle_id = NULL
WHERE s.is_reconciliated = 0
;
```
7. Run the reconciliation in symfony again
8. After it's done, take a screenshot of the reconciliation numbers from the reconciliations page and **name it properly**.
9. Execute this query to create a table containing the costs and billables data from the reconciliation:
```sql
CREATE TABLE IF NOT EXISTS old_vals AS
SELECT bi.shipment_id, bi.invoice_id, 
bi.attribute_id, co.value as costValue, 
bi.value, CONCAT(a.module, a.sub_module, a.name) AS attName, 
s.tracking_number, s.is_adjusted, s.is_manual
FROM entity_billables bi
LEFT JOIN entity_shipments sh ON sh.id=bi.shipment_id
LEFT JOIN attributes a ON  bi.attribute_id = a.id
LEFT JOIN entity_shipments s ON s.id = bi.shipment_id
LEFT JOIN entity_costs co ON co.shipment_id = bi.shipment_id AND co.attribute_id = bi.attribute_id
WHERE bi.invoice_id = [CYCLE_ID_HERE]
AND bi.attribute_id = 56
AND sh.courier_id = [COURIER_ID_HERE]
ORDER BY bi.shipment_id
;
```
10. You may copy the reconlogs that belong to the reconciliation and paste them in a spreadsheet. They may be useful.

#### Running the test reconciliation
Since the interface to run a reconciliation didn't exist at the time this text was written, the reconciliator class test must be used to run a reconciliation.

1. Make sure your app/testing/database.php file is using your development database and not the test one.
2. Test your reconciliator. It should load all the necessary data for a reconciliation and add the required jobs to the queue.
3. If your queue is not listening, execute ```php artisan queue:listen```
4. Wait for the reconciliation to finish.
5. Compare your numbers with the screenshots of the symfony reconciliation. Take a screenshot of the reconciliator numbers and name it properly.
6. Check the reconlogs for problems.
7. This query will show you the differences between the symfony reconciliation data and the one you just ran:
```sql
SELECT ov.*, if((ov.costValue - nv.fcs_total) < 0.004, 0.0, (ov.costValue - nv.fcs_total)) as diffCost
, if((ov.value - nv.total) < 0.004, 0.0, (ov.value - nv.total)) as diffBill
, nv.fcs_amount, nv.fcs_total, nv.total
FROM old_vals ov
RIGHT JOIN (
	SELECT ac.shipment_id, sum(ac.fcs_amount) as fcs_amount, sum(ac.fcs_total) as fcs_total, sum(ac.amount) as amount, sum(ac.total) as total
	FROM entity_accountables ac
	LEFT JOIN entity_shipments s ON s.id = ac.shipment_id
	WHERE ac.cycle_id = [CYCLE_ID_HERE]
	AND s.courier_id = [COURIER_ID_HERE]
	AND ac.type != 'debit'
	GROUP BY ac.shipment_id
	ORDER BY ac.shipment_id
) nv ON nv.shipment_id = ov.shipment_id
HAVING ABS(diffCost) > 0.005 OR ABS(diffBill) >= 0.07 
ORDER BY ov.shipment_id
;
```
Some differences in the costs are normal because we don't reconcile differences of less than one CAD.

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
