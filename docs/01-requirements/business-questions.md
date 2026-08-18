# Business Questions

This document defines the key analytical questions the Zavy platform must answer. Every question must be answerable from the data defined in `data-requirements.md` using the KPIs defined in `kpi-definitions.md`.

**Terminology convention:** sales value (**Gross Merchandise Value / GMV**, also **Seller Sales**) is recognized for orders in `CONFIRMED`, `PROCESSING`, `SHIPPED`, or `DELIVERED` status; cancelled and `PLACED`-only orders are excluded. GMV is the product selling value and belongs to the **sellers**; **Zavy's own revenue is the commission** earned on those sales. Returned value reduces both GMV and the commission owed.

## Sales

1. What is total Gross Merchandise Value (GMV), and how has it changed over time?
2. What is monthly and yearly GMV?
3. What is year-over-year GMV growth?
4. What is Zavy's commission revenue by month and by year?
5. What is the average order value (AOV), and how does it trend?
6. How many orders are placed per day, month, and year?
7. How many units are sold per day, month, and year?
8. Which payment methods account for the most GMV?
9. What is the order cancellation rate, and what does it cost in lost GMV?

## Customers

10. Which customers have the highest lifetime value (top 10 / top 100)?
11. What is the repeat-purchase rate?
12. What is customer retention over monthly cohorts?
13. How many new customers join each month?
14. What share of GMV comes from new vs returning customers?
15. What is the average number of orders per customer?
16. How are customers distributed by segment and location?

## Products

17. Which products and categories generate the most GMV?
18. Which categories are growing fastest?
19. Which products have the highest and lowest units sold?
20. Which products have the highest gross margin (seller view)?
21. How does GMV compare between brands and categories?
22. Which products have the highest return rates?

## Sellers

23. Which sellers perform best by GMV and order count?
24. Which sellers generate the most Zavy commission revenue?
25. What is each seller's average rating?
26. Which sellers have the highest return rates?
27. How concentrated is GMV among the top sellers?
28. How does seller GMV and Zavy commission trend over time?

## Inventory

29. Which products are low on stock or out of stock?
30. What is inventory turnover by product and warehouse?
31. What is the stock value held per warehouse?
32. Which products are slow-moving (low turnover)?

## Returns

33. Which products have the highest return rates?
34. What is the distribution of return reasons?
35. What is the return rate by category and by seller?
36. How do returns affect GMV and Zavy commission over time?

## Regional Analysis

37. Which cities/regions generate the most GMV?
38. How are customers and orders distributed across regions?
39. Which regions have the highest AOV?
40. How does seller performance vary by region?