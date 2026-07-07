# Referential Integrity & Fixtures

Most real test data isn't a single flat table — it's a graph of related entities (customers who have orders, which contain line items, which reference products, which belong to categories...). Generating each entity independently, without respecting these relationships, produces test data that's individually plausible but collectively broken.

---

## The Core Problem

```python
# WRONG: generating entities independently, with no relationship between them
customers = [fake_customer() for _ in range(10)]
orders = [fake_order() for _ in range(20)]  # orders reference customer_ids that
                                              # don't correspond to any generated customer!
```

Any test that joins across these tables, or that exercises cascade behavior (deleting a customer should cascade to their orders, or should be blocked if orders exist — which one is correct is a real business rule worth testing either way), will behave unpredictably against this kind of disconnected fixture set — not because the code under test is wrong, but because the test data itself doesn't represent a coherent state the system could actually be in.

---

## Factory Pattern with Relationship Awareness

```python
class CustomerFactory:
    def create(self, **overrides):
        defaults = {"id": fake.uuid4(), "name": fake.name(), "email": fake.email()}
        return {**defaults, **overrides}

class OrderFactory:
    def create(self, customer, **overrides):
        # Takes the related entity directly, not just an ID — makes the
        # dependency explicit and impossible to get wrong by construction
        defaults = {
            "id": fake.uuid4(),
            "customer_id": customer["id"],
            "total": fake.pydecimal(left_digits=3, right_digits=2, positive=True),
        }
        return {**defaults, **overrides}

# Usage — relationship is explicit and correct by construction
customer = CustomerFactory().create()
order = OrderFactory().create(customer=customer)
```

Passing the related *object* into the factory (not just its ID) is the key pattern: it makes it structurally impossible to create an order referencing a customer that doesn't exist in the same fixture set, because the customer object has to exist first to be passed in.

---

## Building a Coherent Dataset Graph

For larger fixture sets spanning many related entities, build the graph in dependency order and let factories consume upstream entities:

```python
def build_test_dataset(n_customers=5, orders_per_customer=3):
    customers = [CustomerFactory().create() for _ in range(n_customers)]
    orders = []
    for customer in customers:
        for _ in range(orders_per_customer):
            order = OrderFactory().create(customer=customer)
            orders.append(order)
            # Line items depend on both the order and a product —
            # continue the same pattern down the dependency graph
    return {"customers": customers, "orders": orders}
```

This produces a dataset where every foreign key genuinely resolves to a record that exists in the same set — critical for integration tests that exercise joins, cascades, and multi-table transactions, where a disconnected fixture set would produce failures that have nothing to do with the actual code being tested.

---

## Deliberately Testing Broken Referential States

Once coherent data generation is solid, the *inverse* is also a legitimate and important test case: deliberately construct a dataset with a broken relationship (an order referencing a nonexistent customer, a line item referencing a deleted product) specifically to test that the system detects and handles it correctly (rejects the write, cascades appropriately, surfaces a clear error) rather than silently accepting an inconsistent state.

```python
# Deliberately broken — this is the TEST, not a fixture-generation bug
orphaned_order = OrderFactory().create(customer={"id": "nonexistent-customer-id"})
# Assert the system rejects this, or handles it per the defined business rule
```

The distinction that matters: broken referential fixtures used *unintentionally* are a fixture-quality bug; the same broken state constructed *deliberately* to test error handling is a valid and valuable test case. Keep these clearly separated in the test suite (e.g. via naming or a dedicated `invalid_fixtures` module) so it's obvious which is which to anyone reading the test later.

---

## Seed Data for Integration Environments

Beyond per-test fixtures, integration/staging environments often need a standing set of "seed data" that represents a realistic baseline state before any test runs. The same relationship-aware factory approach applies, generated once and loaded via a seed script rather than constructed per-test:

```sql
-- Generated from the same factories, exported as SQL for environment seeding —
-- keeps the seed data and the unit-test fixtures built from the same
-- relationship-aware generation logic, rather than maintaining two
-- independent, potentially inconsistent representations of "valid test data"
INSERT INTO customers (id, name, email) VALUES (...);
INSERT INTO orders (id, customer_id, total) VALUES (...);
```

---

## Checklist

```
- [ ] Related entities are generated with explicit awareness of their relationships, not independently
- [ ] Factories take related objects (not bare IDs) as inputs where practical, making broken references structurally harder to create by accident
- [ ] Deliberately-broken referential states used for error-handling tests are clearly separated from real fixture-generation bugs
- [ ] Seed data for standing environments is generated from the same relationship-aware logic as per-test fixtures, not maintained as a separate, divergent artifact
```
