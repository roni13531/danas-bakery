# Codex Prompt: Dana's Bakery Order Graph (LangGraph)

## Goal
Build a Google Colab notebook (`danas_bakery.ipynb`) that implements a bakery order-processing system as a **LangGraph `StateGraph`**.
All logic is **deterministic Python**. Do **not** call any LLM or external API, and do not require any API key.

## Environment
- First cell: `!pip install -q langgraph`
- The notebook must run top-to-bottom in a fresh Colab session.

## Business data (constants)
```python
PRICES = {"challah": 18, "babka": 45, "rugelach": 30, "sourdough": 25}   # ILS per unit
STOCK  = {"challah": 10, "babka": 0, "rugelach": 3, "sourdough": 4}
SUBSTITUTION_CHAIN = {"babka": "rugelach", "rugelach": "sourdough", "sourdough": "challah"}  # challah has no substitute
MAX_SUBSTITUTIONS = 2
DISCOUNT_RATE = 0.15
DISCOUNT_THRESHOLD = 150      # discount applies only if goods subtotal > 150 (strictly greater)
NO_DISCOUNT_DAY = "friday"
DELIVERY_CITY_NAMES = {"jerusalem", "ירושלים"}
DELIVERY_FEE = 20
VALID_DAYS = {"sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday"}
DISPLAY_NAMES = {  # (singular, plural) for customer messages
    "challah": ("challah", "challahs"),
    "babka": ("babka", "babkas"),
    "rugelach": ("box of rugelach", "boxes of rugelach"),
    "sourdough": ("sourdough loaf", "sourdough loaves"),
}
```
- "croissant" is NOT on the menu. Treat it like any unknown product.
- Stock is read-only. Each `graph.invoke` is independent and must not change `STOCK`.

## Input
```python
graph.invoke({"item": "babka", "quantity": 2, "day": "Thursday", "city": "Jerusalem"})
```
- `item`: product name (str). Compare after `.strip().lower()`.
- `quantity`: must be an `int` (not `bool`) and `> 0`.
- `day`: English weekday name. Compare after `.strip().lower()`.
- `city`: str, may be empty or `None`. Empty/None means customer picks up.

## State (`TypedDict`)
| key | type | meaning |
|---|---|---|
| item | str or None | current product (changes on substitution); `None` if rejected |
| quantity | int | requested quantity (never changes) |
| day | str | order day |
| city | str or None | customer city |
| requested_item | str | the product originally requested |
| attempts | int | number of substitutions tried (starts at 0) |
| reject_reason | str or None | `"invalid"` or `"out_of_stock"` |
| subtotal | float | price × quantity of the item actually sold |
| discount | float | discount amount on goods only |
| delivery_fee | float | 0 or 20 |
| delivery_note | str or None | set when a non-Jerusalem city is switched to pickup |
| status | str | `"accepted"` or `"rejected"` |
| delivery | str | `"pickup"` or `"delivery"` |
| total | float | final amount to pay, rounded to 2 decimals |
| message | str | reply to the customer |
| steps | `Annotated[list[str], operator.add]` | ordered log; every node appends exactly one entry |

## Nodes
1. **`validate`**: Valid if `item` is in `PRICES`, `quantity` is an int > 0, and `day` is in `VALID_DAYS`. Normalize `item` (lowercase) and set `requested_item`, `attempts = 0`. If invalid, set `reject_reason = "invalid"`.
2. **`check_stock`**: Checks whether `STOCK[item] >= quantity`.
3. **`substitute`**: Moves `item` to `SUBSTITUTION_CHAIN[item]` and increments `attempts`. Logs whether the new item has enough stock for the **full** quantity. If the current item has no next item in the chain, set `reject_reason = "out_of_stock"` and do not change `item`.
4. **`pricing`**: `subtotal = PRICES[item] * quantity`. If `subtotal > 150` and the day is not Friday, `discount = 15% of subtotal`; otherwise 0. The discount never applies to the delivery fee.
5. **`delivery`**:
   - city empty/None → `delivery = "pickup"`, fee 0.
   - city (normalized) in `DELIVERY_CITY_NAMES` → `delivery = "delivery"`, fee 20.
   - any other city → `delivery = "pickup"`, fee 0, and set `delivery_note` (we don't deliver outside Jerusalem; the order is kept as pickup).
   - Then `total = round(subtotal - discount + delivery_fee, 2)`.
6. **`compose_message`**: `status = "accepted"`. Build a warm, personal reply from Python templates (no LLM). Use `DISPLAY_NAMES` for product names in every customer message (never add "s" to a raw product key). Mention: the item and quantity, the substitution if `item != requested_item` (apologize that the original was out of stock and state the new unit price), the discount if one was given, the switch to pickup if `delivery_note` is set, and the total.
7. **`reject`**: `status = "rejected"`, `item = None`, `delivery = "pickup"`, `total = 0`.
   - `reject_reason == "invalid"` → polite message that we couldn't understand the order.
   - `reject_reason == "out_of_stock"` → apologize that the product the customer **originally requested** (`requested_item`, not the last substitute tried) and our suggested alternatives are out of stock, and that the order is cancelled. Use `DISPLAY_NAMES`.

## Edges
```
START → validate
validate → reject                      if invalid
validate → check_stock                 otherwise
check_stock → pricing                  if in stock
check_stock → substitute               otherwise
substitute → pricing                   if the new item has enough stock
substitute → substitute                if not in stock AND attempts < MAX_SUBSTITUTIONS AND item has a next item in the chain
substitute → reject                    otherwise (reject_reason = "out_of_stock")
pricing → delivery → compose_message → END
reject → END
```
Use `add_conditional_edges` with named router functions. Make sure `reject_reason = "out_of_stock"` is set in state before routing to `reject` (set it inside the `substitute` node, not in the router).

## Output
Every run must return all six keys: `status`, `item`, `delivery`, `total`, `message`, `steps`.

`steps` entries are short, readable strings, e.g.:
`"validate: ok"`, `"check_stock: babka x2 not in stock"`, `"substitute #1: babka -> rugelach (in stock)"`, `"pricing: subtotal 60, discount 0"`, `"delivery: delivery to Jerusalem, fee 20"`, `"compose_message: accepted"`, `"reject: invalid order"`.

## Visualization
Render the graph in the notebook:
```python
from IPython.display import Image, display
display(Image(graph.get_graph().draw_mermaid_png()))
```

## Tests
Add a helper `run(order)` that invokes the graph and prints the six keys (one `steps` entry per line).
Then run these orders, each in its own clearly titled cell, and `assert` the expected `status`, `item`, `delivery` and `total`:

| # | input (item, qty, day, city) | path | expected |
|---|---|---|---|
| 1 | challah, 3, Thursday, "Jerusalem" | in stock → delivery | accepted, challah, delivery, 74 |
| 2 | challah, 10, Thursday, "" | in stock → discount → pickup | accepted, challah, pickup, 153 |
| 3 | challah, 10, Friday, "Jerusalem" | Friday → no discount | accepted, challah, delivery, 200 |
| 4 | challah, 9, Thursday, "jerusalem " | discount on goods only + delivery | accepted, challah, delivery, 157.7 |
| 5 | babka, 2, Thursday, "Jerusalem" | 1 substitution | accepted, rugelach, delivery, 80 |
| 6 | babka, 4, Sunday, "Tel Aviv" | 2 substitutions → outside Jerusalem → pickup | accepted, sourdough, pickup, 100 |
| 7 | rugelach, 5, Monday, "ירושלים" | 2 substitutions ending at challah | accepted, challah, delivery, 110 |
| 8 | babka, 5, Tuesday, "Jerusalem" | 2 failed substitutions → cancelled | rejected, None, pickup, 0 |
| 9 | challah, 11, Wednesday, "" | out of stock, no substitute → cancelled | rejected, None, pickup, 0 |
| 10 | pizza, 1, Sunday, "" | invalid product | rejected, None, pickup, 0 |
| 11 | croissant, 2, Sunday, "" | discontinued product | rejected, None, pickup, 0 |
| 12 | challah, 0, Sunday, "" | invalid quantity | rejected, None, pickup, 0 |
| 13 | babka, -2, Sunday, "" | invalid quantity | rejected, None, pickup, 0 |
| 14 | challah, 2, Funday, "" | invalid day | rejected, None, pickup, 0 |

Finally, a cell that loops over all test orders and asserts all six keys are present in every result.

## Code style
- Clear markdown headers between sections (setup, data, state, nodes, routers, graph, drawing, tests).
- Short docstring on each node.
- No global mutable state.

---

## Revision log
- v1: initial spec. All 14 tests passed on the first run.
- v2: reading the outputs showed two message problems, fixed in the spec:
  - The cancellation message named the last substitute tried ("sourdough ... out of stock") instead of the product the customer asked for. Now it must use `requested_item`.
  - Product names were pluralized by adding "s" ("rugelachs", "sourdoughs"). Added `DISPLAY_NAMES` and required using it in all messages. The substitution message now also states the new unit price.
